# 7.1 SvelteKit 实战要点 🧭

本章目标：把 **SvelteKit** 的“路由 + 数据 + 动作（actions）+ 适配器（adapters）”一锅端，形成**可部署可维护**的实战骨架：  
- 文件式路由与目录规范  
- `load` / `actions` / API `+server.ts` 的分工  
- 认证与 `hooks`、Cookie、`locals`  
- SSR / CSR / 预渲染（Prerender）/ 边缘部署  
- 类型安全、缓存、流式与失败处理

---

## 0) 核心心法（一句话版）

- **页面 = 路由文件 + 数据加载 + 动作提交**。  
- **服务器数据只在服务端拿**（`+page.server.ts` / `+server.ts`），客户端只吃页面数据。  
- **所有写操作走 `actions` 或 API**，并配 **渐进增强**（`use:enhance`）。  
- **按部署环境选 Adapter**，静态可预渲染的页面尽量 `prerender`，剩余走 SSR/Edge。  

---

## 1) 目录与路由清单

```
src/
  routes/
    +layout.svelte         # 布局（可嵌套）
    +layout.ts|.server.ts  # 布局级 load（客户端/服务端）
    +page.svelte           # 页面组件（必需）
    +page.ts|.server.ts    # 页面级 load（客户端/服务端）
    +server.ts             # 同路径 API（GET/POST/...）
    login/
      +page.svelte
      +page.server.ts      # actions：表单处理
    posts/
      [id]/+page.svelte    # 动态段 /posts/123
      [id]/+server.ts      # /posts/123 的 API
    api/
      users/+server.ts     # /api/users REST 端点
  hooks.server.ts          # 全局拦截（认证/日志/代理）
  app.d.ts                 # 类型扩展（Locals、PageData）
lib/                       # $lib 别名（共享模块）
static/                    # 静态资源（原样拷贝）
```

> 动态段：`[id]`。捕获全部：`[...rest]`。  
> 同一路径下的 `+server.ts` 是路由同名的 API 端点。

---

## 2) `load`：拿数据的三种姿势

### 2.1 客户端 + 服务端通吃：`+page.ts` / `+layout.ts`
```ts
// src/routes/posts/[id]/+page.ts
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ fetch, params, depends }) => {
  depends('post:detail', params.id);          // 失效标签
  const res = await fetch(`/api/posts/${params.id}`); // 会把 Cookie 等带给同源 API
  if (!res.ok) throw new Error('加载失败');
  return { post: await res.json() };
};
```
> 首次请求在服务端运行，随后在客户端导航时也会运行。**不要**在这里直接读私密环境变量/后端凭据。

### 2.2 **仅服务端**：`+page.server.ts`（推荐首选）
```ts
// src/routes/dashboard/+page.server.ts
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ locals, fetch }) => {
  if (!locals.user) return { status: 401 };
  const [me, reports] = await Promise.all([
    fetch('/api/me').then(r => r.json()),
    fetch('/api/reports?limit=10').then(r => r.json())
  ]);
  return { me, reports };
};
```
> 可以安全使用 `locals`、私密 `fetch`（会携带请求上下文 Cookie）。

### 2.3 布局级数据与“父数据”
```ts
// src/routes/+layout.server.ts
export const load = async ({ locals }) => ({ user: locals.user });

// 任一子页面
export const load = async ({ parent }) => {
  const { user } = await parent(); // 读取父布局数据
  return { greeting: `Hi, ${user.name}` };
};
```

---

## 3) 表单 `actions` 与渐进增强

### 3.1 `actions`（仅服务端）
```ts
// src/routes/login/+page.server.ts
import { fail, redirect, type Actions } from '@sveltejs/kit';

export const actions: Actions = {
  default: async ({ request, cookies, locals }) => {
    const form = await request.formData();
    const username = String(form.get('username') ?? '');
    const password = String(form.get('password') ?? '');

    if (!username || !password) return fail(400, { message: '缺少字段' });

    const ok = await locals.auth.login(username, password);
    if (!ok) return fail(401, { message: '账号或密码错' });

    cookies.set('session', ok.token, { path: '/', httpOnly: true, sameSite: 'lax', secure: true, maxAge: 60*60*24 });
    throw redirect(303, '/dashboard');
  }
};
```

### 3.2 客户端渐进增强
```svelte
<!-- src/routes/login/+page.svelte -->
<script>
  import { enhance } from '$app/forms';
  let message = '';
</script>

<form method="post" use:enhance={({ result }) => {
  if (result.type === 'failure') message = result.data.message;
}}>
  <input name="username" required />
  <input name="password" type="password" required />
  <button>登录</button>
</form>

{#if message}<p class="error">{message}</p>{/if}
```
> 无 JS 时回退为普通表单；有 JS 时 `use:enhance` 拦截、局部更新，自动处理跳转/错误。

