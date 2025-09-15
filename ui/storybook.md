# 4.1 Storybook / Docs / 可视化用例 📚🧩

本章目标：把**组件即文档（Docs）**、**组件即测试（Stories + Play + 视觉回归）**、**组件即用例（交互脚本）**整合到一套可复制的 Storybook 工作流里。你将得到：安装到首个故事、自动文档、交互测试、可视化回归、数据 Mock、主题/i18n 切换、CI 集成与质量检查清单。示例以 React 为主，附 Vue 对照。

---

## 0) 什么时候用 Storybook？

- 设计系统/组件库建设（跨项目复用，统一规范）。  
- 复杂 UI 的**可交互用例**（把“产品需求”沉淀成可运行的故事）。  
- 无后端/不连环境也能开发（MSW mock）。  
- 自动化**可访问性/交互/视觉**检查（a11y、Play、test-runner、Chromatic/视觉回归）。  
- 作为**产品文档**输出（DocsPage/MDX 导出为静态站点）。

---

## 1) 安装与初始化（React / Vue 对照）

> 建议 Node ≥ 18，包管任选（示例用 pnpm）。Storybook 7/8 都可用；若你用 v7：`main.ts`；v8：`storybook.config.ts`。内容一致。

```bash
# React
pnpm dlx storybook@latest init --type react

# Vue 3
pnpm dlx storybook@latest init --type vue
```

目录关键物：  
```
.storybook/
  main.ts            # v7 用名；v8 叫 storybook.config.ts
  preview.ts         # 预览配置（全局装饰器、参数）
src/
  components/Button.tsx
  components/Button.stories.tsx
```

**最小配置（React，v7 命名）**：
```ts
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/react-vite';
const config: StorybookConfig = {
  framework: '@storybook/react-vite',
  stories: ['../src/**/*.stories.@(ts|tsx|mdx)'],
  addons: [
    '@storybook/addon-essentials',        // actions, controls, docs, viewport 等
    '@storybook/addon-interactions',      // Play 函数与 test-runner
    '@storybook/addon-a11y'               // 可访问性
  ],
  docs: { autodocs: true }                 // 自动文档
};
export default config;
```

```ts
// .storybook/preview.ts
import type { Preview } from '@storybook/react';
const preview: Preview = {
  parameters: {
    actions: { argTypesRegex: '^on[A-Z].*' },
    controls: { expanded: true, matchers: { color: /(background|color)$/i, date: /Date$/ } },
    layout: 'centered'
  }
};
export default preview;
```

---

## 2) 从“第一个故事”开始（CSF 3）

**Button 组件**
```tsx
// src/components/Button.tsx
import React from 'react';
type Props = React.PropsWithChildren<{
  kind?: 'primary' | 'ghost';
  disabled?: boolean;
  onClick?: React.MouseEventHandler<HTMLButtonElement>;
}>;
export function Button({ kind='primary', disabled, children, ...rest }: Props) {
  return (
    <button
      data-kind={kind}
      disabled={disabled}
      style={{
        padding: '8px 16px',
        borderRadius: 6,
        color: kind === 'primary' ? '#fff' : '#111',
        background: kind === 'primary' ? '#2563eb' : '#fff',
        border: kind === 'primary' ? '1px solid transparent' : '1px solid #e5e7eb',
        cursor: disabled ? 'not-allowed' : 'pointer'
      }}
      {...rest}
    >
      {children}
    </button>
  );
}
```

**CSF3 故事**
```tsx
// src/components/Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta = {
  title: 'Core/Button',
  component: Button,
  tags: ['autodocs'],                // 开启自动文档
  args: { children: '点击我' },        // 默认 Args
  argTypes: {
    kind: { control: 'radio', options: ['primary', 'ghost'] }
  }
} satisfies Meta<typeof Button>;
export default meta;

type Story = StoryObj<typeof meta>;

export const Primary: Story = { args: { kind: 'primary' } };
export const Ghost:   Story = { args: { kind: 'ghost' } };

// 交互用例（Play 函数）
export const ClickToFire: Story = {
  args: { children: '点这里触发事件' },
  play: async ({ canvasElement, step }) => {
    const c = within(canvasElement);
    await step('点击按钮', async () => {
      await userEvent.click(await c.findByRole('button'));
    });
  }
};
```

> `play` 依赖 `@storybook/test` 提供的测试工具（`within/userEvent/expect`）。如果没自动装，`pnpm add -D @storybook/test @testing-library/dom @testing-library/user-event`。

---

## 3) Docs：自动文档 + MDX

