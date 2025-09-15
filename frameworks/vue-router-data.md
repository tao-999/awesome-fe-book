# 6.2 路由与数据层 🧭

本章目标：用 **Vue Router 4** + **数据层（@tanstack/vue-query 或 SWRV）** 搭出“**页面 = 路由 + 数据**”的稳定范式：  
- 路由负责**导航、作用域、懒加载、守卫与滚动行为**；  
- 数据层负责**缓存、并发、失效、乐观更新与预取**；  
- 二者用 **预取（prefetch）** 与 **Suspense** 顺滑衔接；支持 **Abort 取消**、**SSR 脱水/水合** 与 **KeepAlive**。

---

## 0) 选型心法

- **路由**：`vue-router@4`（history/嵌套路由/动态段/守卫/滚动行为/懒加载）。  
- **数据层**：  
  - **@tanstack/vue-query**（强缓存/失效/乐观更新/预取/无限列表/SSR 支持）——推荐。  
  - **SWRV（swrv）**（SWR 哲学，轻量“取—改—再取”）。  
- **客户端全局状态**：见 6.1（Pinia）。**不要**把“服务器状态”塞 Pinia。

---

## 1) Vue Router 4 基线

### 1.1 安装与骨架
```ts
// router.ts
import { createRouter, createWebHistory, type RouteRecordRaw } from 'vue-router';

const routes: RouteRecordRaw[] = [
  { path: '/', name: 'home', component: () => import('./pages/Home.vue') },
  {
    path: '/posts/:id',
    name: 'post',
    props: true, // 将 params 传为组件 props
    component: () => import('./pages/Post.vue'),
    meta: { title: '文章详情' }
  },
  { path: '/:pathMatch(.*)*', name: '404', component: () => import('./pages/NotFound.vue') }
];

export const router = createRouter({
  history: createWebHistory(),
  routes,
  scrollBehavior(to, from, saved) {
    if (saved) return saved;            // 返回历史位置
    return { top: 0 };                  // 默认回到顶部
  }
});
```

```ts
// main.ts
import { createApp, h } from 'vue';
import { router } from './router';
import { VueQueryPlugin, QueryClient, dehydrate, hydrate } from '@tanstack/vue-query';

const app = createApp({ render: () => h(App) });
app.use(router);
app.use(VueQueryPlugin, { queryClient: new QueryClient() });
app.mount('#app');
```

### 1.2 守卫与页面标题
```ts
router.beforeEach((to) => {
  document.title = (to.meta.title as string) ?? 'App';
});
```

---

## 2) 数据层：@tanstack/vue-query（推荐）

### 2.1 安装与全局配置
```bash
pnpm add @tanstack/vue-query
```

```ts
// query/client.ts
import { QueryClient } from '@tanstack/vue-query';
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 60_000, gcTime: 5 * 60_000, retry: 2, refetchOnWindowFocus: false }
  }
});
```

### 2.2 查询与变更（最小用法）
```ts
// query/posts.ts
const fetchJSON = (u: string, init?: RequestInit) =>
  fetch(u, init).then(r => { if (!r.ok) throw new Error('HTTP ' + r.status); return r.json(); });

export const postQuery = (id: string) => ({
  queryKey: ['post', id],
  queryFn: () => fetchJSON(`/api/posts/${id}`)
});
```

```vue
<!-- pages/Post.vue -->
<script setup lang="ts">
import { useRoute } from 'vue-router';
import { useQuery, useMutation, useQueryClient } from '@tanstack/vue-query';
import { postQuery } from '../query/posts';

const route = useRoute();
const id = route.params.id as string;
const qc = useQueryClient();

const { data, isPending, error } = useQuery(postQuery(id));

const addComment = useMutation({
  mutationFn: (content: string) =>
    fetch(`/api/posts/${id}/comments`, {
      method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ content })
    }).then(r => { if (!r.ok) throw new Error('提交失败'); }),
  onMutate: async (content) => {
    await qc.cancelQueries({ queryKey: ['post', id] });
    const prev = qc.getQueryData<any>(['post', id]);
    qc.setQueryData(['post', id], (p: any) => ({ ...p, comments: [...(p?.comments ?? []), { id: 'tmp', content, optimistic: true }] }));
    return { prev };
  },
  onError: (_e, _v, ctx) => ctx?.prev && qc.setQueryData(['post', id], ctx.prev),
  onSettled: () => qc.invalidateQueries({ queryKey: ['post', id] })
});
</script>

<template>
  <section v-if="isPending">读取中…</section>
  <section v-else-if="error">出错：{{ String(error) }}</section>
  <section v-else>
    <h1>{{ data.title }}</h1>
    <ul><li v-for="c in data.comments" :key="c.id">{{ c.content }}</li></ul>
    <button @click="addComment.mutate('Hello!')">乐观评论</button>
  </section>
</template>
```

