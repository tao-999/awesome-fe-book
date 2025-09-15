# 15.1 Cloudflare Workers / Vercel / Deno 🚀🌍

目标：把三大 **Edge/无服务器**运行时捏成一盘菜。你将拿到：**选型对照、最小样例、存储与一致性策略、调试与部署、常见坑**。口号：**离用户越近，越要轻、快、可观测。**

---

## 0) TL;DR 选型心法 🎯

| 维度 | Cloudflare Workers | Vercel（Edge & Serverless） | Deno Deploy |
|---|---|---|---|
| 运行模型 | **V8 Isolates**（Service Worker 风格，ESM 优先） | **Edge Functions**（Isolates，受限 Web API）与 **Serverless Functions**（Node.js） | **Deno Runtime**（TypeScript/ESM 原生，Web 标准 API） |
| 冷启动 | 极短（隔离级） | Edge 极短，Serverless 依平台与包体 | 极短 |
| 生态集成 | KV / D1(SQLite) / R2(S3) / Durable Objects / Queues / Cron / AI | Vercel KV/PG/Blob/Edge Config / Image / OG / 中间件 / Preview | Deno KV / Cron / KV 监听 / Web 标准 API |
| 何时优先 | **全球分发 + 状态协调（DO）+ 高并发** | Next.js/前端一体化、**预览环境**、边缘中间件 | TS 原生、**最贴近 Web 标准**、少依赖 Node 专属包 |
| 不适合 | 强依赖 Node 内置模块/本地文件/长计算 | 超长计算/硬粘 Node C++ 扩展（Edge 不支持） | Node-only 包（无 Web 等价） |

> 共识：**能用 Web 标准 API 就别绑 Node 专属**。Edge = 小而美、短而快；重计算交给后端任务队列/批处理。

---

## 1) Cloudflare Workers（含 Durable Objects）☁️

### 1.1 Hello Worker
```ts
// src/index.ts（模块 Worker）
export default {
  async fetch(req: Request, env: Env, ctx: ExecutionContext) {
    return new Response('Hello from Workers');
  }
}
```

**wrangler.toml**
```toml
name = "awesome-worker"
main = "src/index.ts"
compatibility_date = "2025-01-01"

# 绑定示例（按需启用）
kv_namespaces = [{ binding = "KV", id = "xxxxxxxx" }]
d1_databases   = [{ binding = "DB", database_id = "xxxx" }]
r2_buckets     = [{ binding = "BUCKET", bucket_name = "assets" }]

# Durable Object（下节示例）
[[durable_objects.bindings]]
name = "COUNTER"
class_name = "Counter"
```

开发与部署：
```bash
pnpm add -D wrangler
pnpm wrangler dev
pnpm wrangler deploy
pnpm wrangler tail            # 实时日志
```

### 1.2 KV / D1 / R2 / Queues 速用
```ts
// KV：键值缓存 / 配置
await env.KV.put('greet', 'hi', { expirationTtl: 60 });
const v = await env.KV.get('greet');

// D1（SQLite）：轻量结构化数据
const { results } = await env.DB.prepare('select * from posts where id=?').bind(1).all();

// R2（S3 兼容对象存储）
await env.BUCKET.put('logo.png', await fetch('https://...').then(r=>r.arrayBuffer()));

// Queues：异步任务
await env.MY_QUEUE.send({ type: 'email', to: 'user@example.com' });
```

### 1.3 Durable Objects（强一致“单主房间”）
```ts
export class Counter {
  state: DurableObjectState;
  constructor(state: DurableObjectState) { this.state = state; }

  async fetch(req: Request) {
    const n = (await this.state.storage.get<number>('n')) ?? 0;
    if (new URL(req.url).pathname === '/inc') {
      await this.state.storage.put('n', n + 1);
      return new Response(String(n + 1));
    }
    return new Response(String(n));
  }
}

// 路由到 DO
export default {
  async fetch(req: Request, env: Env) {
    const id = env.COUNTER.idFromName('global');
    const stub = env.COUNTER.get(id);
    return stub.fetch(new URL(req.url).origin + '/inc');
  }
}
```
**用途**：聊天室/协作文档房间、限流计数器、事务性序列号等需要**强一致+有状态**的热点。

