# 30.1 模块联邦 / 共享依赖 / 跨应用通信 🧩🚚📡

> 把一个“大前端巨舰”拆成多支“快艇”：**各自独立发布**，**运行时拼装**，还能**共享依赖与状态**。这章给你能上线的配置、适配层与通信基建。

---

## 0）你要的落地图（一句话版）

- **模块联邦**：运行时 `import()` 远程包（Remote），宿主（Host）按需拉取。
- **共享依赖**：`react/react-dom/zustand/...` 设为 **singleton + 严格版本**，避免多份实例。
- **跨应用通信**：建立**事件总线（CustomEvent/BC）**或**共享 Store（RxJS/Zustand 外挂）**，保证**可观测与可回放**。
- **灰度与回滚**：Remote 走 CDN/Manifest，Host 可**切换版本/降级到本地 Fallback**。

---

## 1）Webpack Module Federation：Host/Remote 最小可跑 🏗️

> 适用：Webpack 5 / Rspack。Vite 见 §2。

### 1.1 Remote 应用（远程暴露）

```js
// apps/profile/webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  entry: './src/bootstrap.tsx', // 仅导入运行必需
  plugins: [
    new ModuleFederationPlugin({
      name: 'profile',
      filename: 'remoteEntry.js',
      exposes: {
        './Card': './src/components/ProfileCard.tsx',
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.3.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.3.0' },
        zustand: { singleton: true, requiredVersion: '^4.5.0' },
      },
    }),
  ],
};
```

```tsx
// apps/profile/src/components/ProfileCard.tsx
export default function ProfileCard({ userId }: { userId: string }) {
  return <div className="card">Profile of {userId}</div>;
}
```

### 1.2 Host 应用（按需引入）

```js
// apps/shell/webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        // 固定地址
        profile: 'profile@https://cdn.example.com/profile/remoteEntry.js',
      },
      shared: {
        react: { singleton: true, strictVersion: true, requiredVersion: '^18.3.0' },
        'react-dom': { singleton: true, strictVersion: true, requiredVersion: '^18.3.0' },
        zustand: { singleton: true, strictVersion: true },
      },
    }),
  ],
};
```

```tsx
// apps/shell/src/routes/user.tsx
import React, { Suspense } from 'react';
const RemoteProfile = React.lazy(() => import('profile/Card')); // 运行时拉

export default function UserPage() {
  return (
    <Suspense fallback={<span>Loading…</span>}>
      <RemoteProfile userId="u_123" />
    </Suspense>
  );
}
```

### 1.3 动态 Remote（灰度 / 多环境）

```ts
// apps/shell/src/mf/loadRemote.ts
export async function loadRemote(name: string, url: string) {
  // 1) 注入 remoteEntry
  await new Promise<void>((resolve, reject) => {
    const s = document.createElement('script');
    s.src = url; s.type = 'text/javascript'; s.async = true;
    s.onload = () => resolve(); s.onerror = () => reject(new Error('load failed: '+url));
    document.head.appendChild(s);
  });

  // 2) 初始化共享（来自 UMD 容器规范）
  // @ts-ignore
  await __webpack_init_sharing__('default');
  // @ts-ignore
  const container = (window as any)[name];
  // @ts-ignore
  await container.init(__webpack_share_scopes__.default);

  return async <T>(expose: string): Promise<T> => {
    const factory = await container.get(expose);
    return factory() as T;
  };
}

// 使用
// const get = await loadRemote('profile', manifest.profile['remoteEntry.js']);
// const Card = await get('./Card');
```

> 建议：把各 Remote 的 `remoteEntry.js` 放进 **环境可切换的 manifest.json**，Host 首屏拉 manifest，之后选择版本（stable/canary）加载。

---

## 2）Vite / Rspack 的 Module Federation ⚙️

- **Vite**：使用 `@originjs/vite-plugin-federation` 或 `@module-federation/vite`。
- **Rspack**：原生支持（与 Webpack5 近似），速度更快。

