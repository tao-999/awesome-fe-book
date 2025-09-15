# 3.2 TS 类型系统与实战技巧 🧠

本章目标：把 **TypeScript（TS）** 的“强类型 + 类型推断 + 运行时守护”玩明白，让你在**建模、API、安全与工程**层面都更稳。内容包含：严格配置、判别联合、条件/模板字面量类型、`infer`、`satisfies`、品牌（Opaque）类型、运行时校验（Zod）、React 常用范式、项目工程与反模式。

---

## 0) 严格模式与工程基线

**推荐 `tsconfig.json`（严格 + 现代打包器友好）**：
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "useUnknownInCatchVariables": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "verbatimModuleSyntax": true,
    "types": []
  },
  "exclude": ["dist","node_modules"]
}
```

> 关键开关：  
> - `noUncheckedIndexedAccess` 让索引返回 `T | undefined`，逼你处理越界。  
> - `exactOptionalPropertyTypes` 区分“缺失”和“值为 `undefined`”。  
> - `useUnknownInCatchVariables` 让 `catch (e: unknown)` 更安全。

---

## 1) 建模领域：联合、判别与字面量

### 1.1 字面量联合（替代 `enum`）
```ts
type Role = 'user' | 'admin' | 'owner';
const r: Role = 'admin' as const;
```

### 1.2 判别联合（Discriminated Union）
```ts
type WeChat = { kind: 'wechat'; openid: string };
type Email  = { kind: 'email'; address: string };
type Login  = WeChat | Email;

function login(u: Login) {
  switch (u.kind) {
    case 'wechat': return bindOpenId(u.openid);
    case 'email' : return sendMagicLink(u.address);
    default:       return exhaust(u); // 保证穷尽
  }
}
function exhaust(x: never): never { throw new Error('unreachable'); }
```

> 通过单一判别字段 `kind` 让分支**类型收窄**；`exhaust` 保证新增变体时编译报错。

---

## 2) 品牌（Opaque）类型与范围约束

### 2.1 品牌 ID（防止“拿错 ID”）
```ts
declare const UserIdBrand: unique symbol;
type UserId = string & { readonly [UserIdBrand]: true };

const asUserId = (s: string): UserId => s as UserId;

function getUser(id: UserId) {/* ... */}
getUser(asUserId('u_123')); // 普通 string 不能误传
```

### 2.2 范围/格式约束（构造时校验）
```ts
declare const PercentBrand: unique symbol;
type Percent = number & { readonly [PercentBrand]: true };
function makePercent(n: number): Percent {
  if (Number.isFinite(n) && n >= 0 && n <= 1) return n as Percent;
  throw new Error('0..1 之间');
}
```

---

## 3) 自定义类型守卫与断言函数

```ts
type Json = null | boolean | number | string | Json[] | { [k: string]: Json };

export function isRecord(x: unknown): x is Record<string, unknown> {
  return typeof x === 'object' && x !== null && !Array.isArray(x);
}

export function assert(condition: unknown, msg='Assertion failed'): asserts condition {
  if (!condition) throw new Error(msg);
}

function parseMaybeJSON(s: string): Json | undefined {
  try { const v = JSON.parse(s); if (isRecord(v) || Array.isArray(v)) return v as Json; } catch {}
  return;
}
```

> `is` 返回**类型谓词**；`asserts` 让失败时“后续分支视为已收窄”。

---

## 4) 条件类型 & `infer`（类型层“函数”）

### 4.1 常用工具的实现思路
```ts
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;
type NonEmptyArray<T> = [T, ...T[]];
type ElementOf<T> = T extends (infer U)[] ? U : never;
type PropsOf<C> = C extends React.ComponentType<infer P> ? P : never;
```

### 4.2 深度工具（演示）
```ts
type DeepReadonly<T> =
  T extends Function ? T :
  T extends Map<infer K, infer V> ? ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>> :
  T extends Set<infer U> ? ReadonlySet<DeepReadonly<U>> :
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } :
  T;

