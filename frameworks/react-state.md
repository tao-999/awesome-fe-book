# 5.2 状态管理：Redux / Zustand / Recoil / Jotai 🧠

本章目标：把前端“状态丛林”修剪成**清晰边界**：**UI/客户端状态**（由本章工具管理）与**服务器状态**（由 React Query/SWR 管理）。给出四类库的**心智模型、最佳实践、性能坑位与迁移策略**，配 TypeScript 可拷贝样例。

---

## 0) 先划边界：什么状态该放哪？

- **服务器状态**（可缓存、可失效、来自 API）：交给 **React Query / SWR**（见 5.1）；不要塞进任何全局状态库。  
- **客户端状态**（本地 UI、表单向导步骤、临时草稿、偏好设置、跨路由的业务上下文）：用 Redux / Zustand / Recoil / Jotai。  
- **派生状态**（可以由现有状态计算出）：尽量**计算得到**，不要重复存。  
- **组件局部、一次性状态**：`useState/useReducer` 就够了。

> 经验法则：**先局部，后全局；先派生，后存储。**

---

## 1) 选型一览（一句话判断）

| 场景 | 推荐 |
|---|---|
| 中大型团队、强规范、可预测流/审计、时光旅行 | **Redux（Redux Toolkit）** |
| 轻量、心智简单、店铺式 API、最少样板 | **Zustand** |
| 需要 **原子化依赖跟踪**、跨组件细粒度订阅、跨树派生 | **Recoil / Jotai** |
| 需要 “服务端状态 + 客户端状态” 并举 | **React Query +（Zustand/Redux/原子库）** |

---

## 2) Redux（配 Redux Toolkit, RTK）🟥

### 2.1 何时用
- 团队大、多人协作、需要时光旅行/审计、复杂写入流程（中间件）、可序列化日志。
- 与 **Redux DevTools** 生态配合良好；**RTK Query** 可做轻量数据拉取（仍建议复杂数据交给 React Query）。

### 2.2 最小可用（RTK Slice + Typed Hooks）
```ts
// store.ts
import { configureStore, createSlice, PayloadAction } from '@reduxjs/toolkit';

type AuthState = { user?: { id: string; name: string } | null; token?: string | null };
const auth = createSlice({
  name: 'auth',
  initialState: { user: null, token: null } as AuthState,
  reducers: {
    signedIn: (s, a: PayloadAction<{ user: AuthState['user']; token: string }>) => {
      s.user = a.payload.user; s.token = a.payload.token;
    },
    signedOut: (s) => { s.user = null; s.token = null; }
  }
});

export const { signedIn, signedOut } = auth.actions;

export const store = configureStore({ reducer: { auth: auth.reducer } });
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

```ts
// hooks.ts
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './store';
export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

```tsx
// use cases
const name = useAppSelector(s => s.auth.user?.name);  // 只订阅需要的切片
const dispatch = useAppDispatch();
dispatch(signedIn({ user: { id: 'u1', name: 'Ada' }, token: 'jwt' }));
```

### 2.3 中间件与持久化
```ts
// persist（可选）
import storage from 'redux-persist/lib/storage';
import { persistStore, persistReducer } from 'redux-persist';

const persistConfig = { key: 'root', storage, whitelist: ['auth'] };
export const store = configureStore({
  reducer: persistReducer(persistConfig, { auth: auth.reducer }),
  middleware: (gDM) => gDM({ serializableCheck: false })
});
export const persistor = persistStore(store);
```

### 2.4 最佳实践
- **使用 RTK**（不可变 + Immer + Action Creator + Slice），避免手写样板。
- **只存必需字段**，列表大对象在组件内做选择/分页；派生用 `reselect`。  
- 与 React Query 集成：**写操作成功后**触发 `queryClient.invalidateQueries()`，而非把远程数据复制到 Redux。

---

## 3) Zustand 🟧

### 3.1 何时用
- 轻量、无需模板代码；天然支持**选择器订阅**（只重渲染用到的字段）；更像“**小店铺**”而非大仓库。

### 3.2 最小可用（TS + 选择器）
```ts
// useCart.ts
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

type Item = { id: string; name: string; price: number };
type CartState = {
  items: Item[];
  add: (it: Item) => void;
  remove: (id: string) => void;
  total: () => number;
};

export const useCart = create<CartState>()(
  devtools(persist((set, get) => ({
    items: [],
    add: (it) => set(state => ({ items: [...state.items, it] })),
    remove: (id) => set(state => ({ items: state.items.filter(i => i.id !== id) })),
    total: () => get().items.reduce((s, i) => s + i.price, 0)
  }), { name: 'cart' }))
);
```

```tsx
// 选择器订阅（只订一小块）
const items = useCart(s => s.items);
const total = useCart(s => s.total());
const add = useCart(s => s.add);
```

### 3.3 提示
- 使用 **选择器函数**（`useCart(s => s.field)`）+ **浅比较 `shallow`** 提升性能。  
- **分片 store**：多个 `create()` 拆分领域（auth/cart/ui），避免巨无霸。  
- 需要复杂 flow/日志？Zustand 可以加 `devtools`，但**不等于** Redux 的严格可序列化与时光旅行。

---

## 4) Recoil 🟦

### 4.1 何时用
- 需要**原子化依赖跟踪**，跨树**细粒度更新**；选择/派生关系复杂；像把“全局状态”当“很多小 `useState`”。

