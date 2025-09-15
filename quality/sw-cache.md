# 17.1 Service Worker / Cache / Background Sync 🧰⚡

目标：把 **离线能力、缓存策略、后台重试** 三件事做成“稳得起、快得出、断网也能用”的 PWA 套件。你将获得：最小可跑骨架 + 可抄的缓存策略 + 背景同步队列范式 + 监测与排错清单。

---

## 0) 核心概念速查

- **Service Worker（SW）**：运行在页面外的脚本，拦截 `fetch`、管理 Cache、收消息、做推送/同步。HTTPS Only。
- **Cache Storage API**：`caches.open(name).put(Request, Response)`，与浏览器 HTTP 缓存不同，**完全由你控制**。
- **同步族**  
  - **Background Sync（一次性）**：断网时把操作排队，连上网后由 SW 的 `sync` 事件**重放**。  
  - **Periodic Background Sync（周期性）**：PWA 安装后、满足节电与网络条件时**定期唤醒**刷新数据（平台限制较多，主要在 Chromium 系列可用）。

> 心法：**静态资源预缓存，数据走 SWR；写请求排队，连网再补；导航场景要有离线兜底页**。

---

## 1) 最小可跑骨架（注册 + manifest + SW 壳）

### 1.1 前端注册
~~~html
<!-- index.html -->
<link rel="manifest" href="/manifest.webmanifest">
<script>
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js', { scope: '/' });
  });
}
</script>
~~~

### 1.2 manifest（把站点装进主屏）
~~~json
{
  "name": "Awesome FE Book",
  "short_name": "AFB",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#111111",
  "icons": [
    { "src": "/icons/192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
~~~

### 1.3 服务工作线程骨架
~~~js
// sw.js
const VERSION = 'v1.0.0';
const PRECACHE = `precache-${VERSION}`;
const RUNTIME = `runtime-${VERSION}`;
const OFFLINE_URL = '/offline.html';

// 安装：预缓存应用壳
self.addEventListener('install', (event) => {
  event.waitUntil((async () => {
    const cache = await caches.open(PRECACHE);
    await cache.addAll([
      '/', '/offline.html',
      '/assets/app.css', '/assets/app.js',
      '/icons/192.png'
    ]);
    // 可选：启用导航预加载，降低首包等待
    if ('navigationPreload' in self.registration) {
      await self.registration.navigationPreload.enable();
    }
  })());
  self.skipWaiting(); // 新版本就绪后尽快接管
});

// 激活：清理旧缓存
self.addEventListener('activate', (event) => {
  event.waitUntil((async () => {
    const keys = await caches.keys();
    await Promise.all(keys
      .filter(k => ![PRECACHE, RUNTIME].includes(k))
      .map(k => caches.delete(k)));
    self.clients.claim();
  })());
});
~~~

---

## 2) 运行时缓存策略（读多写少的世界）

### 2.1 路由级分流（导航 / 静态 / API / 图片）
~~~js
self.addEventListener('fetch', (event) => {
  const req = event.request;

  // 仅处理 GET；POST/PUT 等交给 §4 背景同步
  if (req.method !== 'GET') return;

  const url = new URL(req.url);
  const isHTMLNavigation = req.mode === 'navigate';

  // 1) 页面导航：Network-First + 离线兜底
  if (isHTMLNavigation) {
    event.respondWith((async () => {
      try {
        // 若支持导航预加载，先用
        const preload = await event.preloadResponse;
        if (preload) return preload;
        const net = await fetch(req);
        const cache = await caches.open(RUNTIME);
        cache.put(req, net.clone());
        return net;
      } catch {
        // 离线：返回上次缓存或兜底页
        const cache = await caches.open(RUNTIME);
        return (await cache.match(req)) || (await caches.match(OFFLINE_URL));
      }
    })());
    return;
  }

  // 2) 静态资源（指纹文件）：Cache-First
  if (/\.(?:css|js|woff2|png|jpg|webp|avif)$/.test(url.pathname)) {
    event.respondWith(cacheFirst(req));
    return;
  }

  // 3) API GET：Stale-While-Revalidate（SWR）
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(staleWhileRevalidate(req));
    return;
  }

  // 4) 其他：直接放行
});

async function cacheFirst(req) {
  const cache = await caches.open(PRECACHE);
  const hit = await cache.match(req);
  if (hit) return hit;
  const resp = await fetch(req);
  cache.put(req, resp.clone());
  return resp;
}

async function staleWhileRevalidate(req) {
  const cache = await caches.open(RUNTIME);
  const hit = await cache.match(req);
  const prom = fetch(req).then((res) => {
    cache.put(req, res.clone());
    return res;
  }).catch(() => null);
  return hit || prom || fetch(req); // 三重兜底
}
~~~

### 2.2 图片离线占位 & 限制缓存大小
~~~js
const IMG_FALLBACK = '/assets/placeholder.png';

self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);
  if (/\.(?:png|jpg|jpeg|gif|webp|avif|svg)$/.test(url.pathname)) {
    event.respondWith((async () => {
      try {
        return await cacheFirst(event.request);
      } catch {
        return (await caches.match(IMG_FALLBACK)) || new Response('', { status: 404 });
      }
    })());
  }
});