**Vite 例子：**

```ts
// apps/profile/vite.config.ts
import federation from '@module-federation/vite';

export default defineConfig({
  plugins: [
    federation({
      name: 'profile',
      filename: 'remoteEntry.js',
      exposes: {
        './Card': './src/components/ProfileCard.tsx'
      },
      shared: ['react', 'react-dom', 'zustand']
    })
  ],
  build: { target: 'es2022' }
});
```

```ts
// apps/shell/vite.config.ts
import federation from '@module-federation/vite';

export default defineConfig({
  plugins: [
    federation({
      name: 'shell',
      remotes: {
        profile: 'https://cdn.example.com/profile/remoteEntry.js'
      },
      shared: { react: { singleton: true }, 'react-dom': { singleton: true } }
    })
  ]
});
```

> 注意：Vite 开发模式下的联邦有热更细节差异，**优先验证构建产物**的行为；生产使用 **静态资源缓存 + hash 文件名**。

---

## 3）共享依赖策略（不会炸的那种）🧪

| 包 | 策略 | 原因 |
|---|---|---|
| react / react-dom | `singleton + strictVersion` | 避免多份 React 破坏 Hook 规则 |
| 状态库（zustand/redux） | `singleton` | 全局唯一状态源 |
| UI 库（antd/tailwind runtime） | **非 singleton** 或按需 | UI 可多版本并存以避免升级地狱 |
| 工具库（lodash/dayjs） | 不共享/按需共享 | 体积小，避免版本锁死 |
| 路由（react-router） | `singleton`（可选） | 确保路由上下文一致 |
| i18n（react-i18next） | `singleton` | 多实例可能切换冲突 |

**版本锁**：在 monorepo 用 **pnpm + workspace ranges** 与 **changesets**，把“共享的包”统一版本；Remote 构建时 `requiredVersion` 一律从 `package.json` 读取，别手写。

---

## 4）跨应用通信（六种可靠姿势）📡

### 4.1 CustomEvent（同页、同窗口）

```ts
// bus/events.ts
type Topic = 'user:login' | 'cart:add' | 'theme:change';
export function emit(topic: Topic, payload?: any) {
  window.dispatchEvent(new CustomEvent(topic, { detail: payload }));
}
export function on<T = any>(topic: Topic, handler: (p:T)=>void) {
  const fn = (e: Event) => handler((e as CustomEvent).detail);
  window.addEventListener(topic, fn);
  return () => window.removeEventListener(topic, fn);
}
```

### 4.2 BroadcastChannel（同源多标签/iframe）

```ts
const bc = new BroadcastChannel('mf-bus');
export const bcEmit = (t: string, data?: any) => bc.postMessage({ t, data });
export const bcOn = (h: (t:string,d:any)=>void) => (bc.onmessage = e => h(e.data.t, e.data.data));
```

### 4.3 Shared Store（RxJS/Zustand 外挂）

```ts
// shared-store/index.ts (单实例，作为 shared 暴露)
import { create } from 'zustand';
type State = { theme: 'light'|'dark'; setTheme: (t:State['theme'])=>void };
export const useAppStore = create<State>(set => ({
  theme: 'light',
  setTheme: (t) => set({ theme: t })
}));
```

Remote/Host 都用同一个 `useAppStore`（通过 shared singleton 提供），自然同步。

### 4.4 postMessage（跨源/沙箱 iframe）

```ts
// Host
iframe.contentWindow!.postMessage({ t: 'ping' }, 'https://remote.app');

// Remote（iframe 内）
window.addEventListener('message', (e) => {
  if (e.origin !== 'https://host.app') return; // 安全校验
  // handle e.data
});
```

### 4.5 Service Worker / SharedWorker（实验性）

- **SharedWorker** 可做轻量消息中枢（同源）。
- **Service Worker** 可拦截请求，实现“统一缓存策略/鉴权头”下发。

### 4.6 事件契约（TS 级别约束）