### 3.1 Autodocs（零成本）
- 给 `Meta` 标 `tags:['autodocs']` 或在 `main.ts` 里 `docs.autodocs = true`。  
- Props/事件/控件来自 TS 类型+JSDoc，控件映射来自 `argTypes`。

### 3.2 自定义 MDX 文档
```mdx
<!-- src/components/Button.docs.mdx -->
import { Meta, Story, ArgsTable, Primary } from '@storybook/addon-docs';
import * as ButtonStories from './Button.stories';

<Meta of={ButtonStories} title="文档/Button" />

# Button 组件

用于触发操作的主要按钮。支持 `primary` 与 `ghost` 两种风格。

<Primary />
<ArgsTable of={ButtonStories} />
```

---

## 4) Mock 数据与后端隔离（MSW）

> 目标：不连接真实后端，也能让故事“跑得起来”。

```bash
pnpm add -D msw msw-storybook-addon
```

```ts
// .storybook/preview.ts（片段）
import { initialize, mswDecorator } from 'msw-storybook-addon';
initialize();                               // 初始化 worker
export const decorators = [mswDecorator];   // 注册装饰器
```

```tsx
// src/stories/UserCard.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { http, HttpResponse } from 'msw';
import { UserCard } from './UserCard';

export default {
  title: 'Data/UserCard',
  component: UserCard
} satisfies Meta<typeof UserCard>;

type Story = StoryObj<typeof UserCard>;

export const Loaded: Story = {
  parameters: {
    msw: {
      handlers: [
        http.get('/api/user/42', () => HttpResponse.json({ id: 42, name: 'Ada' }))
      ]
    }
  }
};
```

---

## 5) 可访问性（a11y）、交互与视觉测试

### 5.1 a11y
- 安装 `@storybook/addon-a11y` 后，在面板可直接查看对比度/语义问题。  
- 在 CI 用 test-runner 自动跑 a11y 检查：

```bash
pnpm add -D @storybook/test-runner
pnpm storybook                               # 先起本地（或 build 出静态）
pnpm test-storybook                          # 运行 Play + a11y 基线
```

> `test-runner` 基于 Playwright；它会跑每个 story 的 `play`，并进行 a11y 扫描。

### 5.2 视觉回归（两种方案）

**A) 独立化（开源链）**
- `@storybook/test-runner` + `@playwright/test` 自己截屏对比（需要写基线管理逻辑）。  
- 或者 Playwright UI 测试指向 **静态 Storybook**（`pnpm build-storybook` → `storybook-static`）。

**B) 托管服务（上手快）**
- 接入 **Chromatic**（官方出品）：自动静态构建、按 PR 对比、审阅工作流、按故事/视口组合生成快照。  
- 也可用 Percy/Applitools 等。

> 视觉测试的关键：**冻结不稳定因素**（时间/随机/网络）——在故事里为日期注入固定种子，或在 `play` 前 stub 出 `Date.now()`。

---

## 6) 主题 / 视口 / 伪状态 / i18n

```ts
// .storybook/preview.ts（增量）
const preview = {
  parameters: {
    viewport: { defaultViewport: 'responsive' },
    backgrounds: { default: 'light', values: [{ name: 'light', value: '#ffffff' }, { name: 'dark', value: '#0b1020' }] },
    pseudo: { hover: ['[data-hover]'], focus: ['[data-focus]'] } // 需要安装 @ergosign/storybook-addon-pseudo-states
  },
  decorators: [
    (Story, ctx) => {
      document.documentElement.dataset.theme = ctx.globals.theme ?? 'light';
      return <Story />;
    }
  ],
  globals: {
    theme: 'light',                // 自定义全局开关（展示在工具条上可加 toolbar）
    locale: 'zh-CN'
  }
};
export default preview;
```

---

## 7) 组件“契约用例”：Play + 断言（把需求写进故事）

> 用 `play` 写“交互脚本 + 断言”。这既是 demo，也是 E2E-like 的契约测试。

```tsx
// Counter.stories.tsx
import { expect, userEvent, within } from '@storybook/test';
export const Increase = {
  play: async ({ canvasElement }) => {
    const c = within(canvasElement);
    const btn = await c.findByRole('button', { name: /加一/i });
    await userEvent.click(btn);
    await expect(c.getByText(/计数：1/)).toBeInTheDocument();
  }
};
```

在 CI 用 `pnpm test-storybook` 跑所有 `play`。**收益**：当你改了 DOM 结构或 aria 文案，破坏用户旅程，CI 会立刻红。

---

## 8) 输出文档站点（静态化）

```bash
pnpm build-storybook
# 输出到 storybook-static/，可挂到任意静态托管（GitHub Pages、Vercel、S3、OSS）
```

---

