# 4.2 可组装组件模式（Hooks / Render Props / Headless）🧩

本章目标：把组件拆成**可复用逻辑**与**可替换外观**，让你用 **Hooks**、**Render Props**、**Headless（无样式组件）** 三板斧，快速拼出**可访问（a11y）**、**可测试**、**可扩展**的 UI。示例主打 React（TS），并给出 Vue 3 对照（组合式 API + Slots）。

---

## 0) 设计心法：分而治之

- **逻辑 × 视图解耦**：逻辑走 Hook/Composable，视图交给使用方（Headless）。
- **受控 / 非受控**：优先支持“受控优先，可退非受控”，用统一的 `onChange(value)` 约定。
- **可访问优先**：键盘、聚焦、语义与 ARIA 优先设计，别让样式绑架交互。
- **组合而非继承**：提供**小而稳的原语**（Hooks、Context、Prop Getters），由使用方装配。
- **最小 API 面**：输入输出明确、事件少即是多；留“扩展点”（slots / render / as / overrides）。

---

## 1) 术语速记

- **Hook**：抽离“状态 + 行为”的可复用逻辑（React `useXxx` / Vue `useXxx`）。
- **Render Props**：组件把渲染权交给函数子组件（`children` 是函数）。
- **Headless 组件**：只提供状态/语义/行为，不内置外观（可选提供 `as` 属性）。
- **Prop Getters**：返回要绑定到元素上的 **props 包**（`getButtonProps()`），避免漏绑 a11y/事件。
- **Compound Components**：`<Select><Select.Trigger/><Select.List/><Select.Option/></Select>` 通过 Context 协作。

---

## 2) Hook：把“会动的部分”抽出来

### 2.1 `useControllableState`（受控/非受控统一）

```ts
// utils/useControllableState.ts
import { useCallback, useRef, useState } from 'react';

type Opts<T> = {
  value?: T;
  defaultValue?: T;
  onChange?: (v: T) => void;
};
export function useControllableState<T>({ value, defaultValue, onChange }: Opts<T>) {
  const [uncontrolled, setUnc] = useState<T | undefined>(defaultValue);
  const isControlled = value !== undefined;
  const v = isControlled ? (value as T) : (uncontrolled as T);
  const onChangeRef = useRef(onChange);
  onChangeRef.current = onChange;

  const set = useCallback((next: T) => {
    if (!isControlled) setUnc(next);
    onChangeRef.current?.(next);
  }, [isControlled]);
  return [v, set] as const;
}
```

### 2.2 折叠面板原语 `useDisclosure`

```ts
// hooks/useDisclosure.ts
import { useCallback } from 'react';
import { useControllableState } from '../utils/useControllableState';

export function useDisclosure(opts: { open?: boolean; defaultOpen?: boolean; onOpenChange?: (o: boolean) => void } = {}) {
  const [open, setOpen] = useControllableState<boolean>({
    value: opts.open,
    defaultValue: opts.defaultOpen ?? false,
    onChange: opts.onOpenChange
  });

  const toggle = useCallback(() => setOpen(!open), [open, setOpen]);

  function getButtonProps(props: React.ButtonHTMLAttributes<HTMLButtonElement> = {}) {
    return {
      'aria-expanded': open,
      'aria-controls': props['aria-controls'],
      onClick: (e: React.MouseEvent<HTMLButtonElement>) => {
        props.onClick?.(e);
        toggle();
      },
      ...props
    } as const;
  }

  function getPanelProps(props: React.HTMLAttributes<HTMLElement> = {}) {
    return { role: 'region', hidden: !open, ...props } as const;
  }

  return { open, setOpen, toggle, getButtonProps, getPanelProps } as const;
}
```

> **Prop Getters** 避免漏绑 `aria-expanded`、`role` 与事件合并的各种坑。

---

## 3) Render Props：把渲染权交给使用者

