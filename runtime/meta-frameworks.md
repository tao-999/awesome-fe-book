# 13.1 Next.js / Nuxt / SvelteKit 对比 🌐⚔️

目标：用**一小时读懂三大元框架**的设计取向、渲染模型与开发体验（DX），并给出**可复制的最小样例**与**选型心法**。  
结论先行：**Next.js = React RSC + 工业级生态**；**Nuxt = Vue 全家桶 + Nitro 万用后端**；**SvelteKit = 极致轻量 + 真·编译友好 UI**。

---

## 0) TL;DR 对照表

| 维度 | **Next.js** | **Nuxt** | **SvelteKit** |
|---|---|---|---|
| 语言/框架 | React + TS | Vue 3 + TS | Svelte + TS |
| 路由形态 | `app/` 文件路由（RSC） | `pages/` 文件路由 + 目录约定 | `+page.svelte/+layout.svelte` 等约定 |
| 数据获取 | **Server Components** + `fetch` 缓存/再验证；Server Actions | `useAsyncData/useFetch`（服务端优先），Server Routes | `load`/`+page.server.ts`，端点 `+server.ts` |
| 渲染模式 | SSR / SSG / ISR / Edge Streaming | SSR / Prerender / **Nuxt Islands** / Nitro ISR | SSR / Prerender / CSR 切换 / Streaming |
| BFF/API | `app/api/route.ts` Route Handlers；中间件 `middleware.ts` | `server/api/*.ts`（Nitro），Route Rules | `+server.ts`（端点）与 `hooks.server.ts` |
| 运行时 | Node / Edge（Vercel/CF） | Nitro 适配 Node/Edge/Workers/Deno 等 | 适配器：Node、Vercel、Netlify、CF 等 |
| 生态内建 | Image/Fonts/Analytics/OG 图 | Image/Fonts/Content/i18n/Pinia 等插件系 | 轻内建，按需装适配器与工具 |
| 默认理念 | **服务端优先 + 分层渲染（RSC）** | **约定式全栈 + 模块化生态** | **最少抽象，编译即优化** |
| 适配场景 | 大型门户、SaaS、多团队协作 | Vue 系业务、中后台、内容站 | 高性能交互、工具站、边缘计算友好 |

---

## 1) 路由与页面骨架

### Next.js（App Router + RSC）
```
app/
  layout.tsx       # 布局（Server Component）
  page.tsx         # 页面（默认 Server Component）
  loading.tsx      # 懒加载占位
  error.tsx        # 错误边界
  route.ts         # 同级 API（可选）
  blog/[slug]/page.tsx
  (marketing)/...  # 并行/分组路由
```

### Nuxt（约定式 `pages/`）
```
pages/
  index.vue
  blog/[slug].vue
middleware/
  auth.ts          # 路由守卫
server/
  api/posts.get.ts # API 路由（Nitro）
  middleware/...   # 服务端中间件
app.vue            # 根组件/布局
```

### SvelteKit（“+ 文件”约定）
```
src/routes/
  +layout.svelte
  +layout.ts            # layout 的 load
  +page.svelte
  +page.ts              # page 的 load
  +server.ts            # 端点（GET/POST...）
  blog/[slug]/
    +page.svelte
    +page.ts
```

---

## 2) 数据获取与渲染模型（最重要！）

### Next.js：RSC（React Server Components）+ Server Actions
- **组件默认在服务端渲染**；客户端交互组件用 `"use client"`.
- `fetch()` 带**缓存与再验证**语义；搭配 `revalidate`/Tag/Path 精准刷新。
- **Server Actions**：表单/动作直达服务器函数，简化 API 层。

```tsx
// app/page.tsx （Server Component）
export default async function Home() {
  const res = await fetch('https://api.example.com/posts', { next: { revalidate: 60 } });
  const posts = await res.json();
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}

// app/actions.ts （Server Action）
'use server';
export async function createPost(fd: FormData) {
  // 直接在服务端运行，无需手搓 API
}
```

### Nuxt：服务端优先的 `useAsyncData` / `useFetch` + Nitro
- Nuxt 在 SSR 期间**先取数后注水**，客户端 hydration 继承状态。
- **Nitro** 做服务路由与部署适配，可本地/边缘同构。

```vue
<!-- pages/index.vue -->
<script setup lang="ts">
const { data: posts } = await useAsyncData('posts', () =>
  $fetch('/api/posts')    // 命中 server/api/posts.get.ts
)
</script>

<template>
  <ul><li v-for="p in posts" :key="p.id">{{ p.title }}</li></ul>
</template>
```