type Simplify<T> = { [K in keyof T]: T[K] } & {};
```

---

## 5) 模板字面量类型 / 键重映射（Mapped Types）

```ts
type EventMap = {
  'user:created': { id: string };
  'user:deleted': { id: string; soft: boolean };
};

// 事件名 union：
type EventName = keyof EventMap; // "user:created" | "user:deleted"

// 事件 → 载荷：
type Payload<E extends EventName> = EventMap[E];

// Key 重映射（将 foo_bar 转 fooBar）
type Camelize<S extends string> =
  S extends `${infer H}_${infer T}` ? `${H}${Capitalize<Camelize<T>>}` : S;

type Keys = 'user_name' | 'user_id';
type CamelKeys = Camelize<Keys>; // "userName" | "userId"

type Remap<T> = {
  [K in keyof T as Camelize<K & string>]: T[K]
};
```

---

## 6) `satisfies` 与 `as const`（校验 + 精确推断）

```ts
const routes = {
  home: '/', post: (id: number) => `/post/${id}`
} as const;

// 校验对象“满足接口”，同时保留更窄的字面量类型：
type RouteSpec = { [k: string]: string | ((...a: any[]) => string) };
const spec = routes satisfies RouteSpec; // ✅

function go(path: (typeof routes)['home']) { /* path 为 "/" 字面量 */ }
```

---

## 7) API 类型安全：运行时校验配合 TS

纯 TS 只在**编译时**安全；要防御**服务端/第三方**返回数据，需**运行时校验**。

### 7.1 Zod（轻量示例）
```ts
import { z } from 'zod';

const User = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  role: z.enum(['user','admin','owner'])
});
type User = z.infer<typeof User>;

async function getUser(id: string, signal?: AbortSignal): Promise<User> {
  const res = await fetch(`/api/users/${id}`, { signal });
  const json = await res.json();
  return User.parse(json); // 运行时报错则抛异常
}
```

### 7.2 `fetch` 泛型 + 校验器注入
```ts
async function getJSON<T>(
  input: RequestInfo, init?: RequestInit & { schema?: { parse(x: unknown): T } }
): Promise<T> {
  const res = await fetch(input, init);
  const data = await res.json();
  return init?.schema ? init.schema.parse(data) : (data as T);
}

// 用法：
const u = await getJSON('/api/user/1', { schema: User });
```

### 7.3 错误安全（`unknown` + 规范化）
```ts
function toError(e: unknown): Error {
  return e instanceof Error ? e : new Error(typeof e === 'string' ? e : 'Unknown error');
}
```

---

## 8) React / 前端常用范式（TS 友好）

### 8.1 Props、Children、Ref
```ts
type ButtonProps = React.PropsWithChildren<{
  kind?: 'primary' | 'ghost';
  onClick?: React.MouseEventHandler<HTMLButtonElement>;
}>;
export function Button({ kind = 'primary', children, ...rest }: ButtonProps) {
  return <button data-kind={kind} {...rest}>{children}</button>;
}

// Ref：避免 ! 非空断言；使用回调 ref
const Input = React.forwardRef<HTMLInputElement, { value: string }>((p, ref) =>
  <input ref={ref} defaultValue={p.value} />
);
```

### 8.2 Reducer + 判别联合（可穷尽）
```ts
type Action =
  | { type: 'inc'; step?: number }
  | { type: 'reset' };

type State = { count: number };