### 4.2 概念
- **Atom**：最小可订阅单元。  
- **Selector**：从 atom/selector 派生，依赖图自动跟踪与缓存（可异步）。  
- **Family**：参数化的 atom/selector 工厂。

```ts
// state.ts
import { atom, selector, atomFamily, selectorFamily } from 'recoil';

export const cartItems = atom<{ id: string; name:string; price:number }[]>({
  key: 'cartItems',
  default: []
});

export const cartTotal = selector<number>({
  key: 'cartTotal',
  get: ({ get }) => get(cartItems).reduce((s, i) => s + i.price, 0)
});

export const itemAtom = atomFamily<{ id:string; count:number }, string>({
  key: 'item',
  default: { id: '', count: 0 }
});

export const itemCountSel = selectorFamily<number, string>({
  key: 'itemCountSel',
  get: (id) => ({ get }) => get(itemAtom(id)).count
});
```

```tsx
// use
import { useRecoilState, useRecoilValue, RecoilRoot } from 'recoil';
const total = useRecoilValue(cartTotal);
const [items, setItems] = useRecoilState(cartItems);
```

### 4.3 提示
- 异步 selector（返回 Promise）可直接驱动 Suspense。  
- 合理拆分 atom，**避免巨 atom**；使用 family 做“按 key 管理”对象集合。

---

## 5) Jotai 🟩

### 5.1 何时用
- 比 Recoil 更**极简**的原子模型；**无需上下文 Provider**（v2 支持多作用域）、更接近“把 state 拉出组件”的感觉；生态里有 `jotai/query`、`jotai/immer` 等配件。

### 5.2 基本用法
```ts
// atoms.ts
import { atom } from 'jotai';

export const countAtom = atom(0);
export const doubleAtom = atom((get) => get(countAtom) * 2);
export const incAtom = atom(null, (_get, set) => set(countAtom, c => c + 1));
```

```tsx
// use
import { useAtom, useAtomValue, useSetAtom } from 'jotai';
const [count, setCount] = useAtom(countAtom);
const double = useAtomValue(doubleAtom);
const inc = useSetAtom(incAtom);
```

### 5.3 提示
- 写入原子（**write atom**）统一变更路径；与 React Query 可通过 `jotai/query` 互操作。  
- 原子过多时注意命名与分组；**派生优先**，不要把可计算的再存一份。

---

## 6) 性能与可维护性清单 ⚙️

- **细粒度订阅**：Redux 用 selector + `useSelector`；Zustand 使用选择器 + `shallow`；Recoil/Jotai 天生原子化。  
- **不可变更新**：Redux（Immer）自动；Zustand/原子系建议不可变，便于时间旅行/比较。  
- **SSR/Hydration**：  
  - Redux：在服务器 `preloadedState`，客户端 `hydrate`。  
  - Zustand：`createStore` + `useStore(store)` 或导出 `initStore`；避免在 Node 全局复用同一实例。  
  - Recoil/Jotai：用 `initializeState` / `Provider` 注入首屏值。  
- **持久化**：谨慎选择字段（token、偏好），不要把大 JSON 塞 localStorage。  
- **边界设计**：全局只放「跨路由共享、会被多处读取或写入」的**少数**状态。

---

## 7) 迁移策略 🛠️

- **从 Redux → Zustand/原子**：先把“纯 UI 局部状态”下放；保留审计/权限相关的“真正全局”。  
- **从原子 → Redux**：当需要**严格序列化、日志、审计、复杂中间件**时再上大仓库。  
- 渐进式：**同仓并存**，用模块边界隔离（比如 `@state/redux-*`、`@state/zustand-*`）。

---

## 8) 反模式与纠偏

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 把服务器数据复制进全局状态库 | 双份真相、失效难 | 交给 React Query/SWR；全局只存**引用/选择** |
| 巨无霸 store/atom | 任意改动全页面重渲 | 拆分切片（Redux）、多 store（Zustand）、原子化（Recoil/Jotai） |
| 重复存派生数据 | 状态不一致 | 用 memo/selector/派生原子 |
| 未做选择器订阅 | 细改动引发全局重渲 | 统一使用 selector/原子级订阅 |
| 滥持久化 | 本地存储胀大、隐私泄漏 | 仅白名单字段 + 版本化迁移 |

---

## 9) 提交前检查清单 ✅

- [ ] 服务器状态全部交由 **React Query/SWR**，不混入全局状态库。  
- [ ] 全局状态**最小化**：确需跨路由共享才入库。  
- [ ] 订阅使用 **selector/原子**，避免无关重渲。  
- [ ] 更新路径单一（Redux Action / Zustand setter / 写入原子）。  
- [ ] SSR/Hydration 策略与持久化字段明确。  
- [ ] 单测覆盖核心 reducer/setter/原子派生。

---

## 10) 练习 🏋️

1. 选一处把“组件内 useState 的跨路由依赖”上移到 **Zustand** 或 **Redux**，并写选择器。  
2. 使用 **Recoil/Jotai** 将“收藏列表”拆为 `itemsAtom` + `selectedIdsAtom` + `derivedTotal`，验证细粒度重渲。  
3. 与 5.1 集成：写一个评论提交（React Query mutation），成功后使依赖评论的全局视图状态自动刷新/同步。

---

## 11) TL;DR

- **分层**：服务器状态 ≠ 客户端状态。  
- **Redux**：团队规范与可审计；**Zustand**：简洁店铺；**Recoil/Jotai**：原子/细粒度。  
- **少存储，多派生**；**小订阅，大世界**。把状态当刀，不是收集癖。🗡️
