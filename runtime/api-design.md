# 14.1 REST / GraphQL / tRPC 设计 🧩

这一章给接口“定江山”：**什么时候用谁、怎么写、怎么演进**。一句话结论：
- **REST**：对外开放、强缓存、CDN 友好、生态广。
- **GraphQL**：一次拿齐、多源聚合、非破坏演进。
- **tRPC**：TS 单仓/同构团队的丝滑内网协议，端到端类型全闭环。

---

## 0) TL;DR 选型心法 🎯
- 面向第三方与公开文档、要求浏览器/CDN 缓存 → **REST（配 OpenAPI）**  
- 页面复杂取数（卡片+列表+侧边栏）/移动端省往返 → **GraphQL**  
- 前后端同仓、TypeScript 全家桶、BFF 内部协议 → **tRPC**

> 同一系统可以“混编”：**REST（公开） + GraphQL（聚合层） + tRPC（内部/BFF）**。

---

## 1) 通用“地方法”（三家共用）🧱
- **命名**：资源用复数（`/users`），JSON 字段 `camelCase`。  
- **时间/金额**：UTC ISO-8601（`2025-09-15T00:00:00Z`）；金额用最小单位整数或 decimal 字符串。  
- **分页**：游标优先 `?cursor=&limit=20`；保留 `page/size` 兼容旧客户端。  
- **过滤/排序**：`?filter.status=active&sort=-createdAt,title`。  
- **稀疏字段集**：`?fields=id,title,price`。  
- **一致性/幂等**：`ETag` + `If-*`；写操作支持 `Idempotency-Key`。  
- **错误格式**：统一错误信封（参考 RFC7807）：`code/message/details/requestId`。  
- **观测**：`x-request-id`、结构化日志、慢查询与错误率仪表。  
- **安全**：输入校验（zod/ajv）、速率限制、CORS 白名单、避免 SSRF。  
- **版本策略**：REST `/v1`；GraphQL 字段 `@deprecated`；tRPC 按命名空间 `v1Router`。

---

## 2) REST 设计（OpenAPI 驱动）🚦

### 2.1 URL 与方法规范
```
GET    /v1/products?filter.category=book&sort=-createdAt&limit=20&cursor=abc
GET    /v1/products/{id}
POST   /v1/products                  # 幂等靠 Idempotency-Key
PATCH  /v1/products/{id}
DELETE /v1/products/{id}
GET    /v1/orders/{id}/items         # 子资源
```

### 2.2 缓存与并发
- 强缓存：`Cache-Control: public, max-age=60, stale-while-revalidate=600`  
- 条件请求：`ETag` + `If-None-Match`（读）；`If-Match`（写的乐观并发）  
- 分页链接：`Link: <.../products?cursor=xyz>; rel="next"`

### 2.3 错误信封（示例）
```json
{
  "type": "https://errors.example.com/validation",
  "title": "Validation Failed",
  "status": 400,
  "code": "VALIDATION_ERROR",
  "requestId": "b2ad-42...",
  "details": [{ "path": "price", "message": "must be >= 0" }]
}
```

### 2.4 最小可用（Express + zod）
```ts
import express from 'express';
import { z } from 'zod';
const app = express(); app.use(express.json());

const CreateProduct = z.object({
  title: z.string().min(1),
  price: z.number().nonnegative(),
  tags: z.array(z.string()).default([])
});

app.get('/v1/products', async (req, res) => {
  const { cursor, limit = '20' } = req.query as any;
  res.set('Cache-Control','public, max-age=30, stale-while-revalidate=600');
  res.json({ items: [], nextCursor: null });
});

app.post('/v1/products', async (req, res, next) => {
  try {
    const idemp = req.get('Idempotency-Key'); // 去重用
    const body = CreateProduct.parse(req.body);
    res.status(201).json({ id: 'p_1', ...body });
  } catch (e) { next(e); }
});
```

### 2.5 OpenAPI 片段
```yaml
paths:
  /v1/products:
    get:
      parameters:
        - in: query
          name: cursor
          schema: { type: string, nullable: true }
      responses:
        "200":
          content:
            application/json:
              schema:
                type: object
                properties:
                  items: { type: array, items: { $ref: "#/components/schemas/Product" } }
                  nextCursor: { type: string, nullable: true }
```

