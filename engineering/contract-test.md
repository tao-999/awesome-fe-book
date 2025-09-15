# 11.3 契约测试 & Mock 服务 🤝🧪

本章目标：用**契约（Contract）**把「前端 ↔ BFF/后端」的依赖**写死在文档与测试里**，用**Mock 服务**把外部依赖“虚拟出来”。做到：**改后端不惊动前端**、**改前端不背刺后端**、**在本地与 CI 都能稳定跑**。

---

## 0) 心智图（一句话版）

- **契约 = 可机读的接口真相**（OpenAPI/JSON Schema/GraphQL SDL/Protobuf）。  
- **契约测试**验证“**实现**是否满足**约定**”（CDC/Pact、OpenAPI 校验、GraphQL 契约）。  
- **Mock 服务**在**不连真实后端**的情况下，提供**可信响应**（MSW/WireMock/Prism/Mockoon）。  
- **流水线**：生成/发布契约 → 消费方跑 CDC → 提交契约到 Broker → 提供方做 Provider Verification → 通过才允许部署。

---

## 1) 契约的四种形态 📜

1. **OpenAPI + JSON Schema（REST）**  
   - 单一真相（SSOT），机器可校验，配合 Prism/WireMock 生成 Mock。  
2. **GraphQL SDL/Operation 契约**  
   - Schema + 真实用到的 Operation（Document）。  
3. **Protobuf（gRPC）**  
   - `.proto` 就是契约，走 codegen/compat 检查。  
4. **消费者驱动契约（CDC, Pact）**  
   - 由**消费者**（前端/BFF）记录期望交互，生成 Pact 文件，由**提供者**验证。

> 心法：**先设计，再实现**。先落契约（Design-first），再开干；或至少做到 **实现后立契约不再走样**。

---

## 2) 消费者驱动契约（Pact）🧑‍🍳→🧑‍🔬

### 2.1 前端（消费者）生成契约
```ts
// tests/pact/consumer.auth.pact.test.ts
import { pactWith } from '@pact-foundation/pact';
import { Matchers } from '@pact-foundation/pacts'; // v11 也可直接用 @pact-foundation/pact
import fetch from 'node-fetch';

pactWith(
  { consumer: 'web-app', provider: 'auth-service' },
  (interaction) => {
    interaction('登录成功', ({ provider, addInteraction }) => {
      addInteraction({
        state: '用户存在且密码正确',        // provider state
        uponReceiving: 'POST /login with valid credentials',
        withRequest: {
          method: 'POST',
          path: '/login',
          headers: { 'content-type': 'application/json' },
          body: { email: 'user@acme.test', password: 'hunter2' }
        },
        willRespondWith: {
          status: 200,
          headers: { 'content-type': 'application/json; charset=utf-8' },
          body: {
            token: Matchers.regex(/^[A-Za-z0-9-_]+\.[A-Za-z0-9-_]+\.[A-Za-z0-9-_]+$/),
            user: { id: Matchers.uuid(), name: Matchers.like('Ada') }
          }
        }
      });

      // 实际调用 mock provider
      return (async () => {
        const res = await fetch(`${provider.mockService.baseUrl}/login`, {
          method: 'POST',
          headers: { 'content-type': 'application/json' },
          body: JSON.stringify({ email: 'user@acme.test', password: 'hunter2' })
        });
        const json = await res.json();
        // 正常写断言
        if (!('token' in json)) throw new Error('no token');
      })();
    });
  }
);
```

生成的 Pact 文件（`pacts/web-app-auth-service.json`）可**上传到 Broker**（自建/云托管），由**提供者流水线**拉取验证。

### 2.2 提供者（后端）验证契约
```bash
# 启动真实服务（指向测试数据库）
SERVICE_BASE_URL=http://localhost:3000

# 使用 verifier 拉取 Broker 最新契约并对服务逐条验证
pact-provider-verifier \
  --provider-base-url=$SERVICE_BASE_URL \
  --pact-broker-base-url=$PACT_BROKER_URL \
  --provider=auth-service \
  --publish-verification-results \
  --provider-app-version=$GIT_SHA
```