---

## 3) 路由 × 数据：**预取（prefetch）+ Suspense** 模式

> 目标：**路由切换 = 数据就绪**。在进入页面前预取关键数据，组件内继续由 Query 管缓存/失效。

### 3.1 在路由 `beforeResolve` 里预取
```ts
// prefetch.ts
import type { RouteLocationNormalized } from 'vue-router';
import type { QueryClient } from '@tanstack/vue-query';
import { postQuery } from './query/posts';

export async function prefetchRoute(qc: QueryClient, to: RouteLocationNormalized) {
  if (to.name === 'post') {
    const id = String(to.params.id);
    await qc.ensureQueryData(postQuery(id)); // 已缓存则跳过
  }
}
```

```ts
// main.ts（挂接预取）
import { queryClient } from './query/client';
router.beforeResolve(async (to) => {
  // 仅首要数据预取；次要数据在组件内懒取或用 <Suspense>
  await prefetchRoute(queryClient, to);
});
```

### 3.2 组件内用 `<Suspense>` 协调多资源
```vue
<!-- App.vue -->
<template>
  <RouterView v-slot="{ Component }">
    <Suspense>
      <component :is="Component" />
      <template #fallback><p>加载页面中…</p></template>
    </Suspense>
  </RouterView>
</template>
```

---

## 4) 取消与竞态：AbortController + 作用域清理

```ts
// composables/useAbortableFetch.ts
import { ref, watchEffect, onScopeDispose } from 'vue';

export function useAbortableFetch<T>(url: () => string | null) {
  const data = ref<T | null>(null), error = ref<Error | null>(null), loading = ref(false);
  watchEffect((onCleanup) => {
    const u = url(); if (!u) return;
    const ac = new AbortController(); loading.value = true;

    fetch(u, { signal: ac.signal })
      .then(r => { if (!r.ok) throw new Error('HTTP ' + r.status); return r.json(); })
      .then(j => (data.value = j, error.value = null))
      .catch(e => { if (e.name !== 'AbortError') error.value = e; })
      .finally(() => (loading.value = false));

    onCleanup(() => ac.abort());
  });
  onScopeDispose(() => {/* 其它清理 */});
  return { data, error, loading };
}
```

> 在 **路由切换** 或 **组件卸载** 时，副作用被销毁并触发 `abort`；Query 内部也会在组件卸载时停止订阅。

---

## 5) 无限列表与预取（首屏丝滑）

```ts
// query/feed.ts
import { useInfiniteQuery } from '@tanstack/vue-query';

export const useFeed = () => useInfiniteQuery({
  queryKey: ['feed'],
  queryFn: ({ pageParam = 1 }) => fetch(`/api/feed?page=${pageParam}`).then(r => r.json()),
  getNextPageParam: (last) => last.next ?? undefined,
  staleTime: 30_000,
  placeholderData: (prev) => prev // 保留旧数据，翻页不抖
});
```

```vue
<!-- pages/Feed.vue -->
<script setup lang="ts">
import { onMounted, ref } from 'vue';
import { useFeed } from '../query/feed';

const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useFeed();
const sentry = ref<HTMLElement | null>(null);

onMounted(() => {
  const io = new IntersectionObserver(e => e[0].isIntersecting && hasNextPage.value && fetchNextPage());
  if (sentry.value) io.observe(sentry.value);
});
</script>

<template>
  <ul>
    <li v-for="p in data?.pages.flatMap(p => p.items)" :key="p.id">{{ p.title }}</li>
  </ul>
  <div ref="sentry">{{ isFetchingNextPage ? '加载中…' : '加载更多' }}</div>
</template>
```

---

## 6) SWRV（SWR 风格）对照（可选）

```bash
pnpm add swrv
```

```vue
<script setup lang="ts">
import useSWRV, { mutate } from 'swrv';
const { data, error, isValidating } = useSWRV('/api/posts/42', (u) => fetch(u).then(r => r.json()));

async function add() {
  await mutate('/api/posts/42', (p: any) => ({ ...p, comments: [...p.comments, { id:'tmp', content:'Hi' }] }), false);
  await fetch('/api/posts/42/comments', { method: 'POST' });
  await mutate('/api/posts/42'); // 重新校验
}
</script>
```

