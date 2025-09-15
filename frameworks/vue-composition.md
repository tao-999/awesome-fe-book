# 6.1 组合式 API 与 Pinia 🍣

本章目标：把 **Vue 3 组合式 API（Composition API）** 与 **Pinia** 玩成“逻辑像乐高、状态像小店铺”。你将拿到：`ref/reactive/computed/watch` 的实战套路、`<script setup>` 宏的正确打开方式、可复用 **composable** 模式、**Pinia** 的建模/持久化/SSR 策略、以及反模式与检查清单。

---

## 0) 心智图与选型 🧭

- **组件局部状态/业务逻辑** → 组合式 API（用 composables 抽取并复用）。
- **客户端全局状态**（登录态、跨路由上下文、偏好等） → **Pinia**。
- **服务器状态**（来自接口、可缓存/失效） → 交给 **TanStack Query / SWR for Vue**（见 5.1），不要塞进 Pinia。

> 宗旨：**先局部，后全局；先派生，后存储。**

---

## 1) 组合式 API 基础 🍱

### 1.1 `ref` / `reactive` / `computed`

```ts
import { ref, reactive, computed } from 'vue';

const count = ref(0);                // 基元/单值首选 ref
const user  = reactive({ id: 'u1', profile: { name: 'Ada' } });

const double = computed(() => count.value * 2);

// 修改
count.value++;
user.profile.name = 'Lovelace';
```

> 经验：**对象树**用 `reactive`，**基元或可能替换整个对象**用 `ref`（例如把 `ref({})` 直接换成新对象）。

### 1.2 解构失活 & `toRef(s)`/`toRefs`
```ts
import { toRef, toRefs } from 'vue';

const state = reactive({ a: 1, b: 2 });
// ❌ 直接解构会失活：
const { a } = state; // a 是普通值
// ✅ 用 toRefs：
const { a: aRef, b } = toRefs(state);
aRef.value++;
```

### 1.3 `watch` / `watchEffect`
```ts
import { watch, watchEffect } from 'vue';

watch(() => route.query.q, (q) => search(q), { immediate: true });
// 副作用自动收集依赖、回收：
watchEffect((onCleanup) => {
  const id = startPolling();
  onCleanup(() => stopPolling(id));
}, { flush: 'post' }); // UI 更新后再执行，避免“抖 UI”
```

### 1.4 浅/只读与大对象优化
```ts
import { shallowRef, markRaw, readonly } from 'vue';
const chart = shallowRef<any>();              // 大型第三方实例
chart.value = markRaw(new ECharts(...));      // 跳过深层代理
const settings = readonly(reactive({ theme:'dark' })); // 只读视图
```

### 1.5 生命周期 & 作用域
```ts
import { onMounted, onUnmounted, effectScope, onScopeDispose } from 'vue';

onMounted(() => {/* ... */});
onUnmounted(() => {/* ... */});

const scope = effectScope();
scope.run(() => {
  const id = startSomething();
  onScopeDispose(() => stopSomething(id)); // 作用域销毁时清理
});
// 需要时：scope.stop()
```

---

## 2) `<script setup>` 宏与 TypeScript ⚙️

```vue
<script setup lang="ts">
/** 1) 定义 Props/Emits（全推断） */
const props = defineProps<{
  modelValue?: string
  size?: 'sm' | 'md' | 'lg'
}>();
const emit = defineEmits<{
  'update:modelValue': [string]
  submit: [{ value: string }]
}>();

/** 2) v-model 简写（Vue 3.4+） */
const model = defineModel<string>({ default: '' }); // 等价 modelValue + update 语法糖

/** 3) 暴露实例 API（父组件可拿到） */
function focus() { /* ... */ }
defineExpose({ focus });

/** 4) 插槽类型（可选） */
defineSlots<{ default(props: { active: boolean }): any }>();
</script>
```

> 贴士：搭配 Volar（或 Vue Language Tools）享受“宏级别”的类型推断；`<script setup>` 中的顶级变量直接暴露给模板，无需 `return`。

---

## 3) Composable 模式（可复用逻辑） 🧩

### 3.1 事件监听 composable
```ts
// composables/useEventListener.ts
import { onMounted, onUnmounted } from 'vue';

export function useEventListener<T extends keyof WindowEventMap>(
  target: Window | Document | HTMLElement,
  type: T,
  handler: (ev: WindowEventMap[T]) => void,
  opts?: boolean | AddEventListenerOptions
){
  onMounted(() => target.addEventListener(type, handler as any, opts));
  onUnmounted(() => target.removeEventListener(type, handler as any, opts));
}
```