### 1.4 常见坑 & 对策
- **Node 专属模块不可用** → 用 Web 标准（`fetch`、`Web Crypto`、`URL`、`ReadableStream`）。  
- **CPU 时间预算** → 流式 I/O、尽量把重活投递到 Queues。  
- **本地文件系统不可用** → R2 或外部存储。  
- **跨区域一致性** → 用 DO 作为**协调点**；其他读走 KV + `stale-while-revalidate`。

---

## 2) Vercel（Edge Functions / Serverless Functions）▲

### 2.1 Next.js App Router（Edge）
```ts
// app/api/hello/route.ts
export const runtime = 'edge';           // 指定 Edge 运行时
export async function GET() {
  return new Response('Hello from Vercel Edge');
}
```

### 2.2 Node Serverless（长尾生态/依赖 Node）
```ts
// pages/api/hello.ts（或 /api/hello.ts）
import type { NextApiRequest, NextApiResponse } from 'next';
export default function handler(req: NextApiRequest, res: NextApiResponse) {
  res.status(200).json({ ok: true });
}
```

### 2.3 平台资源常见组合
- **Vercel KV / Postgres / Blob / Edge Config**：配置用 Edge Config，低延迟 KV 做会话/节流，PG 存业务数据，Blob 放文件。  
- **中间件**：**A/B、国际化、鉴权门卫**（在 Edge 上改写/重定向）。  
- **预览环境**：每个 PR 自动部署，配合 E2E（Playwright）很好用。

### 2.4 Streaming 与 OG
```ts
// app/api/stream/route.ts
export const runtime = 'edge';
export async function GET() {
  const { readable, writable } = new TransformStream();
  const writer = writable.getWriter();
  writer.write(new TextEncoder().encode('chunk-1\n'));
  writer.write(new TextEncoder().encode('chunk-2\n'));
  writer.close();
  return new Response(readable, { headers: { 'content-type': 'text/plain' }});
}
```

### 2.5 注意事项
- **Edge 运行时 API 受限**（不等价 Node）：不要引入 `fs`、`crypto` Node 扩展。  
- **部署包体**：小而快；大依赖放到 Serverless/后端。  
- **长连接/WS**：按官方能力选 Edge 或转托第三方实时服务；流式 SSE 往往更稳。

---

## 3) Deno Deploy（TS 原生，Web 标准友好）🦕

### 3.1 Hello（Web 标准最纯正）
```ts
// main.ts
Deno.serve((_req) => new Response('Hello from Deno Deploy'));
```

### 3.2 Deno KV（轻量一致存储）
```ts
const kv = await Deno.openKv();
await kv.set(['counter'], 1);
const v = await kv.get<number>(['counter']);
```

### 3.3 Cron 与 WebSocket
```ts
// Cron（平台设置中配置）执行到此代码
Deno.cron('nightly-clean', '0 3 * * *', async () => {
  // 清理/归档
});

// WebSocket 原生
Deno.serve((req) => {
  const { socket, response } = Deno.upgradeWebSocket(req);
  socket.onmessage = (e) => socket.send(`echo:${e.data}`);
  return response;
});
```

### 3.4 注意事项
- **依赖策略**：用 URL 导入（`https://deno.land/std` / `esm.sh`）；尽量选 Web 标准库。  
- **文件系统**：不可持久，放 KV/对象存储。  
- **Node-only 包**：不宜使用（除非有 `npm:` 兼容且不依赖 Node 内置）。

---

## 4) 边缘数据与一致性：通用战术 🧠

