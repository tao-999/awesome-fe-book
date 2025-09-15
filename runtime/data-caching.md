# 14.2 缓存 / 分页 / 一致性策略 🧠⚡

目标：把 **API 层的“速度”和“正确”**同时拿下。三件套：
- **缓存**：HTTP / CDN / 客户端 / 服务端多层协同，少走弯路。
- **分页**：游标优先，稳定排序，吞吐与正确兼顾。
- **一致性**：读写顺序、并发冲突、重试幂等，系统要“讲道理”。

---

## 1) 多层缓存的角色与边界

| 层 | 典型实现 | 优势 | 注意点 |
|---|---|---|---|
| 浏览器本地 | HTTP 缓存、Service Worker (SW)、IndexedDB | 0ms 命中、离线 | `Cache-Control`、`Vary`、SW 版本管理 |
| 共享缓存 | CDN（Akamai/Cloudflare/Vercel Edge）、反向代理（NGINX/Varnish） | 大流量卸载 | `s-maxage`、`stale-while-revalidate`、鉴权 |
| BFF/应用层 | 内存/LRU、Redis/KV、应用级 DataLoader | 避免热点穿透 | TTL、标签失效、雪崩与击穿 |
| 数据源 | DB/搜索引擎 | 强一致 | 慢、昂贵，不要每次直打 |

> **策略金句**：**能让共享缓存扛的，让共享缓存扛**；不能缓存的，尽量**条件请求**（304），最后再上应用级缓存。

---

## 2) HTTP 缓存语义速查

### 2.1 常用头
- `Cache-Control`:  
  - `public, max-age=60, stale-while-revalidate=600`（浏览器+CDN都可缓存）  
  - `private, max-age=0, no-store`（强制禁用）  
  - 对 CDN 使用 `s-maxage=60` 覆盖共享缓存 TTL
- 条件请求：`ETag` + `If-None-Match`、`Last-Modified` + `If-Modified-Since`
- 变体：`Vary: Accept-Encoding, Origin, Authorization`（谨慎增加，避免缓存碎片化）
- 授权响应：默认**不缓存**；可用 `Cache-Control: private, max-age=30`（仅浏览器）

### 2.2 典型配置（详情页）
```
Cache-Control: public, max-age=60, stale-while-revalidate=600
ETag: "p_123@rev_42"
```
客户端下一次：
- 带 `If-None-Match: "p_123@rev_42"` → 命中返回 `304 Not Modified`，极省带宽。

### 2.3 列表/搜索的缓存
- 热门列表：可 `public, s-maxage=30` 给 CDN，浏览器 `max-age=0` 以免用户看到过旧列表。
- 个性化/私有列表：**只用浏览器缓存** 或直接 `no-store`；改走 **本地 index + 增量同步**（见 §6）。

---

## 3) 失效与再验证：从“清缓存是世界上最难的事”毕业

### 3.1 事件驱动失效（推荐）
- 为资源打**标签**（Tag）：`product:123`, `category:book`
- 写入/变更后发布失效事件 → CDN/BFF 根据 Tag **批量 purge**  
  （Cloudflare Cache-Tag、Fastly Surrogate-Key、Vercel Revalidate Tag）

### 3.2 基于版本的被动失效
- `ETag` 使用 **版本号/内容哈希**；每次读取都条件请求（304），“看似新鲜”的老副本也能被服务器拒绝。
- 适用 **读多写少** 场景，避免主动清缓存的复杂性。

### 3.3 SWR / SIE
- `stale-while-revalidate`：先回老数据，后台异步拉新（**快且最终一致**）。
- `stale-if-error=600`：上游故障时回旧副本，抗抖动。

---

## 4) 应用层缓存：命中要稳、失效要准

### 4.1 Key 设计
- 读：`cacheKey = ${route}?${canonicalQuery}`（排序/默认值标准化）
- 写：**两段式**  
  1) **写数据库**（成功）  
  2) **发布失效**（按 Tag 或 Key 前缀）  
  ——失败可重试，保证**先写后失效**

### 4.2 防击穿/穿透/雪崩
- **热点预热**：部署后先拉取热门 Key。
- **空值缓存**：对不存在的资源缓存短 TTL 的“空”，挡住暴力穷举。
- **抖动 TTL**：`baseTTL * (0.9 ~ 1.1)` 随机化，防同时过期雪崩。
- **单飞阀**：对同一 Key 的并发 miss 合并为一次（请求合并）。

---

## 5) 分页：游标优先，稳定排序

