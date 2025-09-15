# 3.1 现代 JS 能力（ESNext 必修） 🚀

本章目标：把**现代 JavaScript（ES2020+）**里真正能提高开发/运行质量的能力揉成一张“能背会用”的速查与范式清单：**模块化、异步并发、集合/迭代、类与私有字段、可选链/空值合并、Abort/Streams、Intl/URL/结构化拷贝**等。所有片段均可直接拷贝。

---

## 0) 运行基线与风格约定 🧱

- **Node ≥ 20，浏览器：近两年主流版本**，默认 **ESM（`type:"module"`）**。  
- **严格模式**、**不可变优先**（复制而非原地改）、**空值显式处理**。  
- 统一工具：`pnpm i -D typescript @types/node`，TS 编译目标 ≥ `ES2022`。

---

## 1) 语法与数据工具箱 🧰

### 1.1 let/const 与 TDZ
- 变量默认 `const`，确需重赋再 `let`；避免 `var`。  
- 暂时性死区（TDZ）：在声明前使用会抛错，帮你提前踩坑。

### 1.2 解构 / Rest / Spread
```js
const user = { id: 1, name: 'Ada', role: 'admin' };
const { name, ...rest } = user;           // rest: { id, role }
const next = { ...user, role: 'owner' };  // 不可变更新
const [head, ...tail] = [1,2,3];          // head=1, tail=[2,3]
```

### 1.3 模板串 & 标签模板
```js
const lang = 'zh';
const url = String.raw`/i18n/${lang}\n`; // 原样反斜杠与换行
```

### 1.4 可选链 ?. 与空值合并 ??
```js
// 避免“0/''/false 被误判为空”
const port = cfg.server?.port ?? 3000;         // 仅当 null/undefined 才用 3000
const name = user.profile?.name ?? '匿名';
```

### 1.5 逻辑赋值 ||= &&= ??=
```js
opts.retries ??= 3;        // 仅在 null/undefined 时赋值
flags.ready &&= check();   // 仅为真时继续赋值
cache[key] ||= compute();  // 若是假值(null/undef/0/''/false)之外？注意与 ??= 区分
```

### 1.6 数字分隔符 / BigInt / 全局
```js
const BUDGET = 12_000_000;
const n = 9007199254740993n;         // BigInt
globalThis.mySingleton = { /* … */ } // 任意环境里的全局
```

### 1.7 Symbol 与枚举式标签
```js
const kType = Symbol('type');
const node = { [kType]: 'leaf' };
```

---

## 2) 集合：Map/Set/WeakMap/WeakSet 🗂️

- `Map`：键可以是对象，迭代有序；`Object` 适合记录/结构化数据，`Map` 更适合映射关系。  
- `WeakMap`：键为对象且**弱引用**，适合私有数据/缓存，不可枚举。

```js
const seen = new Set();
const memo = new Map();

function dedup(arr){ return arr.filter(x => !seen.has(x) && seen.add(x)); }

function heavy(x){
  if (memo.has(x)) return memo.get(x);
  const v = compute(x);
  memo.set(x, v);
  return v;
}
```

---

## 3) 迭代协议 / 生成器 / 异步迭代 🔁

### 3.1 可迭代协议
```js
const bag = { a:1, b:2, [Symbol.iterator]: function*(){
  for (const k of Object.keys(this)) yield [k, this[k]];
}};
for (const [k,v] of bag){ /* … */ }
```

### 3.2 生成器与异步生成器
```js
function* idGen(){ let i=0; while(true) yield i++; }
const g = idGen(); g.next(); // {value:0, done:false}

async function* readLines(stream){
  for await (const chunk of stream) yield chunk.toString();
}
```

---

## 4) Promise 并发与 async/await ⚡

### 4.1 并发 map（限制并发度）
```js
async function pMap(inputs, mapper, limit = 5){
  const queue = new Set();
  const out = [];
  for (const x of inputs){
    const p = Promise.resolve().then(() => mapper(x))
      .then(v => { out.push(v); queue.delete(p); });
    queue.add(p);
    if (queue.size >= limit) await Promise.race(queue);
  }
  await Promise.all(queue);
  return out;
}
```

### 4.2 组合原语
```js
// all: 全部成功；any: 任一成功；allSettled: 全部结束
const [a,b] = await Promise.all([fa(), fb()]);
const first = await Promise.any([ping1(), ping2(), ping3()]);
const results = await Promise.allSettled(tasks);
```

### 4.3 微任务 vs 宏任务
```js
queueMicrotask(() => { /* 微任务：在渲染前、then 同队列 */ });
setTimeout(() => { /* 宏任务：下一轮事件循环 */ }, 0);
```

---

## 5) Fetch + Abort + 超时与重试 🌐

```js
export async function request(input, { method='GET', json, timeout=8000, retries=1, signal } = {}){
  const ac = new AbortController();
  const t = setTimeout(() => ac.abort(new DOMException('Timeout','AbortError')), timeout);

  try {
    const res = await fetch(input, {
      method,
      headers: json ? { 'content-type':'application/json' } : undefined,
      body: json ? JSON.stringify(json) : undefined,
      signal: signal ?? ac.signal
    });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();
  } catch (e){
    if (retries > 0 && (e.name === 'AbortError' || /HTTP 5\d{2}/.test(e.message))){
      return request(input, { method, json, timeout, retries: retries-1, signal });
    }
    throw e;
  } finally {
    clearTimeout(t);
  }
}
```

> 约定：**所有可取消操作都接受 `AbortSignal`**；复用上层 `signal` 可实现链式取消。

---

## 6) 顶层 await / 动态导入 / JSON 模块 📦