- **读多写少**：KV + `stale-while-revalidate`；写入后**事件驱动失效**（Tag/Purge）。  
- **强一致热点**：Durable Objects（CF）或“单写者”服务；其余副本读缓存。  
- **会话**：Edge 以 **JWT（HttpOnly Cookie）** 为主；私有数据慎缓存，`Vary: Authorization` 或直接 `private`.  
- **幂等**：写请求带 `Idempotency-Key`；在边缘副本做短暂去重。  
- **流式**：用 `ReadableStream`/SSE 提升首包速度与交互感。

---

## 5) 本地开发、打包与观测 🔬

### 5.1 工具 & 命令
- **Workers**：`wrangler dev | tail | deploy`（本地基于 Miniflare）  
- **Vercel**：`vercel dev | vercel --prod`（或框架内 `next dev`）  
- **Deno**：`deno task dev`（`deno.json` 里写 alias）

### 5.2 观测清单
- **结构化日志**：`requestId`、`userId/tenant`、`edgeRegion`、`duration`。  
- **分布追踪**：在边缘转发时**传递 Trace header**（例如 `traceparent`）。  
- **失败证据**：保留近 N 分钟的采样错误日志；流量突刺下计算**冷启动次数**与尾延迟 P95/P99。

---

## 6) 常见架构模板 🍱

1) **边缘鉴权 + 源站缓存**
- Edge 中间件校验 JWT → 通过则加上租户头 → 源站/后端按租户路由。  
- 公共资源 `s-maxage + stale-while-revalidate`，私有接口 `private, no-store` 或条件请求。

2) **ISR/再验证（静态为主）**
- HTML **短 TTL + Tag 失效**，资源 `immutable`。  
- 变更触发 **Revalidate Tag** 或调用 `/api/revalidate`（Vercel/Workers 等同思路）。

3) **强一致计数/房间**
- DO（CF）或单写服务维持状态；客户端与其他边缘仅做读缓存或订阅。

---

## 7) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 把 Edge 当长任务执行器 | 超时/CPU 超额 | 任务丢给 **队列/后端**，Edge 只做入口与编排 |
| Node-only 依赖塞 Edge | 构建失败/运行时报错 | 选择 **Web 标准替代** 或退回 Serverless/后端 |
| 在边缘写本地文件 | 下一次读不到 | 用 **对象存储/KV** |
| 鉴权响应被 CDN 缓了 | 越权/脏缓存 | `private` 或 `Vary: Authorization`，并区分公私路由 |
| 全局写扩散 | 写后读不一致 | **单写点**（DO/区域主）+ 事件回放/最终一致 |
| 大包体部署 | 冷启动与下载慢 | 拆依赖、按路由/函数分离、仅保留用得到的代码 |

---

## 8) 提交前检查清单 ✅

- [ ] 运行时选择明确（Edge / Serverless / 后端），**依赖与 API 兼容**。  
- [ ] 缓存策略：公共 `s-maxage + swr`，私有 `private` 或条件请求；带好 `ETag`。  
- [ ] 写操作幂等（`Idempotency-Key`）与重试策略。  
- [ ] 强一致热点通过 **Durable Objects/单写者** 解决。  
- [ ] 日志含 `requestId/region/duration`；Trace 头透传无误。  
- [ ] 构建包体控制、流式响应就绪（首包更快）。  

---

## 9) 练习 🏋️

1. 用 **Cloudflare Workers + KV** 做一个“短链服务”：创建接口写 KV、跳转路径读 KV，并给创建接口加 `Idempotency-Key`。  
2. 在 **Vercel Edge** 写一个 **SSE 流式搜索**：边搜边推送，前端用 `EventSource` 渲染增量。  
3. 用 **Deno Deploy** 写 WebSocket Echo，并在 UI 显示 `edgeRegion` 与往返延迟；观察不同区域下的 P95。

---

**小结**：边缘平台的内核是三件事——**Web 标准 API、超短冷启动、就近计算**。把重活交给后端，把状态收敛到可控点（DO/KV/数据库），再用流式与缓存把体验打磨到丝滑，你的应用就能在全球范围“快得像本地”。⚡️🌎