// 小型 LRU：控制运行时缓存大小（示例，最多 200 条）
async function trimCache(name, max = 200) {
  const cache = await caches.open(name);
  const keys = await cache.keys();
  while (keys.length > max) {
    await cache.delete(keys.shift());
  }
}
~~~

---

## 3) 用 Workbox 少写错字（可选但很香）

~~~js
// sw.js with Workbox
import { precacheAndRoute } from 'workbox-precaching';
import { registerRoute, setDefaultHandler } from 'workbox-routing';
import { StaleWhileRevalidate, CacheFirst, NetworkFirst } from 'workbox-strategies';
import { ExpirationPlugin } from 'workbox-expiration';

// __WB_MANIFEST 由构建期注入
precacheAndRoute(self.__WB_MANIFEST || []);

registerRoute(
  ({ request }) => request.mode === 'navigate',
  new NetworkFirst({ cacheName: 'pages', networkTimeoutSeconds: 3 })
);

registerRoute(
  ({ request }) => ['style','script','font'].includes(request.destination),
  new CacheFirst({
    cacheName: 'assets',
    plugins: [new ExpirationPlugin({ maxEntries: 200, purgeOnQuotaError: true })]
  })
);

registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new StaleWhileRevalidate({ cacheName: 'api' })
);

setDefaultHandler(new StaleWhileRevalidate());
~~~

> Workbox 还能直接上 **BackgroundSyncPlugin**，省去你手撸队列（见下一节）。

---

## 4) 写请求的正确姿势：Background Sync 队列

### 4.1 手写最小队列（IndexedDB + `sync` 事件）
- 页面把 **失败的写操作**（POST/PUT/DELETE）写入 IndexedDB 队列，然后请求 SW 注册 `sync`。
- SW 被系统唤醒后，从队列取出，**按序重放**；成功即删除，失败指数退避。

~~~js
// app-queue.js（页面环境）
async function enqueueOffline(req) {
  const body = await req.clone().json();
  const item = { url: req.url, method: req.method, body, headers: [...req.headers], ts: Date.now() };
  const db = await openDB('queue', 1, { upgrade(db){ db.createObjectStore('outbox', { keyPath: 'ts' }); }});
  await db.put('outbox', item);
  // 请求一次性同步
  const swReg = await navigator.serviceWorker.ready;
  if ('sync' in swReg) await swReg.sync.register('outbox-sync');
}

// 使用：提交失败或 offline 时调用
async function safeFetch(req) {
  try {
    return await fetch(req);
  } catch {
    await enqueueOffline(req);
    return new Response(JSON.stringify({ queued: true }), { status: 202 });
  }
}
~~~

~~~js
// sw.js（队列重放）
self.addEventListener('sync', (event) => {
  if (event.tag === 'outbox-sync') {
    event.waitUntil(flushQueue());
  }
});

async function flushQueue() {
  const db = await openDB('queue', 1);
  const tx = db.transaction('outbox', 'readwrite');
  const store = tx.objectStore('outbox');
  let cursor = await store.openCursor();
  while (cursor) {
    const { url, method, body, headers } = cursor.value;
    try {
      const res = await fetch(url, { method, headers, body: JSON.stringify(body) });
      if (res.ok || res.status === 409) await cursor.delete(); // 幂等：409 也算“完成”
    } catch {/* 保留待下次重放 */}
    cursor = await cursor.continue();
  }
  await tx.done;
}
~~~

> 服务端要支持**幂等键**（`Idempotency-Key`）或自然幂等（PUT），避免重放造成重复扣费/下单。

### 4.2 Workbox 背景同步（更省心）
~~~js
import { BackgroundSyncPlugin } from 'workbox-background-sync';
import { registerRoute } from 'workbox-routing';
import { NetworkOnly } from 'workbox-strategies';

