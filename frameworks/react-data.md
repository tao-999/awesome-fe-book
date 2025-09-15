# 5.1 路由/数据请求：React Router · SWR / React Query

本章目标：把 **React Router（v6.4+ Data APIs）** 与 **SWR / React Query** 三件套揉成一套“**页面 = 路由 + 数据**”的稳定范式：谁负责**导航/作用域/提交**，谁负责**缓存/并发/失效/乐观更新**，怎么**串**起来不打架。

---

## 0) 选型心法（一句话版）

- **页面首要数据、需要提交与错误边界** → **Router 的 `loader/action`**（可串流 `defer`，天然配表单、取消、路由作用域）。
- **跨页面共享缓存、复杂失效/乐观更新/并发** → **React Query（TanStack Query）**。
- **轻量读多写少、函数式“取-改-再取”** → **SWR**（更极简；API 面更小）。
- **组合拳**：**用 `loader` 预取并“喂”进 Query**；**`action` 做提交**；页面内用 **Query/SWR** 管缓存与交互态。

---

## 1) React Router Data Router 基线（v6.4+）

### 1.1 安装与骨架
```bash
pnpm add react-router-dom
```

```tsx
// main.tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import { Home } from './routes/home';
import { PostPage, postLoader, postAction, PostError } from './routes/post';

const router = createBrowserRouter([
  { path: '/', element: <Home /> },
  {
    path: '/posts/:id',
    loader: postLoader,                // 取数
    action: postAction,                // 提交
    element: <PostPage />,
    errorElement: <PostError />        // 仅该路由生效的错误边界
  }
]);

export function App(){ return <RouterProvider router={router} />; }
```

### 1.2 Loader / Action（带取消、状态码与错误边界）
```ts
// routes/post.tsx
import type { LoaderFunctionArgs, ActionFunctionArgs } from 'react-router-dom';

type Post = { id: string; title: string; body: string };

export async function postLoader({ params, request }: LoaderFunctionArgs) {
  const { id } = params;
  if (!id) throw new Response('Bad Request', { status: 400 });

  // 关键：把 request.signal 传给 fetch，导航中断会自动取消
  const res = await fetch(`/api/posts/${id}`, { signal: request.signal });
  if (res.status === 404) throw new Response('Not Found', { status: 404 });
  if (!res.ok) throw new Response('Server Error', { status: 500 });

  return (await res.json()) as Post;
}

export async function postAction({ params, request }: ActionFunctionArgs) {
  const form = await request.formData();
  const body = JSON.stringify(Object.fromEntries(form));
  const res = await fetch(`/api/posts/${params.id}/comments`, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body,
    signal: request.signal
  });
  if (!res.ok) return { error: '提交失败' };  // 会被 useActionData() 接到
  return { ok: true };
}
```

```tsx
// routes/post.tsx（页面）
import { useLoaderData, useActionData, Form, useNavigation } from 'react-router-dom';

export function PostPage(){
  const post = useLoaderData() as { id: string; title: string; body: string };
  const action = useActionData() as { error?: string } | undefined;
  const nav = useNavigation(); // 'idle' | 'loading' | 'submitting'
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>

      <Form method="post" replace>
        <textarea name="content" required />
        <button disabled={nav.state === 'submitting'}>发表评论</button>
        {action?.error && <p role="alert">{action.error}</p>}
      </Form>
    </article>
  );
}

export function PostError(){
  return <p>页面走丢了或服务器打盹了（路由级错误边界）。</p>;
}
```

### 1.3 流式数据 `defer` + `<Await>`（快的先渲，慢的后到）
```ts
// routes/user.ts
import { defer, type LoaderFunctionArgs } from 'react-router-dom';

export async function userLoader({ request }: LoaderFunctionArgs) {
  const profileP = fetch('/api/me', { signal: request.signal }).then(r => r.json());
  const projectsP = fetch('/api/projects?limit=50', { signal: request.signal }).then(r => r.json());
  return defer({
    profile: await profileP, // 先到
    projects: projectsP      // 慢项延后，通过 <Await> 渲染
  });
}
```