### 3.2 可取消的 fetch（Abort + 竞态防抖）
```ts
// composables/useFetch.ts
import { ref, shallowRef, watchEffect } from 'vue';

export function useFetch<T>(url: () => string | null) {
  const data = shallowRef<T | null>(null);
  const error = ref<Error | null>(null);
  const loading = ref(false);

  watchEffect((onCleanup) => {
    const u = url(); if (!u) return;
    const ac = new AbortController(); loading.value = true;

    fetch(u, { signal: ac.signal })
      .then(r => { if(!r.ok) throw new Error(String(r.status)); return r.json(); })
      .then(json => { data.value = json; error.value = null; })
      .catch(e => { if(e.name !== 'AbortError') error.value = e; })
      .finally(() => { loading.value = false; });

    onCleanup(() => ac.abort());
  });

  return { data, error, loading };
}
```

### 3.3 提供/注入（DI，带类型）
```ts
// di/tokens.ts
import type { InjectionKey } from 'vue';
export type Auth = { userId: string | null; signIn(): Promise<void> };
export const AuthKey: InjectionKey<Auth> = Symbol('Auth');

// provider
provide(AuthKey, { userId: null, async signIn(){/*...*/} });

// consumer
const auth = inject(AuthKey);
if (!auth) throw new Error('缺少 Auth provider');
```

---

## 4) 与路由协作 🗺️

```ts
import { useRoute, useRouter } from 'vue-router';
import { watch } from 'vue';

const route = useRoute(), router = useRouter();
// 监听查询参数
watch(() => route.query.q, (q) => doSearch(String(q ?? '')), { immediate: true });

// 导航守卫（组件内）
onBeforeRouteLeave((to, from, next) => {
  if (dirty.value) return next(confirm('离开会丢失更改，继续？'));
  next();
});
```

---

## 5) Pinia：把全局状态变“小店铺” 🏪

### 5.1 安装与挂载
```ts
// main.ts
import { createApp } from 'vue';
import { createPinia } from 'pinia';
import App from './App.vue';

const app = createApp(App);
app.use(createPinia());
app.mount('#app');
```

### 5.2 两种写法：Options Store vs Setup Store

**Options 风格（更像 Vuex，但简洁）**
```ts
// stores/auth.ts
import { defineStore } from 'pinia';

export const useAuth = defineStore('auth', {
  state: () => ({ user: null as null | { id: string; name: string }, token: null as null | string }),
  getters: {
    isSignedIn: (s) => !!s.token
  },
  actions: {
    async signIn(username: string, password: string) {
      const res = await fetch('/api/login', { method: 'POST', body: JSON.stringify({ username, password }) });
      const { token, user } = await res.json();
      this.token = token; this.user = user;
    },
    signOut() { this.$reset(); }
  }
});
```

**Setup 风格（组合式 + 完整 TS 推断）**
```ts
// stores/cart.ts
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';

export const useCart = defineStore('cart', () => {
  const items = ref<{ id: string; name: string; price: number }[]>([]);
  const total = computed(() => items.value.reduce((s, i) => s + i.price, 0));
  function add(it: { id: string; name: string; price: number }) { items.value.push(it); }
  function remove(id: string) { items.value = items.value.filter(i => i.id !== id); }
  return { items, total, add, remove };
});
```

**在组件里使用（`storeToRefs` 防解构失活）：**
```ts
import { storeToRefs } from 'pinia';
const auth = useAuth();
const { isSignedIn, user } = storeToRefs(auth);
const { signIn, signOut } = auth;
```

### 5.3 批量变更与 Patch
```ts
// 覆盖式 patch
auth.$patch({ token: null, user: null });
// 函数式 patch（可读性佳）
auth.$patch((s) => { s.user = { id: 'u2', name: 'Bob' }; });
```

### 5.4 订阅与持久化（轻量实现）
```ts
// 本地持久化（白名单）
const auth = useAuth();
auth.$subscribe((_mutation, state) => {
  localStorage.setItem('auth', JSON.stringify({ token: state.token, user: state.user }));
}, { detached: true }); // 不随组件销毁而取消

// 初始化时 hydration
const raw = localStorage.getItem('auth');
if (raw) auth.$patch(JSON.parse(raw));
```

> 正式项目建议封装为 **Pinia 插件**（统一白名单/版本迁移），或使用成熟的 `pinia-plugin-persistedstate`。

### 5.5 插件（扩展 store 能力）
```ts
// plugins/logger.ts
import type { PiniaPluginContext } from 'pinia';
export function logger() {
  return (ctx: PiniaPluginContext) => {
    ctx.store.$subscribe((m, s) => {
      console.debug(`[${ctx.store.$id}]`, m.type, m.events ?? m.payload, structuredClone(s));
    });
  };
}

// main.ts
const pinia = createPinia();
pinia.use(logger());
app.use(pinia);
```

### 5.6 SSR 指南（提要）
- **每个请求创建新的 pinia 实例**，避免跨请求污染。  
- 服务端：把关键 store 状态序列化注入 HTML（`pinia.state.value`）。  
- 客户端：`pinia.state.value = window.__PINIA__` 后再 `app.use(pinia)` 挂载。  
- 不要在 store 顶层直接触发副作用；把 I/O 放在 **action** 并在路由钩子或页面生命周期触发。