```ts
// server/api/posts.get.ts (Nitro)
export default defineEventHandler(async () => {
  return [{ id: 1, title: 'Hello Nuxt' }];
});
```

### SvelteKit：`load` 分层 + 端点 `+server.ts`
- 页面/布局的 `load` 在**服务端或客户端**按需执行；`+page.server.ts` 强制服务端。
- 端点 `+server.ts` 替代传统 API 路由；表单可用 `Actions`。

```ts
// src/routes/+page.ts
export async function load({ fetch }) {
  const r = await fetch('/api/posts');
  return { posts: await r.json() };
}
```

```ts
// src/routes/api/posts/+server.ts
export async function GET() {
  return new Response(JSON.stringify([{ id: 1, title: 'Hi SvelteKit' }]), {
    headers: { 'content-type': 'application/json' }
  });
}
```

---

## 3) 渲染策略：SSR / 预渲染 / 增量

- **Next.js**：SSG（构建期）、**ISR**（按 `revalidate` 冷启重建）、边缘流式（Streaming）非常成熟。
- **Nuxt**：`nitro` 提供 **Prerender** 与 **ISR 路由规则**，并支持 **Nuxt Islands**（局部岛屿交互）。
- **SvelteKit**：`prerender` 选项对静态化友好，SSR/CSR 可**每路由开关**；流式响应简洁。

示例：

**Next.js ISR**
```tsx
export const revalidate = 120;            // 文件级：2 分钟再验证
```

**Nuxt 路由规则（示意）**
```ts
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/blog/**': { isr: 60 },             // 60s 增量再生
    '/admin/**': { ssr: false }          // 客户端渲染
  }
});
```

**SvelteKit 预渲染**
```ts
// +layout.ts
export const prerender = true;            // 全站静态化（可在子路由覆写）
```

---

## 4) 中间件与边缘能力

- **Next.js**：`middleware.ts` 运行在 Edge（路由改写、A/B、鉴权门卫）；`headers/cookies` API 简化常见操作。
- **Nuxt**：`server/middleware`（服务端）与 `middleware/`（前端路由守卫）；Nitro 适配 Edge/Workers。
- **SvelteKit**：`hooks.server.ts` 全站钩子；可在 **Edge 适配器**上部署（CF Workers 等）。

**示例：Next 中间件**
```ts
// middleware.ts
import { NextResponse } from 'next/server';
export function middleware(req: Request) {
  const url = new URL(req.url);
  if (!url.pathname.startsWith('/admin')) return;
  // 简例：未登录重定向
  return NextResponse.redirect(new URL('/login', url));
}
```

**示例：SvelteKit 全局钩子**
```ts
// src/hooks.server.ts
export async function handle({ event, resolve }) {
  // 鉴权/多租户/国际化……
  return resolve(event);
}
```

---

## 5) 表单与动作（Action）

- **Next.js**：**Server Actions** 直达服务器函数（防 CSRF 需配合框架策略）。
- **Nuxt**：表单向 `server/api/*` 提交；也可用 `useForm` 系列模块。
- **SvelteKit**：**Form Actions**（`+page.server.ts` 导出 `actions`）+ 渐进式增强。

```ts
// SvelteKit: +page.server.ts
export const actions = {
  create: async ({ request }) => {
    const fd = await request.formData();
    // 处理并返回错误/成功
    return { ok: true };
  }
};
```

---

## 6) 资源与优化（Images / Fonts / SEO）

- **Next.js**：`next/image`（自动优化/占位符/懒加载）、`next/font`（子集化），Head 用 `metadata`。
- **Nuxt**：`@nuxt/image`、`@nuxt/fonts`、`@nuxt/seo` 等官方模块一把梭；`<Head>`/`useSeoMeta`。
- **SvelteKit**：按需接第三方（Astro/Images、imagetools），SEO 用 `svelte-meta-tags` 或手搓 `<svelte:head>`。

---

## 7) 运行配置与环境变量

**Next.js**
```ts
// 仅前端可见需以 NEXT_PUBLIC_ 前缀
process.env.NEXT_PUBLIC_API_BASE
```

**Nuxt**
```ts
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    apiSecret: '',          // 仅服务端
    public: { apiBase: '' } // 客户端可见
  }
});
```

**SvelteKit**
```ts
import { PUBLIC_API_BASE } from '$env/static/public';
import { SECRET_TOKEN } from '$env/static/private';
```

---

## 8) 错误边界与 404

- **Next.js**：`error.tsx` / `not-found.tsx`（可路由级定制）。
- **Nuxt**：`error.vue`（全局），或 `showError`/`createError` 抛错。
- **SvelteKit**：`+error.svelte`（路由级）+ 抛 `error(status, message)`。