```tsx
// routes/user.tsx
import { Suspense } from 'react';
import { useLoaderData, Await } from 'react-router-dom';

export function UserPage(){
  const data = useLoaderData() as { profile: any; projects: Promise<any[]> };
  return (
    <>
      <h2>{data.profile.name}</h2>
      <Suspense fallback={<p>项目加载中…</p>}>
        <Await resolve={data.projects} errorElement={<p>项目加载失败</p>}>
          {(list) => <ul>{list.map((p:any) => <li key={p.id}>{p.name}</li>)}</ul>}
        </Await>
      </Suspense>
    </>
  );
}
```

---

## 2) React Query（TanStack Query）：缓存与失效的主力

### 2.1 安装与 Provider
```bash
pnpm add @tanstack/react-query
```

```tsx
// query/client.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 60_000, gcTime: 5 * 60_000, refetchOnWindowFocus: false, retry: 2 }
  }
});
export const withQuery = (ui: React.ReactNode) => (
  <QueryClientProvider client={queryClient}>{ui}</QueryClientProvider>
);
```

### 2.2 基本用法
```ts
// query/posts.ts
const fetchJSON = (url: string, init?: RequestInit) => fetch(url, init).then(r => {
  if (!r.ok) throw new Error(`HTTP ${r.status}`);
  return r.json();
});

export const postQuery = (id: string) => ({
  queryKey: ['post', id],
  queryFn: () => fetchJSON(`/api/posts/${id}`)
});
```

```tsx
// routes/post-query.tsx
import { useParams } from 'react-router-dom';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { postQuery } from '../query/posts';

export function PostWithQuery(){
  const { id = '' } = useParams();
  const qc = useQueryClient();

  const { data, isPending, error } = useQuery(postQuery(id));
  const addComment = useMutation({
    mutationFn: (content: string) =>
      fetch(`/api/posts/${id}/comments`, { method:'POST', headers:{'content-type':'application/json'}, body: JSON.stringify({ content }) })
        .then(r => { if (!r.ok) throw new Error('提交失败'); }),
    // 乐观更新
    onMutate: async (content) => {
      await qc.cancelQueries({ queryKey: ['post', id] });
      const prev = qc.getQueryData<any>(['post', id]);
      qc.setQueryData(['post', id], (p: any) => ({ ...p, comments: [...(p?.comments ?? []), { id: 'tmp', content, optimistic: true }] }));
      return { prev };
    },
    onError: (_e, _v, ctx) => ctx?.prev && qc.setQueryData(['post', id], ctx.prev),
    onSettled: () => qc.invalidateQueries({ queryKey: ['post', id] })
  });

  if (isPending) return <p>读取中…</p>;
  if (error) return <p>出错：{String(error)}</p>;

  return (
    <>
      <h1>{data.title}</h1>
      <ul>{data.comments?.map((c:any) => <li key={c.id}>{c.content}</li>)}</ul>
      <button onClick={() => addComment.mutate('Hello!')}>乐观评论</button>
    </>
  );
}
```

### 2.3 预取 / 依赖查询 / 无限列表
```tsx
// 悬停预取
<Link
  to={`/posts/${id}`}
  onMouseEnter={() => queryClient.prefetchQuery(postQuery(id))}
/>

// 依赖查询
const user = useQuery(userQuery());
const orgs = useQuery(orgsQuery(user.data!.id), { enabled: !!user.data });

// 无限列表（分页）
const q = useInfiniteQuery({
  queryKey: ['feed'],
  queryFn: ({ pageParam = 1 }) => fetchJSON(`/api/feed?page=${pageParam}`),
  getNextPageParam: (last) => last.nextPage ?? undefined
});
```

---

## 3) SWR：轻量的“取—改—再取”模型