```js
// 顶层 await（仅 ESM）
const { default: cfg } = await import('./config.json', { assert: { type: 'json' } });

// 动态按需：仅在用到时加载重包
async function useChart(){
  const { Chart } = await import('chart.js');
  return new Chart(/* … */);
}
```

---

## 7) 类、字段与私有成员 🏷️

```js
class Counter {
  #n = 0;                      // 私有字段
  static instances = 0;        // 静态字段
  static { this.instances = 0; } // 静态初始化块

  get value(){ return this.#n; }
  inc(step = 1){ this.#n += step; return this; }
}
```

> 私有字段以 `#` 声明，无法从外部/子类访问；避免“约定私有”（下划线）的脆弱性。

---

## 8) 错误与资源管理 🧯

```js
try {
  // …
} catch {                        // 可选捕获绑定（无需命名 e）
  // 记录/兜底
} finally {
  await close();                 // 支持在 finally 里 await
}
```

> 模式：**“失败即返回可诊断对象”**（Result/Either）可与 `throw` 结合，专门抛 P0 异常，其余走可恢复分支。

---

## 9) 标准库锦集（易被忽视但超好用） 🧩

```js
structuredClone(obj);                 // 深拷贝（含 Map/Set/Date/RegExp/TypedArray）
const u = new URL('https://a.com/p?q=1'); u.searchParams.set('q','2');

const enc = new TextEncoder().encode('你好');        // UTF-8 编解码
const dec = new TextDecoder('utf-8').decode(enc);

crypto.getRandomValues(new Uint8Array(16));          // 真随机
await crypto.subtle.digest('SHA-256', enc);          // WebCrypto

// 流与压缩（浏览器支持优）：压缩任意字节流
const cs = new CompressionStream('gzip');
const compressed = await new Response(new Blob(['hello']).stream().pipeThrough(cs)).arrayBuffer();

// 国际化
new Intl.DateTimeFormat('zh-CN', { dateStyle:'long' }).format(new Date());
new Intl.NumberFormat('zh-CN', { style:'currency', currency:'CNY' }).format(1234.56);
new Intl.RelativeTimeFormat('zh-CN').format(-3, 'day');
```

---

## 10) DOM 事件与性能细节 🧠

```js
el.addEventListener('scroll', onScroll, { passive: true }); // 不阻塞滚动
btn.addEventListener('click', onClick, { once: true });     // 只触发一次
```

- **防抖/节流**（微任务版防抖）：
```js
function microDebounce(fn){
  let queued = false;
  return (...args) => {
    if (queued) return;
    queued = true;
    queueMicrotask(() => { queued = false; fn(...args); });
  };
}
```

---

## 11) 编码范式与对照表 🧭

| 目标 | 推荐写法 | 说明 |
|---|---|---|
| 空值兜底 | `a ?? b` | 不误伤 0/''/false |
| 可选访问 | `obj?.foo?.bar()` | 避免“地狱中的 && &&” |
| 并发拉取 | `Promise.all([...])` | I/O 并行 |
| 竞速优先 | `Promise.any([...])` | 任一成功即返回 |
| 取消请求 | `fetch(url,{signal})` | 结合 `AbortController` |
| 深拷贝 | `structuredClone(x)` | 比 `JSON.parse(JSON.stringify())` 更可靠 |
| 单次事件 | `{ once:true }` | 自动移除监听 |
| 流式处理 | Web Streams | 边拉边算，省内存 |

---

## 12) 提交前检查清单 ✅

- [ ] 模块使用 ESM；必要时动态导入。  
- [ ] 所有潜在长耗时操作可被取消（`AbortSignal`）。  
- [ ] 可选链/空值合并替代“手搓兜底”；不误伤合法假值。  
- [ ] Map/Set/WeakMap 用于映射/缓存/私有数据。  
- [ ] 并发控制有上限；避免“风暴式同时发 100 个请求”。  
- [ ] 使用 `structuredClone/URL/Intl` 等标准工具而非第三方轮子。  

---

## 13) 练习 🏋️

1) 把项目里的“工具函数库”替换为 **标准库能力**（`URL`、`Intl`、`structuredClone`）。  
2) 把三处串行 `await` 改为 `Promise.all` 并发；为它们加上 **Abort 超时**。  
3) 写一个 `pMap` 版本，加入 **重试（指数退避）** 与 **并发上限**，并用假接口压测。

---

## 14) 速用片段合集（收藏夹） 📎

```js
// 14.1 等待指定毫秒
export const sleep = (ms, { signal } = {}) => new Promise((res, rej) => {
  const t = setTimeout(res, ms);
  signal?.addEventListener('abort', () => { clearTimeout(t); rej(signal.reason); }, { once:true });
});

// 14.2 超时包装任意 Promise
export const withTimeout = (p, ms) => {
  const ac = new AbortController();
  const t = setTimeout(() => ac.abort('Timeout'), ms);
  return Promise.race([p.finally(() => clearTimeout(t)), sleep(Infinity, { signal: ac.signal })]);
};

// 14.3 安全 JSON 解析
export const safeJson = (s) => { try { return [JSON.parse(s), null]; } catch (e){ return [null, e]; } };

// 14.4 Async 可迭代的分页抓取
export async function* paginate(fetchPage){
  let page = 1, done = false;
  while(!done){
    const { items, nextPage } = await fetchPage(page);
    for (const it of items) yield it;
    done = !nextPage; page = nextPage;
  }
}
```

---

**小结**：把这些 ESNext 能力内化为**默认习惯**：**ESM 模块化、并发优先（可取消）、Map/Set 组织数据、可选链/空值合并处理空值、标准库代替手写轮子**。你写出来的 JS 会更快、更稳、更好维护。🧠⚙️