## 9) CI 集成（GitHub Actions 样例）

**A) 构建 + 测试 + 视觉（以 Chromatic 为例）**
```yaml
name: storybook
on: [push, pull_request]
jobs:
  sb:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm build-storybook
      - run: pnpm test-storybook
      - name: Chromatic
        uses: chromaui/action@v1
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          storybookBuildDir: storybook-static
```

**B) 仅本地截屏对比（Playwright 自管）**：  
- 起本地 Storybook，跑 Playwright 截屏(compare baseline)，把基线存仓库/制品库。

---

## 10) 性能与稳定性小贴士 🏎️

- **Mock 所有不稳定 I/O**（MSW）：避免“偶尔挂一张图导致失败”。  
- **冻结时间与随机**：在装饰器里 `vi.setSystemTime`/`sinon.useFakeTimers` 或覆写 `Math.random`（注意还原）。  
- **拆分大型故事**：一故事只覆盖一个场景；复杂流程拆分为多个步骤故事。  
- **按需加载**：对重型依赖（图表/编辑器）用动态导入，故事中懒加载减少启动时间。  
- **构建器选型**：Vite 框架建议 `@storybook/*-vite`，可显著提升冷启动。

---

## 11) Vue 对照（快速示例）

```vue
<!-- src/components/VButton.vue -->
<script setup lang="ts">
defineProps<{ kind?: 'primary'|'ghost' }>();
</script>

<template>
  <button :data-kind="kind ?? 'primary'"><slot /></button>
</template>

<style scoped>
button[data-kind="primary"]{ background:#2563eb;color:#fff;border-radius:6px;padding:8px 16px; }
button[data-kind="ghost"]{ background:#fff;border:1px solid #e5e7eb;border-radius:6px;padding:8px 16px;color:#111; }
</style>
```

```ts
// src/components/VButton.stories.ts
import type { Meta, StoryObj } from '@storybook/vue3';
import VButton from './VButton.vue';

export default {
  title: 'Core/VButton',
  component: VButton,
  tags: ['autodocs'],
  args: { kind: 'primary', default: '点我' }
} satisfies Meta<typeof VButton>;

type Story = StoryObj<typeof VButton>;
export const Primary: Story = { args: { kind: 'primary' } };
export const Ghost:   Story = { args: { kind: 'ghost' } };
```

---

## 12) 常见坑 & 纠偏

| 坑 | 症状 | 纠偏 |
|---|---|---|
| 故事“看起来对”，CI 偶现红 | 未 mock 时间/网络/随机 | MSW + 冻结时间/随机；把动画/过渡关掉 |
| 控件（Controls）不可用 | Args/ArgTypes 未声明/类型丢失 | 确保组件 Props 有 TS 类型与导出；补 `argTypes` |
| 视觉回归噪点多 | 动画、阴影、异步图片 | 统一禁用动画，加载占位符；为图片设固定尺寸 |
| 文档漂移 | 产品改了，故事没改 | 把验收标准写进 `play`；PR 必须更新对应故事 |
| Monorepo 下重复打包慢 | 每包都起 Storybook | 用 **Storybook Composition** 聚合多个子库 |

---

## 13) 提交前检查清单 ✅

- [ ] 所有公共组件均有 **至少 3 个故事**：默认/禁用/错误或变体。  
- [ ] 关键用例含 **play 断言**，覆盖核心用户旅程。  
- [ ] a11y 检查通过（对焦点、对比度、语义）。  
- [ ] 视觉回归对关键组件生效（冻结时间/随机）。  
- [ ] Docs 自动生成（Autodocs）+ 关键组件有 MDX 说明。  
- [ ] CI：`build-storybook` + `test-storybook` +（可选）Chromatic。

---

## 14) 练习 🏋️

1. 给你项目中的“表单按钮/通知条/卡片”各写 3 个故事（默认/Loading/错误），为其中 1 个写 `play`。  
2. 用 MSW 模拟一个 `/api/todos`，实现 列表加载/失败/空数据 三个故事。  
3. 为“日期控件”故事冻结 `Date.now()`，接入视觉回归，消除不稳定像素差。

---

## 15) TL;DR（带走就能用）

- **故事是最小可验证需求**：在故事里写出“状态 + 交互 + 断言”。  
- **Mock 一切不稳定性**，把组件放在可控沙盒里开发与测试。  
- **Docs 与 CI 一体化**：`Autodocs + MDX` 出文档；`test-runner + 视觉回归` 保质量。  
- **组合拳**：Story（示例）＋ Play（脚本）＋ MSW（数据）＋ a11y（可用性）＋ CI（回归），从此“看得见的质量”不是口号。🚀