> **何时选 REST**：对外开放/三方集成 + CDN/浏览器缓存 + 简洁 CRUD。

---

## 3) GraphQL 设计（Schema 优先）🕸️

### 3.1 SDL 雏形（连接模型）
```graphql
schema { query: Query, mutation: Mutation }

type Query {
  product(id: ID!): Product
  products(after: String, first: Int = 20, filter: ProductFilter): ProductConnection!
}
type Mutation { createProduct(input: CreateProductInput!): CreateProductPayload! }

type Product { id: ID!, title: String!, price: Float!, tags: [String!]!, createdAt: String! }
input ProductFilter { category: String, minPrice: Float, maxPrice: Float }
type ProductEdge { cursor: String!, node: Product! }
type PageInfo { hasNextPage: Boolean!, endCursor: String }
type ProductConnection { edges: [ProductEdge!]!, pageInfo: PageInfo! }

input CreateProductInput { title: String!, price: Float!, tags: [String!] = [] }
type CreateProductPayload { product: Product! }
```

### 3.2 治理与性能
- **N+1 收敛**：Resolver 层使用 **DataLoader** 批量取数。  
- **缓存**：**Persisted Query/APQ** + CDN（按 hash 缓存）；响应 `Cache-Control` Hints。  
- **流式**：`@defer` / `@stream` 分块送达。  
- **演进**：只加不减，弃用走 `@deprecated`，定期清理。  
- **限额**：复杂度/深度限制 + 速率限制；字段级鉴权。

### 3.3 DataLoader 速写
```ts
import DataLoader from 'dataloader';
const productLoader = new DataLoader(async (ids: readonly string[]) => {
  const rows = await db.select('products').whereIn('id', ids as string[]);
  const map = new Map(rows.map(r => [r.id, r]));
  return ids.map(id => map.get(id));
});
export const resolvers = {
  Query: {
    product: (_: any, { id }: any) => productLoader.load(id),
    products: async (_: any, { after, first, filter }: any) => {
      return { edges: [], pageInfo: { hasNextPage: false, endCursor: null } };
    }
  }
};
```

> **何时选 GraphQL**：复杂页面/移动端、多个后端聚合、快速演进需求强。

---

## 4) tRPC 设计（端到端类型安全）🧪

### 4.1 路由与过程
```ts
import { initTRPC } from '@trpc/server';
import { z } from 'zod';
const t = initTRPC.context<{ userId?: string }>().create();

export const appRouter = t.router({
  product: t.router({
    list: t.procedure
      .input(z.object({ cursor: z.string().nullish(), limit: z.number().min(1).max(100).default(20) }))
      .query(async ({ input }) => ({ items: [], nextCursor: null })),
    create: t.procedure
      .input(z.object({ title: z.string().min(1), price: z.number().nonnegative() }))
      .mutation(async ({ input, ctx }) => {
        if (!ctx.userId) throw new Error('UNAUTHORIZED');
        return { id: 'p_1', ...input };
      })
  })
});

export type AppRouter = typeof appRouter;
```

### 4.2 React 客户端（React Query 集成）
```ts
// client.ts
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '../server/appRouter';
export const trpc = createTRPCReact<AppRouter>();

// 组件
function Products() {
  const { data, isLoading } = trpc.product.list.useQuery({ limit: 20 });
  if (isLoading) return <p>Loading…</p>;
  return <ul>{data?.items.map(i => <li key={i.id}>{i.title}</li>)}</ul>;
}
```

> **注意**：tRPC 强依赖 TS 生态，**不适合公开 API** 或跨语言客户端；对外请转 REST/GraphQL。

---

## 5) 鉴权 / 鉴别（AuthN / AuthZ）🔐
- **协议**：OAuth2/OIDC（Auth Code + PKCE）→ 前端拿 `access_token`；服务端校验 `aud/iss/exp`。  
- **会话**：BFF/SSR 场景优先 **HTTP-only Cookie**，配 CSRF 双提交或 SameSite。  
- **权限**：在后端/Resolver/Procedure 严格授权（基于 **租户/角色/资源**），前端只做“显隐”。  
- **多租户**：租户隔离在 **数据层/缓存键** 上，不仅是路由/域名。