### 5.1 为什么不用 offset？
- `offset/limit` 在**频繁插入/删除**时会**翻页错位**，性能在大偏移量下也差。

### 5.2 游标（Keyset）分页规范

**排序键：**使用**单调**且**可比**的字段，通常 `createdAt` + `id` 作为复合键。  
**游标结构：**
```json
{
  "items": [{ "id": "p1" }, { "id": "p2" }],
  "nextCursor": "eyJjcmVhdGVkQXQiOiIyMDI1LTEwLTAxVDA4OjAwOjAwWiIsImlkIjoicDIifQ=="
}
```
- `nextCursor` 是排序键的**序列化并 base64**；**不要**把业务查询直接塞进游标。
- 反向分页需要提供 `prevCursor` 与相反排序。

**SQL（PostgreSQL）示例：**
```sql
-- 正序（created_at, id）
SELECT * FROM products
WHERE (created_at, id) > (:created_at, :id)
ORDER BY created_at, id
LIMIT :limit;
```

**稳定性要点：**
- 时间戳可能同秒并发，**必须加 id 作为次序**。
- 不要允许客户自由排序后还想“游标稳定”，**稳定游标只对固定排序成立**。

### 5.3 分页与缓存的结合
- 列表页缓存 Key：`/v1/products?filter=...&sort=...&limit=20&cursor=abc`  
- **命中率提升**：对“第一页”与常见过滤条件单独加 CDN 缓存；后续页交给客户端本地缓存（见 §6）。

---

## 6) 客户端缓存与脱机策略（SWR/React Query）

- **查询键即契约**：`['products', { filter, sort, cursor }]`
- **聚合窗口**：对同键在短时间内的重复请求合并（`dedupingInterval`）
- **失效驱动**：写成功后 `invalidateQueries(['products'])` → 触发重拉  
- **离线/弱网**：用 IndexedDB + 背景同步（SW），页面用 **SWR（先旧后新）** 上屏。

---

## 7) 一致性：写入顺序、读后可见、并发冲突

### 7.1 一致性等级（对调用者讲清楚）
- **强一致（Read-After-Write）**：同用户写后读立刻可见。  
  - 做法：**写路径**直达主库；**读路径**对写者走**同可用区/同连接池**或 **stickiness（粘滞会话）**。  
- **最终一致**：可能看到旧数据，但在 SLA 时间内收敛。  
  - 做法：SWR、事件驱动重拉、UI 标记“数据可能延迟”。

### 7.2 乐观并发控制（OCC）
- 响应携带 `version` 或 `ETag`；更新时带 `If-Match`：
```
If-Match: "rev_42"
```
- 版本不匹配 → `409 CONFLICT`，前端**拉新版本 + 合并/重试**。

### 7.3 幂等与重试
- **写操作要求 `Idempotency-Key`**：由客户端生成（UUID/ULID），服务端保存（键 + 指纹）  
  - 网络重试/超时重发不会重复下单/扣费。  
- PUT 整体替换天生幂等；**PATCH 必须配幂等键**。

### 7.4 事务与跨边界一致性
- **本地事务** + **Outbox**：写库与记录事件在同事务，异步投递到消息队列，消费者去**清缓存/更新派生表**。  
- **不要追求“绝对一次”**：通过幂等消费与去重表实现**“至少一次 + 幂等 = 看起来一次”**。

---

## 8) 针对不同数据形态的缓存模板

### 8.1 详情资源（读多写少）
- 头部：`public, max-age=60, stale-while-revalidate=600` + `ETag`  
- 失效：按 **Tag**（`product:123`）清除；或版本号上浮到 `ETag`

### 8.2 列表/Feed（变化快）
- 首屏：CDN `s-maxage=15`（仅公共列表）；浏览器 `max-age=0`，配合 SWR  
- 后续页：只做**客户端缓存**，服务端不强缓存

### 8.3 个性化数据（带授权）
- `Cache-Control: private, max-age=0, must-revalidate`  
- 浏览器条件请求节流；CDN 不缓存（或使用 Keyed-By-User 的网关层）

### 8.4 计算昂贵的聚合
- 应用层 KV/Redis，TTL 1~5 分钟 + 抖动；  
- 失效事件触发**精准重算**（例如某个店铺销量变化 → 只重算该店铺榜单）

---

## 9) GraphQL / tRPC 场景补充