---

## 9) 最小样例（各 60 秒可跑）

### Next.js —— 页面 + API + 客户端组件
```tsx
// app/page.tsx
import ClientCounter from './ClientCounter';
export const revalidate = 60;
export default async function Page() {
  const posts = await (await fetch('http://localhost:3000/api/posts', { next: { revalidate: 60 }})).json();
  return (
    <>
      <ul>{posts.map((p:any) => <li key={p.id}>{p.title}</li>)}</ul>
      <ClientCounter />
    </>
  );
}

// app/ClientCounter.tsx
'use client';
import { useState } from 'react';
export default function ClientCounter(){ const [n,set]=useState(0); return <button onClick={()=>set(n+1)}>{n}</button>; }

// app/api/posts/route.ts
export async function GET(){ return Response.json([{id:1,title:'Hello Next'}]); }
```

### Nuxt —— 页面 + Server API
```vue
<!-- pages/index.vue -->
<script setup lang="ts">
const { data } = await useAsyncData('posts', () => $fetch('/api/posts'))
</script>
<template>
  <ul><li v-for="p in data" :key="p.id">{{ p.title }}</li></ul>
</template>
```

```ts
// server/api/posts.get.ts
export default defineEventHandler(() => [{ id: 1, title: 'Hello Nuxt' }]);
```

### SvelteKit —— 页面 + 端点
```svelte
<!-- src/routes/+page.svelte -->
<script>
  export let data;
</script>
<ul>{#each data.posts as p}<li>{p.title}</li>{/each}</ul>
```

```ts
// src/routes/+page.ts
export async function load({ fetch }) {
  const posts = await (await fetch('/api/posts')).json();
  return { posts };
}

// src/routes/api/posts/+server.ts
export const GET = async () =>
  new Response(JSON.stringify([{ id: 1, title: 'Hello SvelteKit' }]), { headers: { 'content-type': 'application/json' }});
```

---

## 10) 性能与可维护性心法 🧠⚡

- **数据靠近服务器**：SSR/边缘优先，减少瀑布流请求；**客户端组件最小化**（Next）。
- **按路由分层**：布局缓存 + 页面再验证；细粒度 **Streaming** 让首屏快。
- **稳定定位器**：E2E/组件测试用 `Role/Label` 或 `data-testid`，别绑 class。
- **图片/字体**走专用管线：避免 JS 处理大静态资源。
- **状态管理简化**：Server-first 后，很多状态是“**派生于数据**”，减少全局 store。

---

## 11) 选型建议（务实版）🎯

- 你在 **React 生态**、需要 **Server Components/Actions**、追求**工业级周边** → **Next.js**。
- 你团队 **Vue 熟练**、希望**一体化模块（内容、图像、i18n）**与**灵活后端（Nitro）** → **Nuxt**。
- 你要 **更小包体/更快交互**、**适配边缘**、喜欢**简单直觉** → **SvelteKit**。

> 组织层面：**人效 > 框架优劣**。选团队**现成栈** + **能招到人**的那个，就是更优解。🧑‍💻

---

## 12) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 全部客户端渲染 | 首屏慢、SEO 弱 | 服务端优先；仅交互组件 `"use client"` |
| 滥用全局状态 | 组件难重构 | 以**数据请求**为中心，最小化 store |
| API 四处散落 | 接口耦合混乱 | 将 API 收敛在 BFF/端点层，统一鉴权与缓存 |
| 不管缓存/再验证 | 请求风暴 | 规划 `revalidate` / ISR / ETag / SWR 策略 |
| 只做成功态 | 一上流量就崩 | 空态/错误/限流/重试策略必须测试 |
| 忽略适配器差异 | 部署踩坑 | 选定目标平台（Node/Edge）先跑“Hello World”链路 |

---

## 13) 练习 🏋️

1. 把一个列表页分别用 **Next/Nuxt/SvelteKit** 实现：SSR 首屏 + 60 秒再验证 + 懒加载详情。  
2. 在三者中各写一个“登录后存储态复用”的 E2E 用例（参考 11.2），测首屏与 TTFB。  
3. 将图片/字体切换到**框架内建方案**（next/image、@nuxt/image、SvelteKit 适配器 + imagetools），对比包体与 LCP。

---

**小结**：三家都能满足现代 Web 的 90% 需求。框架只是在**“服务端/客户端的交界处”**提供不同的手感与护栏。选一个顺手的，把“数据 → 渲染 → 交互 → 部署”串起来，你的站点就会快而稳，还优雅。💃🚀