---

## 6) 幂等 / 并发控制 🧵
- **写接口**要求 `Idempotency-Key`（支付/下单类）。服务端以键 + 指纹去重。  
- **乐观并发**：ETag（或 `version` 字段）+ `If-Match`；冲突返回 `409 CONFLICT`。  
- **事务边界**：GraphQL mutation 尽量**细粒度**，或在单 mutation 内部事务化。

---

## 7) 分页/筛选/搜索 🍱

### 游标分页响应（通用）
```json
{
  "items": [{ "id": "p1" }, { "id": "p2" }],
  "nextCursor": "eyJpZCI6InAyIiwgImNyZWF0ZWRBdCI6IjIwMjUtMDktMTUuLi4ifQ=="
}
```
- 游标包含 **稳定排序键**（`createdAt + id` 复合），防插入/删除抖动。  
- 支持 `prevCursor` 时，服务端需能“反向遍历”。

---

## 8) 性能与缓存 ⚡
- REST：强缓存（ETag/Last-Modified）、CDN、`stale-while-revalidate`。  
- GraphQL：**Persisted Query** + 客户端缓存（Apollo/Relay），边缘可按 hash 缓。  
- tRPC：交给 **React Query/SWR** 做请求级与窗口聚合；BFF 内部加 **Server Cache**（KV/Redis）与**慢源断路**。  
- 传输：压缩（br/gzip）、HTTP/2/3、最小 JSON、避免巨型 payload（分页/字段集）。

---

## 9) 上传/下载与大对象 📦
- **预签名 URL**（S3/GCS）：前端直传；服务端仅签名与回调验证。  
- **断点续传**：tus/S3 分片；服务端记录上传会话。  
- **下载**：`Content-Disposition` 安全文件名；按租户/权限二审。

---

## 10) 反模式与纠偏 🧨
| 反模式 | 症状 | 纠偏 |
|---|---|---|
| REST 把动词塞 URL（`/getUser`） | 语义错乱 | 资源 + 方法：`GET /users/{id}` |
| REST 返回大而全 | 网络浪费 | `fields=` 稀疏字段集 + 合理分页 |
| GraphQL 一把梭全字段 | N+1、费流量 | Selection 只取用到的；**DataLoader** |
| GraphQL 频繁破坏性变更 | 客户端崩 | **新增不删除** + `@deprecated` |
| tRPC 外放给第三方 | 紧耦合 | 仅内部；对外转 REST/GraphQL |
| 错误风格不统一 | 难排查 | 统一错误信封 + `requestId` |
| 没有幂等 | 重复下单/扣费 | `Idempotency-Key` + 指纹去重 |
| 无限自由过滤 | 注入/慢查询 | 白名单参数 + 上限 + 索引策略 |

---

## 11) 提交前检查清单 ✅
- [ ] 有 **OpenAPI/SDL** 或 tRPC zod 输入定义，CI 校验通过。  
- [ ] 分页采用 **游标**，提供 `nextCursor`；有 `fields=` 稀疏字段集。  
- [ ] 写操作支持 **幂等键**；读写均有 **ETag/If-* ** 或版本号。  
- [ ] 鉴权/鉴别在服务端严格执行；多租户隔离到数据与缓存键。  
- [ ] 观测：`x-request-id`、结构化日志、关键指标（P95、错误率）接入。  
- [ ] GraphQL 有复杂度/深度限制与 **DataLoader**；REST 配好 **Cache-Control**。  

---

## 12) 练习 🏋️
1. 用 **OpenAPI** 定义 `/v1/orders` 的游标分页与筛选；生成 TS 客户端并在前端集成。  
2. 为“商品详情页”做一条 **GraphQL** Query，接入 **Persisted Query** 与 DataLoader，测量 N+1 收敛。  
3. 在 BFF 内用 **tRPC** 暴露 `cart.addItem` / `cart.list`，并通过 **Idempotency-Key** 保证重复点击只记一次。

---

**小结**：协议只是容器，**契约 + 校验 + 缓存 + 鉴权 + 观测** 才是 API 的五脏六腑。把三套工具用在该用的地方，你的前后端数据线就会又快、又稳、还优雅。🚀
