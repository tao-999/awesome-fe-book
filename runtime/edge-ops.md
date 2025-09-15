# 15.2 冷启动 / 成本 / 观测 ❄️💰🔭

目标：把 **Edge/Serverless** 的三大难题讲清楚、算明白、量出来。  
关键词：**更小的包、更短的链、更强的证据**。

---

## 0) 概念对齐

- **冷启动（Cold Start）**：平台为你的函数**新开容器/隔离环境** + **加载代码** + **执行模块级初始化**的那段额外延迟。  
- **热启动（Warm Start）**：已有实例复用，只执行用户代码。  
- **尾延迟（Tail Latency）**：P95/P99 的长尾；冷启动是长尾大头。  
- **成本构成**（抽象式）：  
  - `调用数 × 计费时长 × 内存系数`（GB-s 或 ms×MB）  
  - `+ 出站流量（egress） + 存储/队列/KV/日志 + 第三方数据库`。

> 口诀：**冷启动是“编译与初始化”的账，成本是“时间与体积”的账，观测是“证据链”的账。**

---

## 1) 冷启动：测量与预算

### 1.1 怎么量
在入口处打三个时点：`t0=平台进入` → `t1=模块初始化完成` → `t2=业务处理完成`。  
- `cold_init = t1 - t0`（可近似代表冷启动开销）  
- `user_duration = t2 - t1`  
- `TTFB ≈ 网络 + cold_init + user_duration_head`

**结构化日志（通用示例，Web 标准 API）**
```ts
const start = Date.now();
// do module init ...（尽量轻）
export default {
  async fetch(req: Request, env: Env) {
    const t1 = Date.now();
    const res = await handle(req, env);
    const t2 = Date.now();
    console.log(JSON.stringify({
      requestId: crypto.randomUUID(),
      path: new URL(req.url).pathname,
      coldInitMs: t1 - start,
      userMs: t2 - t1
    }));
    return res;
  }
}
```

### 1.2 经验级预算（目标线，不是上限）
- **Edge Isolate（Workers / Edge Functions / Deno Deploy）**：冷启 **10~50ms** 级；包体越小越稳。  
- **Serverless Node 容器**：**几十到数百毫秒**，取决于包体与运行时。  
- P95 TTFB 预算：**静态/缓存命中 ≤ 100ms**；**轻 API ≤ 250ms**；**数据重/聚合 ≤ 500ms**。

---

## 2) 冷启动优化：把“重量”藏起来

### 2.1 包越小越快
- **ESM + Tree Shaking**；尽量 **按路由拆函数**。  
- **避免巨型依赖**：如整包 `lodash`、`aws-sdk`（Edge 无用）等；能写 10 行就别拉 100KB。  
- **动态导入**：把**非首请求必需**的模块 `await import()` 放到**请求路径**而非模块顶层。  
- **禁用顶层重活**：DB 连接、巨型 schema 编译、繁重正则/字典 → 延后到**首次调用**并缓存在全局。

### 2.2 运行时选择
- 纯 HTTP、少 Node 专属能力 → **Edge（Isolate）**。  
- 依赖 Node C++ 扩展 / 大包 / 重 I/O → **Serverless Node** 或**后端服务**。  
- **长计算/批处理** → 丢给 **队列 + 后端 Worker**，边缘只接单。

### 2.3 外部依赖“上岸”
- 数据库：用**无连接/HTTP 驱动**或云厂商的 **serverless driver / 数据代理**。  
- 文件：用 **对象存储（R2/Blob/S3）**，不要碰本地 FS。  
- 会话：优先 **JWT（HttpOnly）+ KV**，别粘内存。

### 2.4 预热不是银弹
- 定时 ping 只能**降低冷启概率**，不能消灭；且会**增加成本**。  
- 正解仍是：**小包 + 拆函数 + 延迟初始化 + 缓存**。

---

## 3) 成本建模：先算再省

### 3.1 抽象公式
```
执行成本 ≈ Σ 函数 ( 调用数 × ceil(执行时长 / 计费粒度) × 内存GB × 单价(每GB-s) )
总成本   ≈ 执行成本 + 出站流量(GB × 单价) + 存储/KV/队列/日志 + 数据库/第三方
```
> 计费粒度、单价随平台/地区变化；模型用于**相对比较与趋势预警**。

### 3.2 降本清单
- **减少调用数**：CDN 缓存、`stale-while-revalidate`、边缘重写（中间件兜底）。  
- **缩短时长**：流式响应（先回头部）、并行外呼、避开 N+1。  
- **降内存档**：Edge/Serverless 可调内存；内存越大**启动更快但更贵**——用压测找拐点。  
- **控制 egress**：图片走对象存储 + 自适应格式/尺寸；API payload 开 gzip/br。  
- **删除“闲置代码”**：每条路由只打包**必需模块**；Monorepo 用 `--filter` 精准构建。  
- **审计日志等级**：信息级打点取样，错误/告警全量；**日志也是钱**。

### 3.3 成本护栏（CI/守夜）
- **包体阈值**：超过 X KB 的函数直接 CI 红灯。  
- **预算回归**：对关键路由跑合成压测，计算 `ms × MB × QPS`，与主干基线对比。  
- **告警**：当“成本/千次调用”或“外呼失败重试率”上扬时推送。

---

## 4) 观测：让每个请求都有“身份证”