---

## 6) 表单与双向绑定（3.4 `defineModel`） ✍️

```vue
<!-- InputText.vue -->
<script setup lang="ts">
const model = defineModel<string>({ default: '' }); // v-model
</script>
<template>
  <input v-model="model" class="input" />
</template>

<!-- 使用 -->
<InputText v-model="form.title" />
```

> 复杂表单建议用 **受控 + 校验库**（如 zod + 自写 composable 或 vee-validate），把“表单状态”留在组件/页面，不入 Pinia。

---

## 7) 性能与稳定性 🍀

- **只动 `ref/computed`**：大对象用 `shallowRef/markRaw` 避免深度代理成本。  
- **`watch` 精准依赖**：不要给整个 `reactive` 加 `deep:true`，首选 **函数 getter** 侦听。  
- **渲染节流**：高频输入用 `useDebouncedRef` 或 `watch` + `debounce`。  
- **Pinia 拆分**：按领域建多个 store，小而清晰；避免“一个 store 管天下”。  
- **测试友好**：store 不直接操作 DOM；I/O 走 actions，可在 Vitest 里打桩。

---

## 8) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 随手解构 `reactive` | 响应性丢失 | 用 `toRefs` / `storeToRefs` |
| 所有数据都塞 Pinia | 重复真相、难失效 | 服务器数据交给 Query；Pinia 只放客户端状态 |
| `watch` deep 满天飞 | 性能抖动 | 精准 getter；必要时 `shallowRef` |
| 在顶层 new 第三方实例 | SSR 污染/内存泄漏 | 用 `shallowRef` + 生命周期创建/销毁 |
| 大型 store | 任意修改引发全场重渲 | 拆分 store + 选择性订阅 |
| 在 action 里吞错 | 静默失败 | 明确返回 `Result` 或抛出，交由调用方处理 |

---

## 9) 提交前检查清单 ✅

- [ ] 组件逻辑抽成 composables，复用点清晰、含清理逻辑。  
- [ ] `<script setup>` 使用 `defineProps/defineEmits/defineModel/defineExpose`，类型齐全。  
- [ ] Pinia 仅存**客户端全局状态**；服务器数据不复制进 store。  
- [ ] 对第三方实例使用 `shallowRef + markRaw`；对大监听避免 `deep`。  
- [ ] 本地持久化走插件/白名单；带版本迁移。  
- [ ] SSR：每请求独立 pinia；水合前注入初始状态。  
- [ ] 单测覆盖关键 composables 与 store actions（Vitest + Vue Test Utils）。

---

## 10) 练习 🏋️

1. 写一个 `useDisclosure` composable（开/关 + 键盘 Esc 关闭 + `onScopeDispose` 清理），并在两个组件里复用。  
2. 建一个 `useAuth` Pinia store（`signIn/signOut` + 持久化 token），并在路由守卫里控制访问。  
3. 把页面的“搜索建议”用 `useFetch` + `AbortController` 重写，输入去抖 250ms，切路由自动取消。  
4. 为一个大型可视化组件用 `shallowRef + markRaw` 管实例，写 `onMounted/onUnmounted` 生命周期管理。

---

## 11) 速用片段（收藏夹） 🗂️

**A. Debounced Ref**
```ts
import { customRef } from 'vue';
export function useDebouncedRef<T>(value: T, delay = 200) {
  let t: number | undefined;
  return customRef<T>((track, trigger) => ({
    get() { track(); return value; },
    set(v: T) { clearTimeout(t); t = window.setTimeout(() => { value = v; trigger(); }, delay); }
  }));
}
```

**B. Typed Injection Helper**
```ts
import type { InjectionKey } from 'vue';
export const createKey = <T>(desc: string) => Symbol(desc) as InjectionKey<T>;
```

**C. Pinia 持久化插件（极简）**
```ts
// pinia-plugin-persist.ts
import type { PiniaPluginContext } from 'pinia';
export function persistPlugin(keys: Record<string, (s:any)=>any>) {
  return (ctx: PiniaPluginContext) => {
    const pick = keys[ctx.store.$id];
    if (pick) {
      // hydrate
      const raw = localStorage.getItem(`pinia:${ctx.store.$id}`);
      if (raw) ctx.store.$patch(JSON.parse(raw));
      // subscribe
      ctx.store.$subscribe((_m, state) => {
        localStorage.setItem(`pinia:${ctx.store.$id}`, JSON.stringify(pick(state)));
      }, { detached: true });
    }
  };
}
// 使用
const pinia = createPinia();
pinia.use(persistPlugin({
  auth: (s) => ({ token: s.token, user: s.user })
}));
```

---

**小结**：用 **组合式 API** 把逻辑做成“可插拔的原语”，用 **Pinia** 把全局状态做成“小而美的店铺”。服务器数据交给数据层库，三者分工明确，你的 Vue 应用会既快又稳还好测。🧪⚡