const bgSync = new BackgroundSyncPlugin('outbox', {
  maxRetentionTime: 24 * 60 // 分钟
});
registerRoute(({ url, request }) =>
  url.pathname.startsWith('/api/') && request.method !== 'GET',
  new NetworkOnly({ plugins: [bgSync] }),
  'POST'
);
~~~

### 4.3 Periodic Background Sync（周期刷新，平台受限）
- 仅在 **已安装的 PWA**、充电 + Wi-Fi 等条件下触发，**不要**依赖它做关键业务。
~~~js
// 页面：申请权限并注册
if ('permission' in navigator && 'serviceWorker' in navigator) {
  const reg = await navigator.serviceWorker.ready;
  if ('periodicSync' in reg) {
    const status = await navigator.permissions.query({ name: 'periodic-background-sync' });
    if (status.state === 'granted') {
      await reg.periodicSync.register('refresh-content', { minInterval: 6 * 60 * 60 * 1000 });
    }
  }
}
// SW：处理事件
self.addEventListener('periodicsync', (event) => {
  if (event.tag === 'refresh-content') {
    event.waitUntil(fetch('/api/refresh').then(r => r.ok && caches.open('api').then(c => c.put('/api/feed', r.clone()))));
  }
});
~~~

---

## 5) 安全与隐私边界

- **不要缓存私有/鉴权响应**（包含 `Authorization`/`Set-Cookie` 的），或使用 `Cache-Control: private, no-store`。  
- 仅缓存**幂等 GET**；写操作走队列并带 **Idempotency-Key**。  
- 外域资源需 **CORS** 正确；CDN 代理时注意 `Vary` 头的碎片化。  
- 尊重存储配额：检测 `navigator.storage.estimate()`；命中配额时做 LRU 清理。

---

## 6) 调试排错与版本升级

- Chrome DevTools → **Application** → Service Workers/Cache Storage；勾选 **Update on reload** 调试更新流程。  
- 模拟离线/慢网速：**Network** 面板。  
- 使用 `postMessage` 打可观测日志到页面；或 `clients.matchAll()` 广播版本号。  
- 升级策略：新 SW `skipWaiting()` + `clients.claim()`；但**要确保**预缓存已完成再接管，避免“黑屏瞬间”。

---

## 7) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 全站请求都拦截缓存 | 数据陈旧/越权 | 仅缓存公共 GET；对导航/API分流策略 |
| 只清新缓存、不清旧缓存 | 存储溢出 | **版本化 cache 名**，激活时清理 |
| 断网直接 500 | 离线全白屏 | 导航 Network-First + `offline.html` 兜底 |
| 写请求直接失败 | 用户数据丢 | Background Sync 队列 + 幂等键 |
| 周期同步当闹钟 | 触发不稳定 | Treat as best-effort，核心刷新走前台或推送驱动 |
| 把敏感响应放 Cache | 信息泄露 | 为私有数据设 `no-store`，或 Cache Key 加租户隔离 |
| 缓存体积失控 | 浏览器清空你的仓 | LRU + `ExpirationPlugin` + 配额监控 |

---

## 8) 验收清单 ✅

- [ ] 首次访问可离线回到“最近一次页面”或 `offline.html`。  
- [ ] 静态资源**预缓存**成功，运行时缓存命中率有监控。  
- [ ] API GET 采用 **SWR**；图片有占位与大小上限；运行时缓存有**逐出策略**。  
- [ ] 写请求失败会进入**队列**，网络恢复后成功重放；服务端有**幂等**。  
- [ ] 有观测：离线命中次数、队列长度、重放成功率、缓存命中率。  
- [ ] 版本升级不黑屏；旧缓存清理OK；配额不足有回退策略。  

---

## 9) 练习 🏋️

1. 给你的文档站添加 SW：导航 Network-First + `offline.html`，测断网体验。  
2. 给 `/api/products` 上 SWR，并记录缓存命中率；为图片加占位与 LRU。  
3. 为 `POST /api/cart/add` 加入 Background Sync，开飞行模式、点三次、恢复网络，验证后端幂等。  

---

**小结**：把 **“预缓存壳 + SWR 数据 + 背景重试 + 离线兜底”** 这四块拼上，你的 Web App 就具备了**可离线、抗抖动、延迟低**的体质。断网不慌，弱网不慢，强网更快。🚀