### 4.1 指标体系（黄金四件套）
- **Rate**（QPS）  
- **Errors**（4xx/5xx/异常）  
- **Duration**（P50/P95/P99，区分 `coldInit` & `userDuration`）  
- **Saturation**（并发/限额/队列长度）

### 4.2 Trace 传播（W3C Trace Context）
```ts
// 生成/透传 traceparent
function getTraceparent(req: Request) {
  return req.headers.get('traceparent') ?? `00-${crypto.randomUUID().replace(/-/g,'').slice(0,32)}-0000000000000000-01`;
}
```
把 `traceparent` 放进下游请求头，保证端到端闭环（前端→边缘→BFF/DB 代理）。

### 4.3 结构化日志（JSON）
```ts
function log(event: Record<string, unknown>) {
  // 所有平台都支持 console.log；关键是打印 JSON，便于后端收集与索引
  console.log(JSON.stringify({ ts: new Date().toISOString(), ...event }));
}
```

### 4.4 错误分级
- **用户错误**：4xx（参数/鉴权）→ 记录 `code`、不重试。  
- **系统错误**：5xx（上游失败/超时）→ 记录 `retryable=true`，**指数退避**重试或走降级。  
- **边缘超限**（CPU/内存/时间）→ 打印 `quota` 字段，触发**包体/算法审计**。

---

## 5) 平台速用片段

### 5.1 Cloudflare Workers（日志 + 计时）
```ts
export default {
  async fetch(req: Request, env: Env) {
    const t0 = Date.now();
    try {
      const data = await fetch(env.API + '/hello', { headers: { traceparent: getTraceparent(req) }});
      const body = await data.text();
      log({ type: 'ok', path: new URL(req.url).pathname, ms: Date.now() - t0 });
      return new Response(body);
    } catch (e) {
      log({ type: 'err', err: String(e), path: new URL(req.url).pathname });
      return new Response('err', { status: 500 });
    }
  }
}
```

### 5.2 Vercel Edge（流式降低计费时长感知）
```ts
export const runtime = 'edge';
export async function GET() {
  const { readable, writable } = new TransformStream();
  const w = writable.getWriter();
  w.write(new TextEncoder().encode('hello\n'));
  queueMicrotask(() => w.close()); // 立即首包，缩短感知等待
  return new Response(readable, { headers: { 'content-type': 'text/plain' }});
}
```

### 5.3 Deno Deploy（KV 计数 + 观测）
```ts
const kv = await Deno.openKv();
Deno.serve(async (req) => {
  const t0 = Date.now();
  const n = (await kv.get<number>(['hits'])).value ?? 0;
  await kv.set(['hits'], n + 1);
  console.log(JSON.stringify({ path: new URL(req.url).pathname, ms: Date.now()-t0 }));
  return new Response(String(n + 1));
});
```

---

## 6) 数据一致性与冷启的交互

- **写路径**直达**强一致点**（Durable Object/主库/单写服务），返回后再**异步广播失效**。  
- **读路径**先查**就近缓存/KV**，命不中再回源；配合 `stale-while-revalidate` 减少冷启期间排队。  
- **幂等**：写操作加 `Idempotency-Key`，避免重试引发重复计费与数据脏写。

---

## 7) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 单一“大函数”装全站逻辑 | 包体>几MB、冷启抖 | **按路由拆分**，依赖分治，动态导入 |
| 顶层做 DB 连接/大校验 | 冷启峰值高 | **延迟初始化**，结果缓存到全局 |
| 日志全量长文本 | 成本飙升 | 结构化 JSON + 取样；错误全量、信息采样 |
| 把 Edge 当批处理 | 超时/计费爆炸 | 队列 + 后端 Worker；Edge 只编排 |
| 无缓存策略 | QPS 都砸到函数 | CDN `s-maxage` + SWR + 条件请求 |
| 不传 trace | 黑盒，难定位 | W3C traceparent 贯通上下游 |

---

## 8) 提交前检查清单 ✅

- [ ] 关键路由**包体大小** ≤ 阈值（如 200KB），CI 帮你卡。  
- [ ] 模块顶层**无重活**；DB/Schema/字典延迟加载。  
- [ ] 响应使用 **流式** 或分块，首包 ≤ 100ms（能做到就做）。  
- [ ] CDN/Edge 缓存策略明确：`Cache-Control`、`ETag`、`stale-while-revalidate`。  
- [ ] 结构化日志带 `requestId/traceparent/path/ms`；错误分级打印。  
- [ ] 预算守护：有“成本/千次调用”与“P95 TTFB”监控与阈值告警。  
- [ ] 写操作具备 **Idempotency-Key**；读写一致性策略已记录到 Runbook。

---

## 9) 练习 🏋️

1. **拆包练习**：把一个 1MB 的 API 函数拆为 3 条路由，测冷启 P95 下降幅度。  
2. **缓存演练**：为“商品详情”加 `public, max-age=60, stale-while-revalidate=600 + ETag`，观测 egress 与函数调用数下降。  
3. **预算守护**：写一个脚本，读取平台用量，计算“**成本/千次调用**”，当增长 > 20% 时推送到告警渠道。  

---

**小结**：  
- **冷启动**：从“把不必要的东西移出首包”开始。  
- **成本**：从“减少调用与执行时长”入手，缓存是第一生产力。  
- **观测**：从“每个请求都能讲出一条故事”做起。  
把这三件事串成闭环，你的边缘应用就会 **快、稳、省**。🚀