```tsx
// components/DisclosureRP.tsx
import * as React from 'react';
import { useDisclosure } from '../hooks/useDisclosure';

type Props = {
  open?: boolean;
  defaultOpen?: boolean;
  onOpenChange?: (o: boolean) => void;
  children: (api: ReturnType<typeof useDisclosure>) => React.ReactNode;
};

export function DisclosureRP({ children, ...opts }: Props) {
  const api = useDisclosure(opts);
  return <>{children(api)}</>;
}

// 用法
/*
<DisclosureRP defaultOpen>
  {({ open, getButtonProps, getPanelProps }) => (
    <section>
      <button {...getButtonProps({ 'aria-controls': 'p1' })}>{open ? '收起' : '展开'}</button>
      <div id="p1" {...getPanelProps()}>{open && '内容……'}</div>
    </section>
  )}
</DisclosureRP>
*/
```

**优点**：最大灵活度、零样式耦合。  
**缺点**：嵌套深时易“函数体海啸”，需搭配小块拆分与 `useMemo` 控制渲染。

---

## 4) Headless：语义/行为内置，外观交给你

### 4.1 Headless Disclosure（无样式）

```tsx
// components/Disclosure.tsx（Headless + Compound）
import * as React from 'react';
import { useDisclosure } from '../hooks/useDisclosure';

type Ctx = ReturnType<typeof useDisclosure>;
const Ctx = React.createContext<Ctx | null>(null);
const useCtx = () => {
  const c = React.useContext(Ctx);
  if (!c) throw new Error('Disclosure.* 必须在 <Disclosure> 内使用');
  return c;
};

export function Disclosure(props: React.PropsWithChildren<{ open?: boolean; defaultOpen?: boolean; onOpenChange?: (o:boolean)=>void }>) {
  const api = useDisclosure(props);
  return <Ctx.Provider value={api}>{props.children}</Ctx.Provider>;
}

export function DisclosureButton(props: React.ButtonHTMLAttributes<HTMLButtonElement>) {
  const { getButtonProps } = useCtx();
  return <button {...getButtonProps(props)} />;
}

export function DisclosurePanel(props: React.HTMLAttributes<HTMLDivElement>) {
  const { getPanelProps, open } = useCtx();
  return <div {...getPanelProps(props)} aria-hidden={!open} />;
}
```

**使用：**
```tsx
<Disclosure defaultOpen>
  <DisclosureButton className="btn">切换</DisclosureButton>
  <DisclosurePanel className="panel">内容区块</DisclosurePanel>
</Disclosure>
```

> 这种 **Compound + Context + Prop Getters** 的 Headless 模式，是构建复杂组件（Select/Combobox/Dialog/Toast）的通用套路。

---

## 5) 列表选择原语：`useListbox`（a11y 就位）

**要点**：  
- 集中管理 **焦点与选中**；  
- 用 **roving tabIndex**（只有一个元素 `tabIndex=0` 可 Tab 进入，其他 `-1`）或 `aria-activedescendant`；  
- 选项用 `role="option"`，容器用 `role="listbox"`，选中项加 `aria-selected="true"`。

```ts
// hooks/useListbox.ts
import { useCallback, useId, useMemo, useRef, useState } from 'react';

type Item = { id: string; label: string; disabled?: boolean };
type Opts = {
  items: Item[];
  value?: string | null;
  defaultValue?: string | null;
  onChange?: (id: string | null) => void;
};

export function useListbox({ items, value, defaultValue=null, onChange }: Opts) {
  const [val, setVal] = useControllableState<string | null>({ value, defaultValue, onChange });
  const [activeIndex, setActiveIndex] = useState(() => Math.max(0, items.findIndex(i => i.id === val)));
  const listId = useId();
  const optionId = (i: number) => `${listId}-opt-${i}`;

  const move = useCallback((dir: 1 | -1) => {
    let i = activeIndex;
    do { i = (i + dir + items.length) % items.length; } while (items[i]?.disabled);
    setActiveIndex(i);
  }, [activeIndex, items]);

  const select = useCallback((i: number) => {
    if (items[i]?.disabled) return;
    setVal(items[i]?.id ?? null);
    setActiveIndex(i);
  }, [items, setVal]);

  function getListProps(props: React.HTMLAttributes<HTMLElement> = {}) {
    return {
      role: 'listbox',
      'aria-activedescendant': optionId(activeIndex),
      tabIndex: 0,
      onKeyDown: (e: React.KeyboardEvent) => {
        if (e.key === 'ArrowDown') { e.preventDefault(); move(1); }
        else if (e.key === 'ArrowUp') { e.preventDefault(); move(-1); }
        else if (e.key === 'Enter' || e.key === ' ') { e.preventDefault(); select(activeIndex); }
        props.onKeyDown?.(e);
      },
      ...props
    } as const;
  }

  function getOptionProps(i: number, props: React.HTMLAttributes<HTMLElement> = {}) {
    const isSelected = items[i]?.id === val;
    return {
      id: optionId(i),
      role: 'option',
      'aria-selected': isSelected || undefined,
      'data-active': i === activeIndex ? '' : undefined,
      'data-selected': isSelected ? '' : undefined,
      tabIndex: i === activeIndex ? 0 : -1, // roving
      onMouseMove: (e: React.MouseEvent) => { setActiveIndex(i); props.onMouseMove?.(e); },
      onClick: (e: React.MouseEvent) => { select(i); props.onClick?.(e); },
      ...props
    } as const;
  }

  return { value: val, activeIndex, getListProps, getOptionProps } as const;
}
```