> SWRV 轻量，但在失效、并发控制、无限列表、SSR 脱水等方面不如 vue-query 完整。

---

## 7) KeepAlive 与缓存协作

```vue
<!-- 保持路由组件实例（表单草稿/滚动位置） -->
<KeepAlive include="post,feed">
  <RouterView />
</KeepAlive>
```

- **KeepAlive** 保存组件局部状态，但**不等于**服务器数据缓存；服务器数据依旧由 Query 管。  
- 如配合分页，使用 Query 的 `keepPreviousData`/`placeholderData` 减少跳动。

---

## 8) SSR / 预渲染（提要）

- **vue-query**：服务端预取 `await queryClient.prefetchQuery(...)` → `dehydrate(queryClient)` 注入 HTML；客户端 `hydrate(queryClient, window.__STATE__)`。  
- **组件级异步**：`<Suspense>`/`async setup()` 在 SSR 阶段会等待；与路由预取配合可实现“首屏直出”。  
- **Nuxt** 有更完整方案（若迁移 Nuxt，复用本章数据层策略即可）。

---

## 9) 错误与加载 UI

- 路由级：在布局里统一放 **错误占位/空状态** 区域；组件内部使用 `isPending/error`。  
- 可以封装一个 **ErrorBoundary** 组件，利用 `onErrorCaptured` 捕获后渲染兜底。  
- 表单提交：组件中用 `useMutation`，或 `@submit.prevent` 调用 API；成功后 `invalidateQueries`。

---

## 10) 性能与体验清单 🏎️

- **预取**：`router.beforeResolve` + `queryClient.ensureQueryData`；移动端也可在视口临界点预取下一页。  
- **缓存策略**：稳定资源放大 `staleTime`；短期变动缩短；谨慎 `retry`。  
- **只取所需**：在选择器/投影层裁剪数据，减少重渲染。  
- **禁用焦点重拉**：编辑表单时 `refetchOnWindowFocus=false`，避免打断输入。  
- **统一 `fetcher`**：封装 `fetchJSON`（超时/基础 headers/错误映射），减少重复样板。  
- **避免重复真相**：不要把 API 返回对象复制进 Pinia；只存**客户端状态**或**引用/选择**。

---

## 11) 反模式与纠偏

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 组件 `onMounted` 自己拉首屏数据 | 首屏闪烁、竞态 | 路由预取 + 组件用 Query 消费 |
| 不传 `signal` 取消 | 路由切走仍在打请求 | 在自写 fetch/Composable 中加 `AbortController` |
| 把服务器数据塞 Pinia | 双份真相 | 用 Query/SWR 管服务器数据；Pinia 仅放客户端状态 |
| 所有页面都 KeepAlive | 内存上涨、数据陈旧 | 仅对需要保活的页面启用；服务器数据照常失效 |
| 缺乏失效策略 | A 页改了 B 页不动 | 统一设计 Query Key 与 `invalidateQueries` |

---

## 12) 提交前检查清单 ✅

- [ ] 路由切换前预取首要数据；组件内用 Query/SWR 消费与更新。  
- [ ] 所有 I/O 可取消（`AbortController`），副作用在作用域销毁时清理。  
- [ ] 设计稳定的 Query Key；写操作后统一失效或乐观回滚。  
- [ ] SSR（如需）：服务端脱水、客户端水合；`<Suspense>` 协调异步。  
- [ ] KeepAlive 仅用于需要保留的页面；滚动/表单状态可配合。  
- [ ] 错误/空/加载态有一致的可视化与无障碍表现。

---

## 13) 练习 🏋️

1. 给「文章详情」路由加 **beforeResolve 预取**，组件用 `useQuery` 读取；为“发表评论”写一个 `useMutation`（乐观更新 + 失败回滚）。  
2. 为「用户中心」拆成两段数据：档案（快）与统计（慢）。档案在路由预取，统计在组件中配 `<Suspense>` 与错误兜底。  
3. 实现一个无限滚动 Feed：IntersectionObserver 触底 → `useInfiniteQuery` 取下一页；返回上一页保持旧数据不抖动。  

---

**小结**：把“**导航**”交给 **Vue Router**，“**服务器数据**”交给 **vue-query/SWRV**。用 **预取 + Suspense** 抹平切换，用 **失效/乐观**管理更新，用 **Abort/作用域**清理副作用。职责分明，应用就会又稳又快。🚀