- **GraphQL**：用 **Persisted Query（APQ）** 把 Query 摘要化 → CDN 以 hash 为 Key 缓存；分块 `@defer/@stream` 减少首屏等待。  
- **tRPC**：依赖 **React Query/SWR**；在 BFF 层加 **路由级缓存**（输入序列化为 Key），输出携带 `ETag` 以便浏览器条件请求。

---

## 10) 监控与压测：验证“又快又对”

- **命中率**：CDN/BFF 命中率（HIT/MISS/EXPIRED/BYPASS）；  
- **304 比例**：条件请求成功率（越高越省带宽）；  
- **热点键**：前 100 Key QPS/延迟；  
- **分页质量**：翻页重复率、漏读率（基于游标回放测试）；  
- **一致性**：写后读延迟 P95；冲突率（409）与重试成功率。  
- **压测**：模拟突刺在“缓存清空 + 冷启动 + 热门列表”组合下的行为，检查是否雪崩。

---

## 11) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 一上来 `no-store` | 带宽与成本爆炸 | 用 `ETag` + 条件请求；可缓存的都缓存 |
| 只靠 `max-age` | 数据陈旧 | 叠加 `stale-while-revalidate` 或事件失效 |
| CDN + `Set-Cookie` 混用 | 被动禁用缓存 | 用 `Cache-Control: private` 或移除不必要 cookie |
| 列表 offset 分页 | 漏/重 | 改成 **游标 + 稳定排序** |
| 游标包含业务查询 | 安全隐患、耦合 | 只包含**排序断点**（时间戳+id） |
| 全局缓存清空 | 雪崩 | 标签/前缀精确清、逐步失效 |
| 没有幂等 | 重复扣费/下单 | `Idempotency-Key` + 指纹去重 |
| 多副本读写混飞 | 写后不可见 | 写直主库 + 读粘滞/同可用区 |

---

## 12) 提交前检查清单 ✅

- [ ] 详情接口返回 `ETag`，支持 `If-None-Match`；公共列表设置 `s-maxage` 与 `stale-while-revalidate`。  
- [ ] 分页采用 **游标**，游标仅包含**排序断点**；提供 `nextCursor`（和 `prevCursor` 如需）。  
- [ ] 写操作支持 **Idempotency-Key**；更新使用 **If-Match/版本号** 做乐观并发。  
- [ ] 变更后有**事件驱动失效**（Tag/Key 前缀），或使用版本化 ETag 被动失效。  
- [ ] 客户端用 **SWR/React Query**：查询键规范、写后失效、离线可回放。  
- [ ] 观测面板包含：CDN 命中率、304 比例、P95 延迟、409 冲突率、写后读延迟。  

---

## 13) 附：最小参考实现片段

**Node（Koa）给详情页加条件缓存**
```ts
router.get('/v1/products/:id', async (ctx) => {
  const p = await db.products.findById(ctx.params.id);
  if (!p) { ctx.status = 404; return; }

  const etag = `"p_${p.id}@rev_${p.version}"`;
  ctx.set('ETag', etag);
  ctx.set('Cache-Control', 'public, max-age=60, stale-while-revalidate=600');

  if (ctx.get('If-None-Match') === etag) { ctx.status = 304; return; }

  ctx.body = { id: p.id, title: p.title, price: p.price, version: p.version };
});
```

**游标分页（TypeScript）**
```ts
type Cursor = { createdAt: string; id: string };
const encode = (c: Cursor) => Buffer.from(JSON.stringify(c)).toString('base64');
const decode = (s?: string | null) =>
  s ? (JSON.parse(Buffer.from(s, 'base64').toString()) as Cursor) : null;

async function listProducts({ cursor, limit = 20 }: { cursor?: string; limit?: number }) {
  const c = decode(cursor) ?? { createdAt: '0001-01-01T00:00:00Z', id: '' };
  const rows = await sql/* sql */`
    SELECT * FROM products
    WHERE (created_at, id) > (${c.createdAt}, ${c.id})
    ORDER BY created_at, id
    LIMIT ${limit}
  `;
  const next = rows.length === limit
    ? encode({ createdAt: rows[rows.length - 1].created_at, id: rows[rows.length - 1].id })
    : null;
  return { items: rows, nextCursor: next };
}
```

---

**小结**：缓存让系统**快**，分页让数据**稳**，一致性让用户体验**可信**。合理组合 `SWR + 游标 + ETag/If-* + 事件失效 + 幂等/乐观并发`，你的 API 就能在高并发与变更频繁的现实世界里，跑得快、站得稳、说得清。🚀