**Headless 视图：**
```tsx
// components/Listbox.tsx
import * as React from 'react';
import { useListbox } from '../hooks/useListbox';

export function Listbox(props: { items: { id:string; label:string; disabled?:boolean }[]; value?: string|null; defaultValue?: string|null; onChange?: (id:string|null)=>void }) {
  const api = useListbox(props);
  return (
    <div {...api.getListProps({ className: 'listbox' })}>
      {props.items.map((it, i) => (
        <div key={it.id} {...api.getOptionProps(i, { className: `option ${it.disabled?'is-disabled':''}` })}>
          {it.label}
        </div>
      ))}
    </div>
  );
}
```

---

## 6) API 设计规范小抄（React）

- **受控/非受控**：`value` + `defaultValue` + `onChange(value)` 三件套。  
- **多值选择**：统一使用“集合”表示（如 `string[]`），避免复合对象。  
- **`as` 属性**：允许使用方替换外层元素类型（多用在 Headless）。  
- **`ref` 转发**：`forwardRef` + `useImperativeHandle` 暴露必要方法（如 `focus()`）。  
- **事件合并**：Prop Getters 内**合并调用**外部事件，避免“内部覆盖外部”。

---

## 7) Vue 3 对照（Composable + Slots + provide/inject）

### 7.1 `useDisclosure`（Vue 版）

```ts
// composables/useDisclosure.ts
import { computed, ref, toRef, watch } from 'vue';

export function useDisclosure(opts: { modelValue?: boolean; defaultOpen?: boolean; 'onUpdate:modelValue'?: (v:boolean)=>void } = {}) {
  const isControlled = opts.modelValue !== undefined;
  const inner = ref<boolean>(opts.defaultOpen ?? false);
  const open = computed({
    get: () => isControlled ? (opts.modelValue as boolean) : inner.value,
    set: (v: boolean) => {
      if (!isControlled) inner.value = v;
      opts['onUpdate:modelValue']?.(v);
    }
  });
  const toggle = () => open.value = !open.value;

  const getButtonProps = (p: Record<string, any> = {}) => ({
    'aria-expanded': open.value, ...p, onClick: (e: any) => { p.onClick?.(e); toggle(); }
  });
  const getPanelProps = (p: Record<string, any> = {}) => ({ role:'region', hidden: !open.value, ...p });

  return { open, toggle, getButtonProps, getPanelProps } as const;
}
```

### 7.2 Headless 组件（Slots）

```vue
<!-- components/Disclosure.vue -->
<script setup lang="ts">
import { provide, inject } from 'vue';
import { useDisclosure } from '../composables/useDisclosure';

const props = defineProps<{ modelValue?: boolean; defaultOpen?: boolean }>();
const emit = defineEmits<{ 'update:modelValue': [boolean] }>();
const api = useDisclosure({ modelValue: props.modelValue, defaultOpen: props.defaultOpen, 'onUpdate:modelValue': v => emit('update:modelValue', v) });
provide('disclosure', api);
</script>

<template>
  <slot />
</template>
```