function reducer(s: State, a: Action): State {
  switch (a.type) {
    case 'inc':   return { count: s.count + (a.step ?? 1) };
    case 'reset': return { count: 0 };
    default:      return exhaust(a);
  }
}
```

### 8.3 泛型 Hook（把约束交给类型）
```ts
function useMap<K, V>(init?: readonly (readonly [K, V])[]) {
  const m = React.useRef(new Map<K, V>(init));
  const set = (k: K, v: V) => { m.current.set(k, v); };
  return { get: (k: K) => m.current.get(k), set };
}
```

---

## 9) 实用工具类型速查

- 标配：`Partial<T>` / `Required<T>` / `Readonly<T>` / `Pick<T,K>` / `Omit<T,K>` / `Record<K,T>`  
- 函数：`Parameters<F>` / `ReturnType<F>` / `ConstructorParameters<C>` / `InstanceType<C>`  
- 其他：`NonNullable<T>` / `Awaited<T>`  
- 小工具：
```ts
type Mutable<T> = { -readonly [K in keyof T]: T[K] };
type Writeable<T> = { -readonly [P in keyof T]-?: T[P] };
type Merge<A,B> = Simplify<Omit<A, keyof B> & B>;
```

---

## 10) 项目工程：类型检查、声明与多包

### 10.1 CI 类型检查（不产物）
```json
// package.json
{ "scripts": { "typecheck": "tsc --noEmit" } }
```

### 10.2 库打包并生成 `.d.ts`
```json
// package.json
{
  "types": "dist/index.d.ts",
  "exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" } }
}
```
使用 **tsup**：
```json
{ "scripts": { "build": "tsup src/index.ts --dts --format esm,cjs" } }
```

### 10.3 多包（Monorepo）与引用
`tsconfig.base.json` 统一规则；各包 `composite: true`，启用 **Project References** 以加速增量编译。

---

## 11) 性能与复杂度

- **类型体操要节制**：层层嵌套的条件/模板类型会拖慢编译；用 `Simplify<T>` 展平。  
- **宽接口、窄实现**：公共 API 类型适度宽松（联合/可选），实现内部再收窄。  
- **避免 any**：若必须逃逸，用 `unknown` + 类型守卫，保留安全边界。

---

## 12) 反模式与纠偏

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 滥用 `any` | 安全性坍塌 | 用 `unknown` + 类型守卫；逐步还债 |
| `enum` 到处飞 | 与 JS 互操作差、树摇难 | 用字面量联合 + `as const` |
| 运行时无验证 | API 改动后静默炸裂 | Zod/valibot 做入口校验 |
| 多处写死字符串键 | 拼写易错 | 用模板字面量类型或常量对象 + `satisfies` |
| `!` 非空断言成瘾 | 运行时 NPE | 改为 `if (!x) return`/守卫/可选链 |
| Reducer 不穷尽 | 新增分支漏处理 | `never`-exhaust 检查 |

---

## 13) 提交前检查清单 ✅

- [ ] `tsc --noEmit` 与 Lint 均通过；`strict` 系列开关开启。  
- [ ] API 边界处有**运行时校验**（至少关键路径用 Zod）。  
- [ ] 判别联合穷尽；无 `switch` 漏分支。  
- [ ] 无 `any`（或集中、带注释解释）；`unknown` 有类型守卫。  
- [ ] 公共类型暴露清晰；库产出含 `.d.ts`。  

---

## 14) 练习 🏋️

1. 用“判别联合 + `never` 穷尽”重构你项目里的状态机/Reducer。  
2. 为一个真实 API 写 Zod 模型，并把 `fetch` 包装成“**校验器可插拔**”的 `getJSON<T>`。  
3. 实现 `DeepReadonly<T>` 与 `Simplify<T>`，并在大型类型上做编译时间对比。  

---

## 15) 速用片段（收藏）

```ts
// 断言 + never 穷尽
export const unreachable = (x: never): never => { throw new Error(`Unreachable: ${String(x)}`); };

// Narrow: truthy
export function isTruthy<T>(x: T): x is Exclude<T, 0 | '' | false | null | undefined> { return Boolean(x); }

// Safe JSON
export const safeJson = <T>(s: string): [T | null, Error | null] => {
  try { return [JSON.parse(s) as T, null]; } catch (e) { return [null, e as Error]; }
};

// Result 风格
export type Ok<T> = { ok: true;  value: T };
export type Err<E=Error> = { ok: false; error: E };
export const ok = <T>(v: T): Ok<T> => ({ ok: true, value: v });
export const err = <E>(e: E): Err<E> => ({ ok: false, error: e });
```

---

**小结**：TS 的威力来自“**建模 + 收窄 + 校验 + 工程**”的闭环。把判别联合、`satisfies`、品牌类型与运行时校验串起来，你的前端就能在**编译期防错、运行期兜底、协作期自证**。🚀