### 3.1 安装与全局配置
```bash
pnpm add swr
```

```tsx
// swr/config.tsx
import { SWRConfig } from 'swr';
const fetcher = (url: string) => fetch(url).then(r => { if (!r.ok) throw new Error('HTTP ' + r.status); return r.json(); });

export const withSWR = (ui: React.ReactNode) => (
  <SWRConfig value={{
    fetcher,
    revalidateOnFocus: true,
    dedupingInterval: 2000
  }}>
    {ui}
  </SWRConfig>
);
```

### 3.2 基本用法 / 变更 / 无限加载
```tsx
import useSWR, { mutate } from 'swr';
import useSWRMutation from 'swr/mutation';

function PostWithSWR({ id }: { id: string }){
  const { data, isLoading, error } = useSWR(`/api/posts/${id}`);

  const { trigger, isMutating } = useSWRMutation(
    `/api/posts/${id}/comments`,
    (url, { arg }: { arg: { content: string } }) =>
      fetch(url, { method:'POST', headers:{'content-type':'application/json'}, body: JSON.stringify(arg) })
  );

  if (isLoading) return <p>加载中…</p>;
  if (error) return <p>出错：{String(error)}</p>;

  const add = async () => {
    // 本地乐观更新
    await mutate(`/api/posts/${id}`, (p: any) => ({ ...p, comments: [...p.comments, { id:'tmp', content:'Hi', optimistic:true }] }), false);
    await trigger({ content: 'Hi' });
    await mutate(`/api/posts/${id}`); // 重新校验
  };

  return <>
    <h1>{data.title}</h1>
    <button disabled={isMutating} onClick={add}>评论</button>
  </>;
}
```

```tsx
// 无限加载
import useSWRInfinite from 'swr/infinite';
const getKey = (index: number, prev: any) => {
  if (prev && !prev.next) return null;  // 没有下一页
  return `/api/feed?page=${index + 1}`;
};
const { data, size, setSize, isValidating } = useSWRInfinite(getKey);
```

---

## 4) Router × Query/SWR —— 最佳搭配

### 4.1 “Loader 预取 + Query 消费”模式（推荐）
> 页面切换时由 Router 统一控制数据“首屏必需项”，并把结果**喂进 Query 缓存**，页面内部继续用 Query 读写、失效、乐观更新。

```tsx
// main.tsx
import { createBrowserRouter } from 'react-router-dom';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from './query/client';
import { postLoaderWithQuery } from './routes/post-q-loader';

const router = createBrowserRouter([
  {
    path: '/posts/:id',
    loader: (args) => postLoaderWithQuery(queryClient, args),
    element: <PostWithQuery />
  }
]);

export const App = () => (
  <QueryClientProvider client={queryClient}>
    <RouterProvider router={router} />
  </QueryClientProvider>
);
```

```ts
// routes/post-q-loader.ts
import type { LoaderFunctionArgs } from 'react-router-dom';
import type { QueryClient } from '@tanstack/react-query';
import { postQuery } from '../query/posts';

export async function postLoaderWithQuery(qc: QueryClient, { params }: LoaderFunctionArgs) {
  const id = params.id!;
  // 确保缓存中有数据；若已有且未过期不会重复拉
  await qc.ensureQueryData(postQuery(id));
  return null; // 组件直接 useQuery 读取，避免重复请求
}
```

> 好处：**路由切换 = 数据就绪**；数据的**缓存/失效**仍由 Query 掌控。

### 4.2 `action` × Query 失效/乐观
- 写操作走 `action`（或组件内 `useMutation`），**成功后**统一 `invalidateQueries`。  
- 多处页面依赖同一 Query 时，**一次失效，处处更新**。

### 4.3 何时直接用 `loader` 返回 JSON？
- **只被该路由消费**、**无需跨页面共享**、**不需要乐观更新/复杂失效** → 直接 `useLoaderData()` 最简单。  
- 需要共享或后续更新 → 用 **4.1** 的“预取+Query”模式。