- **Provider States**：在验证器启动前注册“状态装配器”，如「用户存在」。  
- **验证通过**后，Broker 记录“**web-app@x.y.z ↔ auth-service@commit**”的兼容矩阵；部署策略可设置“**只有全部兼容才放行**”。

> 优点：**反向约束变更**，避免“后端偷偷改响应导致前端爆炸”。

---

## 3) OpenAPI 驱动的契约测试（REST）🌊

### 3.1 规范驱动（Design-first）
- 写 `openapi.yaml`，**定义路径/参数/请求体/响应/错误**，schema 下放 `components/schemas`。  
- 使用 **`examples`** 或 `oneOf/anyOf` 表达多形态；为分页/排序/过滤建统一约定。  
- 用 **Prism** 根据 `openapi.yaml` 起一个 Mock：
```bash
pnpm dlx @stoplight/prism-cli mock openapi.yaml --cors --port 4010
```

### 3.2 合规校验与回归
- **请求/响应校验**：在 BFF/后端中间件用 JSON Schema 校验（如 `ajv`），保证线上符合契约。  
- **合规测试**：用 Dredd / OpenAPI Test 工具对真实服务跑“**规范→实现**”的核对；或在集成测试里用 `openapi-response-validator` 校验响应。

> OpenAPI 适合“**API 面向外部/多语言**”场景；CDC 适合“**紧耦合消费方**”场景。两者可并用：**OpenAPI 做全局规范 + CDC 覆盖关键交互**。

---

## 4) GraphQL 契约思路 🧬

- **Schema** 稳定演进（新增字段/类型向后兼容；删除 → **major**）。  
- 用 **Operation Registry** 锁定已发布的 Query/Mutation 文档（前端构建时上传；服务端只放行已注册文档）。  
- **Contract Test**：对**已注册 operation** 做验证（字段存在、类型符合、指令可用）；对 Resolver 做**字段级**回归。  
- Mock：使用 `graphql-tools`/`msw-graphql` 为特定 Operation 提供可控响应。

---

## 5) Mock 服务的三层用法 🪄

| 层级 | 工具 | 适用场景 |
|---|---|---|
| **浏览器内** | **MSW（Mock Service Worker）** | 前端开发/单测/E2E；拦截 `fetch/XHR`，与用户体验接近 |
| **本地独立进程** | **Prism**（OpenAPI）、**WireMock**（HTTP Stub）、**Mockoon**（GUI） | 团队共享 API 模拟；无需跑后端 |
| **服务虚拟化** | **WireMock Standalone** / 自建 Node Mock | 后端/CI 的外部三方依赖（支付、短信、地图）的稳定替身 |

### 5.1 MSW（前端最佳拍档）
```ts
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw';
export const handlers = [
  http.get('/api/users/:id', ({ params }) =>
    HttpResponse.json({ id: params.id, name: 'Ada' })
  ),
  http.post('/login', async ({ request }) => {
    const { email } = await request.json();
    return HttpResponse.json({ token: 'jwt', user: { id: 'u1', name: email } });
  })
];
```
```ts
// src/mocks/browser.ts
import { setupWorker } from 'msw/browser';
import { handlers } from './handlers';
export const worker = setupWorker(...handlers);
```
```ts
// main.ts（开发模式启动）
if (import.meta.env.DEV) {
  const { worker } = await import('./mocks/browser');
  worker.start({ onUnhandledRequest: 'bypass' });
}
```

### 5.2 WireMock（后端/CI 稳定桩）
```yaml
# docker-compose.yml
services:
  wiremock:
    image: wiremock/wiremock:3
    ports: [ "8081:8080" ]
    volumes:
      - ./stubs:/home/wiremock
```
`stubs/mappings/get-user.json`
```json
{
  "request": { "method": "GET", "urlPattern": "/api/users/([A-Za-z0-9-]+)" },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json" },
    "jsonBody": { "id": "{{request.path.[2]}}", "name": "StubUser" },
    "transformers": ["response-template"]
  }
}
```