```vue
<!-- components/DisclosureButton.vue -->
<script setup lang="ts">
import { inject } from 'vue';
const api = inject<any>('disclosure');
const props = defineProps<{ as?: string }>();
const Tag = (props.as ?? 'button') as any;
</script>

<template>
  <component :is="Tag" v-bind="api.getButtonProps($attrs)">
    <slot />
  </component>
</template>
```

```vue
<!-- components/DisclosurePanel.vue -->
<script setup lang="ts">
import { inject } from 'vue';
const api = inject<any>('disclosure');
</script>
<template>
  <div v-bind="api.getPanelProps($attrs)"><slot /></div>
</template>
```

---

## 8) State Reducer 模式（用户可重写内部决策）

> 当组件内部状态迁移需要**可插拔策略**（例如可否选择被禁用项、输入法合成期间的键盘策略），暴露 `stateReducer(prevState, action) => nextState`：

```ts
type Action =
  | { type: 'move'; dir: 1 | -1 }
  | { type: 'select'; index: number };

type State = { activeIndex: number; value: string | null };

export function useListboxWithReducer(opts: { /* ... */; stateReducer?: (s: State, a: Action) => State }) {
  // 省略：构造默认 nextState
  function dispatch(a: Action) {
    const next = defaultReduce(state, a);
    setState(opts.stateReducer ? opts.stateReducer(state, a) : next);
  }
}
```

---

## 9) a11y 关键要点（必须拿分）

- **键盘**：`Enter/Space` 激活、`Esc` 关闭、`Arrow*` 导航、`Home/End` 边界跳转。  
- **焦点管理**：对话框/菜单内**焦点陷阱**与初始焦点；关闭后回到触发点。  
- **语义角色**：`button`、`listbox/option`、`menuitem`、`dialog`、`switch/checkbox` 等；别乱用 `div`。  
- **ARIA 最小集**：只在原生语义不足时补；避免与原生冲突（详见 1.1/1.2 章节）。

---

## 10) 性能与工程

- **Context 切分**：用多个 Context / **selector** 避免无关重渲染。  
- **虚拟化**：长列表用虚拟滚动（> 200 项建议开启）。  
- **事件合并与稳定引用**：`useCallback`/`useEvent`（或本地实现 `useStableCallback`）降低 diff 与订阅成本。  
- **测试**：Hook 单测（Jest/Vitest + RTL Hooks），Headless 组件用 Story 的 `play` + a11y 检查。

---

## 11) 反模式与纠偏

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 把样式写死在逻辑里 | 无法换肤/品牌化 | Headless：只给语义与状态，样式交给调用方 |
| 漏绑 ARIA/键盘 | 视觉正常但无障碍差 | 用 Prop Getters 封装必要属性 |
| 只支持非受控 | 难以表单/URL 同步 | `value/defaultValue/onChange` 三件套 |
| Context 巨无霸 | 任一变化全树重渲 | 拆分 Context + selector 或局部状态 |
| Render Props 过度 | 回调地狱 | 仅在需要最大灵活时使用，或改 Headless Compound |

---

## 12) 提交前检查清单 ✅

- [ ] 逻辑在 Hook；视图可替换（Headless/Render Props）。  
- [ ] 受控/非受控统一，通过 `onChange` 单一出口。  
- [ ] 提供 Prop Getters，绑定 ARIA/事件合并不丢失。  
- [ ] 键盘行为 & 焦点管理完备；a11y 面板无高危警告。  
- [ ] 单测覆盖关键状态迁移；Story `play` 覆盖主交互路径。  
- [ ] 文档示例：默认样式 + 自定义外观两套用法。

---

## 13) 练习 🏋️

1. 将你现有的“下拉选择”重写为 **Headless + useListbox**，支持键盘导航与受控模式。  
2. 为“日期选择”抽出 `useCalendar`，把视图留给 Slots/Render Props；加入 `stateReducer` 扩展点。  
3. 用 Storybook 写三组故事：默认皮肤、自定义 Tailwind 皮肤、移动端紧凑布局；为其编写 `play` 断言键盘行为。

---

## 14) TL;DR

> **Hook 抽逻辑 → Headless 给语义与行为 → Render Props/Slots 交渲染权**。受控/非受控统一、a11y 优先、Prop Getters 防漏，组合出“既能换装又能上生产”的组件。🚀