---

## 5) 并发、取消与错误边界（稳）

- **取消**：在 `loader/action` 里始终把 `request.signal` 传给 `fetch`。导航中断→请求自动 abort。  
- **并发**：在 Query 用 `Promise.all` 或并行多个 `useQuery`；在 Router 可用 `defer` 分流慢资源。  
- **错误边界**：每个路由配 `errorElement`，或在 `createRoutesFromElements` 里集中定义。  
- **Pending UI**：Router 用 `useNavigation()`；Query/SWR 用 `isPending/isLoading/isValidating`。

---

## 6) SSR / 预渲染（提要）

- **React Router + Vite SSR**：在服务端跑 `router.fetch()`（或命中你自建的 loader 执行器）收集数据 → 水合。  
- **React Query**：服务端 `dehydrate(queryClient)` 注入，客户端 `Hydrate` 接棒；与 4.1 模式天然契合。  
- **SWR**：通过 `fallback` 注入首屏数据；或静态化 Story/页面时把数据内联为 `window.__DATA__` 后挂到 `SWRConfig`。

> 选择哪条路取决于你的框架脚手架；若是 Next/Remix 有各自内建方案，此处不展开。

---

## 7) 性能与体验清单

- **预取**：鼠标悬停 `prefetchQuery`；移动端可在视口临界点预取下一页。  
- **保持旧数据**：Query 的 `placeholderData: keepPreviousData` 或 `keepPreviousData: true`（v4→v5参数名变化，按所用版本调整），翻页不卡顿。  
- **选择器**：`select` 只取需要的字段，减少重渲染。  
- **缓存策略**：`staleTime` 对稳定资源放大（如字典/元数据），`gcTime` 根据内存和页面规模权衡。  
- **去抖与合并**：搜索/筛选请求用 `useDeferredValue` 或手写防抖；服务端合并批量请求。  
- **禁用焦点重拉**：表单编辑中可临时关闭 `refetchOnWindowFocus`，避免打断输入。

---

## 8) 反模式与纠偏

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 页面必需数据放组件里 `useEffect` 拉 | 首屏闪烁、竞争条件 | 放到 `loader` 或“预取+Query” |
| 写操作没有统一失效策略 | A 页更新，B 页不刷新 | 设计稳定的 Query Key 与失效表 |
| 不传 `signal` | 导航后仍在打无用请求 | 在 `loader/action` 的 `fetch` 传 `request.signal` |
| 所有数据都塞 Query | 重复缓存、心智负担重 | 路由专属数据就用 `loader` 即可 |
| 滥用全局错误边界 | 错误定位困难 | 路由级 `errorElement` + 组件局部兜底 |

---

## 9) 提交前检查清单 ✅

- [ ] 路由必需数据通过 **loader / defer** 就位。  
- [ ] 写操作走 **action** 或 **useMutation**；成功后统一 **invalidate**。  
- [ ] 存在跨页面共享或复杂更新 → 采用 **“loader 预取 + Query/SWR 消费”**。  
- [ ] 所有 `fetch` 都有 **超时/取消（signal）**；错误经由 **路由级边界** 呈现。  
- [ ] Query Key 设计稳定、具层次；分页/筛选参数纳入 Key。  
- [ ] 性能策略明确：`staleTime/gcTime`、预取、去抖、保持旧数据。

---

## 10) 练习 🏋️

1. 把“文章详情”路由改造成：`loader` 预取 + Query 消费 + 评论 `mutation` 乐观更新。  
2. 给“用户中心”加 `defer`：基础档案先到、资产统计后到；为后到数据加 `<Await>` 和错误兜底。  
3. 实现一个“无限滚动列表”：Query `useInfiniteQuery` / SWR `useSWRInfinite` 二选一，并加预取下一页与窗口焦点重校验策略。

---