### 5.3 Prism（OpenAPI 一键起服）
```bash
pnpm dlx @stoplight/prism-cli mock openapi.yaml --errors --dynamic
```
- `--dynamic`：根据 schema 生成逼真随机值；  
- `--errors`：返回规范里定义的错误示例（覆盖异常路径）。

---

## 6) “状态”与“示例”的管理 🧱

- **Provider State**（Pact）：为每条交互命名「预置状态」，后端在验证前**布置数据**（工厂/seed/fixtures）。  
- **OpenAPI Examples**：为 `200/4xx/5xx` 各自给出 `examples`，并写明分页、多态（`oneOf`）、空态。  
- **数据去真实化**：Mock/Fixtures 不要包含**真实个人数据**；用生成器（如 `faker`）保持**稳定 + 随机种子**。

---

## 7) 版本与兼容策略 🔀

- **加字段**：向后兼容（MINOR）；**默认不变**。  
- **改语义/删字段**：破坏性（MAJOR）；在契约里标注 `deprecated: true` + **迁移窗口**。  
- **多版本共存**：通过 **URL 版本**（`/v1`）、**Header 版本**、或 **资源版本**（`application/vnd.acme+v2`）。  
- Broker/Registry 记录**兼容矩阵**，阻止“未兼容的组合”上线。

---

## 8) CI/CD：把契约放进闸门 🚦

**消费者仓库（前端/BFF）**
1. 跑单测/组件测/E2E（本地用 MSW）。  
2. 生成 Pact 并**发布到 Broker**（带上 commit/版本）。  
3. 可选：用 OpenAPI 校验当前调用是否都**存在于规范**。

**提供者仓库（后端）**
1. 启动服务（指向测试 DB）。  
2. 从 Broker 拉取**所有仍受支持消费者**的契约，逐条验证（含 Provider States）。  
3. 只有 **全部验证通过** 才允许构建镜像与部署。  
4. 部署后在 **合约环境** 再跑一遍合规校验（Smoke + Contract）。

---

## 9) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| “写完再补文档” | 规范总落后，前后端吵架 | **设计先行**，PR 必带契约变更 |
| 只写成功例子 | 空态/异常全挂 | 为每个端点提供 **200/4xx/5xx** 示例 |
| Mock 跑偏现实 | 一上真服就炸 | Mock 用**真实 schema/示例**生成；关键路径用 CDC/合规校验 |
| CDC 只在本地跑 | CI 绿但线上翻车 | 接入 Broker + Provider Verification 进 **必经闸门** |
| Pact 例子太“写死” | 易碎 | 用 `Matchers` 表达**结构与约束**，不要硬编码完整 body |
| Mock 永远开着 | 和真实环境脱节 | 本地开发/单测开 Mock；E2E/预览环境尽量真连 |

---

## 10) 提交前检查清单 ✅

- [ ] 契约（OpenAPI/SDL/Proto）已更新，并通过校验。  
- [ ] 为新增/修改接口补上 **成功/空/错误** 示例与说明。  
- [ ] 前端增加/修改的交互已生成 **Pact** 并上传。  
- [ ] 后端 Provider Verification 全绿；必要的 **Provider States** 可复用。  
- [ ] Mock 服务（MSW/Prism/WireMock）与契约**同源生成**，不手搓魔法数据。  
- [ ] 变更说明：是否 **向后兼容**？是否标注 **deprecated** 与迁移建议？

---

## 11) 练习 🏋️

1. 给“登录 + 获取当前用户”写一组 **Pact 用例**：成功、凭证错误、速率限制（429）。在后端接入 Provider States。  
2. 维护一个 `openapi.yaml`，用 **Prism** 启动 Mock，前端在无后端环境下完成“搜索页”的开发与 E2E。  
3. 在 CI 加入 **OpenAPI 合规校验**（请求/响应 JSON Schema 校验）与 **Pact Provider Verification**；未兼容的提交一律阻塞。

---

**小结**：契约测试负责“**变更可控**”，Mock 服务负责“**环境可控**”。把两者接上 CI/CD，你的前后端协作会从“喊话沟通”升级为“机器验证”，上线稳得像一台自动贩卖机。🧃🚀