---

## 4) API 路由 `+server.ts`

```ts
// src/routes/api/posts/[id]/+server.ts
import { json, error } from '@sveltejs/kit';
import type { RequestHandler } from './$types';

export const GET: RequestHandler = async ({ params, locals, setHeaders }) => {
  if (!locals.user) throw error(401, 'Unauthorized');
  const post = await locals.db.post.find(params.id);
  if (!post) throw error(404, 'Not found');

  setHeaders({ 'cache-control': 'public, max-age=60' }); // 缓存
  return json(post);
};

export const PATCH: RequestHandler = async ({ request, params, locals }) => {
  const body = await request.json();
  const updated = await locals.db.post.update(params.id, body);
  return json(updated);
};
```

---

## 5) 全局钩子：认证、代理、统一错误

```ts
// src/hooks.server.ts
import type { Handle, HandleFetch, HandleServerError } from '@sveltejs/kit';

export const handle: Handle = async ({ event, resolve }) => {
  // 1) 认证：从 Cookie 还原用户
  const token = event.cookies.get('session');
  event.locals.user = token ? await getUserByToken(token) : null;

  // 2) 继续处理
  return resolve(event, { transformPageChunk: ({ html }) => html });
};

export const handleFetch: HandleFetch = async ({ event, request, fetch }) => {
  // 对同源 API 可注入 Header、走网关、记录链路等
  return fetch(request);
};

export const handleError: HandleServerError = ({ error, event }) => {
  console.error('❌', event.url.pathname, error);
  return { message: '服务器打盹了，请稍后再试' }; // 会注入到 +error.svelte
};
```

- 在 `app.d.ts` 给 `App.Locals` 扩类型：
```ts
// src/app.d.ts
declare namespace App {
  interface Locals {
    user: { id: string; name: string } | null;
    db: { /* ... */ };
    auth: { login(u:string,p:string): Promise<{ token: string } | null> };
  }
  // interface PageData { ... } // 可按需扩展
}
```

---

## 6) 错误与跳转

- 抛错：`throw error(404, 'Not found')`  
- 跳转：`throw redirect(303, '/login')`  
- 页面错误 UI：`src/routes/+error.svelte`  
- 未匹配路由：`src/routes/[...catchall]/+page.svelte`（自定义 404）

---

## 7) SSR / CSR / 预渲染

- 在任意 `+layout.ts` / `+page.ts` 指定：
```ts
export const ssr = true;        // 允许 SSR（默认 true）
export const csr = true;        // 允许在客户端水合（默认 true）
export const prerender = true;  // 静态化该路由（可配合 adapter-static）
```
- 适合**纯静态**的页面 `prerender`；需要会话/个性化的走 **SSR/Edge**。  
- 流式：在 `load` 中返回 `Promise`，在 Svelte 中 `{#await data.slow}{/await}` 分段渲染。

---

## 8) 适配器与部署

`svelte.config.js`（或 `svelte.config.ts`）示例：
```ts
import adapter from '@sveltejs/adapter-auto';
// 可替换：@sveltejs/adapter-node / adapter-vercel / adapter-cloudflare / adapter-netlify / adapter-static

const config = {
  kit: {
    adapter,
    alias: { $lib: 'src/lib' }
  }
};
export default config;
```

- **Node 进程**：`adapter-node`（自己托管，能长连接/WS）  
- **Vercel/Netlify**：各自 adapter（函数或边缘）  
- **Cloudflare**：`adapter-cloudflare`（Workers / Pages Functions）  
- **静态站**：`adapter-static`（配合 `prerender`）

---

## 9) 环境变量与配置

```ts
// 仅构建时注入、不会暴露：$env/static/private
import { SECRET_TOKEN } from '$env/static/private';

// 前缀公开（编译期注入到客户端）：$env/static/public
import { PUBLIC_API_BASE } from '$env/static/public';

// 动态（运行时）读取：$env/dynamic/*
import { env } from '$env/dynamic/private';
```
> 公共变量需要以 `PUBLIC_` 前缀命名（避免私密泄漏）。

---

## 10) 缓存与头部

- 任意 `load` / `+server.ts` 中：
```ts
setHeaders({
  'cache-control': 'max-age=60, stale-while-revalidate=600'
});
```
- 静态资源（`static/`）通过 CDN 缓存；接口/SSR 走合适的 `cache-control` 与 ETag（反向代理层可加速）。