```ts
// bus/contracts.d.ts
export interface Events {
  'user:login': { id: string; role: 'admin'|'user' };
  'cart:add': { sku: string; qty: number };
}
```

> 在发布 Remote 之前跑一轮 **契约测试**：Host 用 `Events` 校验 Remote 发出的事件与载荷结构。

---

## 5）样式与路由隔离 🎨

- **CSS 冲突**：优先 CSS Modules / CSS-in-JS；全局变量前缀化（如 `--appA-*`）。
- **Tailwind**：为每个 Remote 单独构建，限制扫描范围，避免类名冲突；或使用前缀 `tw-`。
- **路由**：Host 只管 **顶层路由**，Remote 处理**子路由**；通过 `basename` 避免路径碰撞。

---

## 6）安全与健壮性 🛡️

- **CSP**：限制 `script-src` 到你信任的 CDN 域名；Remote 走子资源完整性（SRI）不现实（动态），可使用 **签名 manifest**。
- **越权**：Remote 不得直接访问 Host 私有对象；只能通过 **公共桥接层（bus/store/api）**。
- **错误边界**：每个 Remote 外面包一层 `ErrorBoundary`；加载失败提供 **本地 Fallback**。
- **版本兼容**：Host 在加载前校验 Remote 的 `peerManifest`（声明 `react@18` 等），不符合则降级。

---

## 7）发布与回滚 ⏩⏪

- **产物结构**：`remoteEntry.js + chunks/*` 全量上 CDN，文件名携 **内容 hash**。
- **Manifest**：`manifest.json` 描述各环境版本：
  ```json
  {
    "profile": {
      "stable": "https://cdn/app/profile/1.8.3/remoteEntry.js",
      "canary": "https://cdn/app/profile/1.9.0-rc.2/remoteEntry.js"
    }
  }
  ```
- **Host 策略**：优先 `stable`，灰度用户命中 `canary`；失败自动切回 `stable`。
- **回滚**：只需切 manifest 指针（CDN 秒生效）。

---

## 8）监控与可观测 📈

- 埋点：`mf_load_start/success/fail`（带 url 与耗时）、`mf_remote_render_error`。
- 指标：**远程加载成功率**、**TTI 变化**、**依赖冲突率**（版本不匹配次数）。
- 日志：记录 Remote 版本/commit，与线上告警串联。

---

## 9）端到端自测（契约 + 视觉回归）🧪

- **契约测试**：Host 用 `@module-federation/enhanced/runtime` 的 mock 容器，拉取 Remote 的类型声明（或 `d.ts`），校验导出签名。
- **视觉回归**：Playwright + percy/loki 对关键组件截图；Remote 升级不应破坏 Host UI。
- **烟雾用例**：`加载 remote → 渲染 → 触发事件 → 总线收到 → 状态同步`。

---

## 10）Checklist（实战版）✅

- [ ] `react/react-dom` 单例、严格版本  
- [ ] Remote 暴露清单最小化（只暴露稳定契约）  
- [ ] 动态 Remote + manifest + 灰度回滚  
- [ ] 事件总线 + 类型契约 + 采样日志  
- [ ] ErrorBoundary + 本地 Fallback  
- [ ] CSS/路由隔离策略  
- [ ] 监控埋点与远程加载耗时  
- [ ] 契约测试 & 视觉回归  
- [ ] 安全：CSP/跨源校验/只经公共桥接访问

---

## 11）Bonus：把“共享服务”抽成 SDK 🧪

把鉴权、请求、埋点、主题、i18n 做成 **独立 npm 包（内网注册表）**，由 Host 注入实例，Remote 通过 `shared: { '@org/sdk': { singleton: true } }` 复用——**这是真正可演化的“平台层”**。

> 下一节我们可以把 **微前端路由网关 + 动态权限菜单 + 远程图标集** 一起拉起来，做一个“企业级工作台”骨架，支持**多业务团队独立发布**且**壳层零改动**。🚀