---

## 11) 类型安全与 `PageData`

### 11.1 为 `load` 与 `actions` 加类型
```ts
import type { PageServerLoad, Actions } from './$types';

export const load: PageServerLoad = async (e) => { /* ... */ };
export const actions: Actions = { /* ... */ };
```

### 11.2 显式声明页面数据
```ts
// src/routes/posts/[id]/$types.d.ts（可选，或放在 app.d.ts）
export type PostData = { post: { id: string; title: string } };

// +page.server.ts
export const load = (async () => {
  return { post: { id: '1', title: 'Hello' } } satisfies PostData;
})();

// +page.svelte
<script lang="ts">
  export let data: import('./$types').PageData & import('./$types').PostData;
</script>
<h1>{data.post.title}</h1>
```

---

## 12) 常见食谱（拷走即用）

### 12.1 保护路由（需登录）
```ts
// src/routes/protected/+layout.server.ts
import { redirect } from '@sveltejs/kit';
export const load = async ({ locals, url }) => {
  if (!locals.user) throw redirect(303, `/login?next=${encodeURIComponent(url.pathname)}`);
};
```

### 12.2 上传文件（Action）
```ts
// +page.server.ts
export const actions = {
  upload: async ({ request }) => {
    const form = await request.formData();
    const file = form.get('file') as File;
    // 保存到 S3 / R2 / 本地
    return { ok: true, name: file.name };
  }
};
```

### 12.3 外部 API 代理（保密 Token）
```ts
// src/routes/api/github/+server.ts
import { json } from '@sveltejs/kit';
import { GITHUB_TOKEN } from '$env/static/private';

export const GET = async ({ fetch, url }) => {
  const q = url.searchParams.get('q') ?? 'svelte';
  const res = await fetch(`https://api.github.com/search/repositories?q=${q}`, {
    headers: { Authorization: `Bearer ${GITHUB_TOKEN}` }
  });
  return json(await res.json());
};
```

---

## 13) 测试与质量

- **端到端**：Playwright（官方模板内置）  
- **单元**：Vitest + Svelte Testing Library  
- **无障碍**：lint a11y 规则 + Storybook（见 4.1）  
- **类型检查**：`svelte-check`、`tsc --noEmit`  

---

## 14) 反模式与纠偏

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 在 `+page.ts` 里读私密配置/后端接口 | 泄密 / CORS / 403 | 改到 `+page.server.ts` / `+server.ts` |
| 写操作走 `fetch` 直打外部 | 缺少 CSRF/鉴权 | 走 `actions` / 自建 API，统一鉴权 |
| 滥用 `csr=false` 或 `ssr=false` | SEO / 交互受损 | 逐页评估；可混用与分层 |
| 所有页面都不开 `prerender` | 首屏慢 / 浪费 SSR | 能静态化就静态化 |
| 不使用 `locals` | 认证散落、难测试 | 在 `hooks.server.ts` 统一注入与读取 |
| 错误不抛 `error()` | 状态码错、SEO 差 | 用 `error(status, msg)` 与 `+error.svelte` |

---

## 15) 提交前检查清单 ✅

- [ ] 路由清晰；动态段命名规范；`$lib` 别名使用统一。  
- [ ] 服务端读取（私密/会话）集中在 `+page.server.ts` / `+server.ts`。  
- [ ] 表单动作统一走 `actions`，并配 `use:enhance`。  
- [ ] `hooks.server.ts` 注入 `locals.user` / `db` / `auth`，所有页面统一读取。  
- [ ] 适配器与 `prerender/ssr/csr` 策略明确；缓存头设置合理。  
- [ ] 类型齐全（`$types`、`App.Locals`、`PageData`），`svelte-check` 通过。  

---

## 16) 练习 🏋️

1. 为「仪表盘」路由实现：`+layout.server.ts` 注入 `user`，子页面读取 `parent()`。  
2. 用 `actions` 做一个“头像上传”，失败用 `fail(400, {message})` 回显；成功 `redirect(303, '/me')`。  
3. 写一条 API：`/api/search?q=...` 代理外部搜索，给结果加 `cache-control`，并在页面 `+page.ts` 以 `{#await}` 实现流式显现。  

---

**小结**：SvelteKit 的威力在“**路由即架构**”。把读取与写入**分层**、把私密逻辑**收口到服务端**、把可静态的**提前渲染**，再根据部署环境挑 Adapter，你就能做出既快又稳、代码量还少的现代应用。🚀
