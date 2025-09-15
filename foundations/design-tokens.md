# 2.3 设计令牌（Design Tokens）与主题化

本章目标：把**视觉与交互决策**抽象为可被代码消费的“最小事实集合”（Design Tokens），建立从设计 → 构建 → 运行时的统一管道，实现**多主题（亮/暗/品牌）**、**密度（紧凑/舒适）**与**平台（Web/iOS/Android）**的一处声明、多处生效。

---

## 1) 核心概念与分层

- **基础令牌（Core/Base）**：与品牌/语义无关的“原材料”。例：色板、字号刻度、间距刻度、圆角、阴影、动效时长。
- **语义令牌（Semantic/Alias）**：与场景绑定的别名。例：`color.bg.surface`、`color.fg.muted`、`border.focus`。
- **组件令牌（Component）**：特定组件的可变位。例：`button.primary.bg`、`card.radius`、`tag.warn.fg`。

> 原则：**组件只消费“语义/组件令牌”，不直接消费基础令牌**。基础变更不会直接“炸”组件。

---

## 2) 命名与刻度（推荐规范）

- 命名格式：`<category>.<role>[.<variant>][.<state>]`
  - 例：`color.fg.default`、`space.3`、`radius.md`、`shadow.lg`、`motion.fast`、`z.modal`。
- 刻度策略：**小而均匀**，便于压缩选择空间  
  - 间距：`space.1..8`（`2,4,6,8,12,16,24,32px` 等比例或指数刻度）  
  - 字号：`font.size.100..900`（搭配 `line-height`、`letter-spacing`）  
  - 圆角：`radius.sm/md/lg` + `pill/full`  
  - 动效：`motion.fast/normal/slow`（如 `120/200/320ms`）

---

## 3) 令牌源（W3C Tokens 格式示例）

`tokens/tokens.json`（**唯一真源**，含元数据与模式）：
```json
{
  "color": {
    "$type": "color",
    "palette": {
      "brand": { "50": "#eff6ff", "500": "#2563eb", "600": "#1d4ed8" },
      "gray":  { "50": "#f9fafb", "900": "#0b1020" },
      "red":   { "500": "#ef4444" },
      "green": { "500": "#22c55e" }
    }
  },
  "space": { "$type": "dimension", "1": "2px", "2": "4px", "3": "6px", "4": "8px", "6": "12px", "8": "16px", "12": "24px", "16": "32px" },
  "radius": { "$type": "dimension", "sm": "6px", "md": "8px", "lg": "12px", "pill": "9999px" },
  "shadow": { "$type": "shadow", "sm": { "color": "#00000033", "x": "0", "y": "1px", "blur": "2px", "spread": "0" }, "lg": { "color": "#00000040", "x": "0", "y": "8px", "blur": "24px", "spread": "-4px" } },
  "font": {
    "family": { "$type": "fontFamily", "sans": "ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, sans-serif" },
    "size": { "$type": "dimension", "100": "12px", "200": "14px", "300": "16px", "400": "18px", "500": "20px" },
    "lineHeight": { "$type": "dimension", "normal": "1.5", "tight": "1.35" }
  },
  "semantic": {
    "$type": "color",
    "light": {
      "bg": { "surface": "{color.palette.gray.50}", "elevated": "#ffffff" },
      "fg": { "default": "#111827", "muted": "#374151", "onBrand": "#ffffff" },
      "brand": { "bg": "{color.palette.brand.600}", "fg": "{semantic.light.fg.onBrand}" },
      "status": { "error": "{color.palette.red.500}", "success": "{color.palette.green.500}" },
      "border": { "subtle": "#e5e7eb", "focus": "{color.palette.brand.500}" }
    },
    "dark": {
      "bg": { "surface": "#0b1020", "elevated": "#111827" },
      "fg": { "default": "#e5e7eb", "muted": "#cbd5e1", "onBrand": "#0b1020" },
      "brand": { "bg": "{color.palette.brand.500}", "fg": "{semantic.dark.fg.onBrand}" },
      "status": { "error": "{color.palette.red.500}", "success": "{color.palette.green.500}" },
      "border": { "subtle": "#1f2937", "focus": "{color.palette.brand.500}" }
    }
  },
  "component": {
    "button": {
      "base": { "radius": "{radius.md}", "px": "{space.8}", "py": "{space.4}", "shadow": "{shadow.sm}" },
      "primary": { "bg": "{semantic.light.brand.bg}", "fg": "{semantic.light.brand.fg}" },
      "ghost": { "bg": "transparent", "fg": "{semantic.light.fg.default}", "border": "{semantic.light.border.subtle}" }
    },
    "card": { "bg": "{semantic.light.bg.elevated}", "radius": "{radius.lg}", "shadow": "{shadow.lg}" }
  },
  "$schema": "https://design-tokens.org/schema.json"
}
```

> 亮/暗模式用 **别名**引用同一路径；换主题时只切换别名根。

---

## 4) 构建管道：生成 CSS/TS/iOS/Android

以 **Style Dictionary** 为例（可替换为 Theo、token-transformer 等）。

`package.json` 片段：
```json
{ "scripts": { "build:tokens": "style-dictionary build" }, "devDependencies": { "style-dictionary": "^4.0.0" } }
```

`tokens/config.json`：
```json
{
  "source": ["tokens/tokens.json"],
  "platforms": {
    "css": {
      "transformGroup": "css",
      "buildPath": "dist/tokens/css/",
      "files": [
        { "destination": "variables.css", "format": "css/variables", "options": { "outputReferences": true } }
      ]
    },
    "ts": {
      "transformGroup": "js",
      "buildPath": "dist/tokens/ts/",
      "files": [{ "destination": "index.ts", "format": "javascript/es6" }]
    },
    "android": {
      "transformGroup": "android",
      "buildPath": "dist/tokens/android/",
      "files": [{ "destination": "tokens.colors.xml", "format": "android/colors" }]
    },
    "ios": {
      "transformGroup": "ios",
      "buildPath": "dist/tokens/ios/",
      "files": [{ "destination": "tokens.json", "format": "json/nested" }]
    }
  }
}
```

执行：
```bash
pnpm build:tokens
```

产物（节选）：`dist/tokens/css/variables.css`
```css
:root {
  --space-8: 16px;
  --radius-md: 8px;
  --font-size-300: 16px;
  --semantic-light-bg-surface: #f9fafb;
  --semantic-dark-bg-surface: #0b1020;
  --component-button-base-radius: var(--radius-md);
  /* ... */
}
```

---

## 5) 主题化落地（Web）

### 5.1 ITCSS × Cascade Layers 对齐（Settings 层注入令牌）
```css
@layer settings, tools, generic, elements, objects, components, utilities;

@layer settings {
  /* 基础：默认亮主题 */
  :root {
    color-scheme: light dark; /* 告诉 UA 存在双主题，表单/滚动条也适配 */
    --bg-surface: var(--semantic-light-bg-surface);
    --fg-default: #111827;
    --fg-muted: #374151;
    --brand-bg: var(--semantic-light-brand-bg);
    --brand-fg: var(--semantic-light-brand-fg);
    --border-subtle: #e5e7eb;
    --border-focus: var(--semantic-light-border-focus);
    --radius-md: 8px;
    --space-4: 8px; --space-8: 16px;
  }

  /* 暗主题覆盖（运行时切换 data-theme="dark"） */
  [data-theme="dark"] {
    --bg-surface: var(--semantic-dark-bg-surface);
    --fg-default: #e5e7eb;
    --fg-muted: #cbd5e1;
    --brand-bg: var(--semantic-dark-brand-bg);
    --brand-fg: var(--semantic-dark-brand-fg);
    --border-subtle: #1f2937;
    --border-focus: var(--semantic-dark-border-focus);
  }

  /* 密度（紧凑） */
  [data-density="compact"] {
    --space-4: 6px;
    --space-8: 12px;
    --radius-md: 6px;
  }
}
```

### 5.2 组件消费（Components 层仅引用变量，不写硬编码）
```css
@layer components {
  .btn {
    display:inline-flex; align-items:center; gap:.5em;
    padding: var(--space-4) var(--space-8);
    border-radius: var(--radius-md);
    border: 1px solid var(--border-subtle);
    background: #fff; color: var(--fg-default);
  }
  .btn--primary { background: var(--brand-bg); color: var(--brand-fg); border-color: transparent; }
  .card { background: var(--bg-surface); border-radius: var(--radius-md); box-shadow: 0 8px 24px -4px #00000040; }
}
```

### 5.3 运行时切换（SSR 安全、闪烁最小化）
```html
<script>
  // SSR 前置：尽量在 <head> 里同步执行，避免“闪白/闪黑”
  (function(){
    const pref = localStorage.getItem('theme');
    const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    document.documentElement.dataset.theme = pref || (prefersDark ? 'dark' : 'light');
  })();
</script>
```

---

## 6) 多品牌/多租户

- 规则：**品牌只影响语义根，不直连组件**。以 data 属性隔离：
```css
/* brand=blue / green 切换品牌主色 */
:root[data-brand="blue"]  { --brand-bg: #1d4ed8; --brand-fg: #ffffff; }
:root[data-brand="green"] { --brand-bg: #16a34a; --brand-fg: #052e16; }
```
- 多租户建议在 **HTML 根**打 tag（`data-tenant="acme"`），产出**租户覆盖层**样式或动态装配对应 tokens 片段。

---

## 7) 对比度与可访问性（把可用性内化进令牌）

- **前景/背景配对**：维护 `pair.*` 令牌（如 `pair.primary.bg` + `pair.primary.fg`），在设计阶段即保证 WCAG AA（4.5:1 或 3:1 for large）。
- **状态色**：不要只定义色相，定义**带语义的对比对**：`status.error.bg/fg/border`。
- **焦点样式**：把 `--border-focus`、`--ring.focus` 做成令牌，指向品牌色的对比友好梯度。

> 组件层禁用“随手改颜色”。只允许改用**语义令牌**，才能持续对比达标。

---

## 8) 与 JS/TS 集成（类型与运行时）

- 通过构建产物 `dist/tokens/ts/index.ts` 提供类型与值：
```ts
import TOKENS from '@acme/tokens'; // 由构建产物导出
type TokenPath = keyof typeof TOKENS; // 或生成枚举/字面量类型
```
- 在图表/动画等 JS 场合，消费 **语义令牌**，避免硬编码：
```ts
const seriesColor = getComputedStyle(document.documentElement).getPropertyValue('--brand-bg').trim();
chart.setColor(seriesColor);
```

---

## 9) 版本化与弃用（治理）

- 在源 JSON 中加入元数据：
```json
"semantic": {
  "light": {
    "fg": {
      "muted": { "$value": "#374151", "$description": "次级文字", "$extensions": { "deprecated": false, "since": "1.2.0" } }
    }
  }
}
```
- **变更日志**：变更令牌 = 影响面最广的“重构”。用 Changesets / SemVer 标识 **breaking**；为被弃用项提供“迁移映射表”。

---

## 10) 反模式与纠偏

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 组件直连基础令牌 | 主题/品牌切换牵一发动全身 | 组件改为消费语义/组件令牌 |
| 在组件写死色值/间距 | 失去全局一致性 | 全量替换为 `var(--*)`，指向 tokens |
| 主题在多个 CSS 文件散落 | 维护困难、相互覆盖 | 用 `@layer settings` + 根 data 属性集中控制 |
| 运行时闪烁 | FOUC/FOIT | `<head>` 同步注入 `data-theme` + `color-scheme` |
| 刻度过密/过多 | 选择困难、样式不一致 | 收敛刻度，限制“自由度”，用 Lint 校验 |

---

## 11) 提交前检查清单

- [ ] 组件未直接写硬编码色值/尺寸，全部使用变量化引用。  
- [ ] 语义令牌完整：`bg/fg/border/focus/status` 至少一套。  
- [ ] 亮/暗主题都有对应别名映射；对比度达标（WCAG AA）。  
- [ ] 构建产物含 CSS/TS（或平台需要的 iOS/Android）多目标输出。  
- [ ] 运行时切换通过 `data-theme / data-density / data-brand`，无闪烁。  
- [ ] 令牌变更附带迁移指引与变更日志。

---

## 12) 练习

1. 把现有项目的“按钮/卡片/输入框”改造成只消费语义令牌；支持亮/暗主题切换。  
2. 为“紧凑模式”定义密度令牌（`space/radius/lineHeight`），用根 data 切换。  
3. 用 Style Dictionary 生成 CSS + TS 产物；在 E2E 里验证主题切换无视觉回归（截图对比）。

---

## 13) 快速模板（可直接拷贝）

**A. HTML 根切换主题/品牌/密度**
```html
<html data-theme="light" data-brand="blue" data-density="comfortable">
  <!-- 切换：data-theme="dark" / data-brand="green" / data-density="compact" -->
</html>
```

**B. 主题切换按钮（最小 JS）**
```js
const root = document.documentElement;
document.getElementById('toggle-theme').addEventListener('click', () => {
  const next = root.dataset.theme === 'dark' ? 'light' : 'dark';
  root.dataset.theme = next;
  localStorage.setItem('theme', next);
});
```

**C. 组件样式消费令牌**
```css
.badge {
  display: inline-flex; align-items: center; gap: .25em;
  padding: 0 .5em; height: 1.75em;
  border-radius: var(--radius-md);
  color: var(--fg-default); background: color-mix(in oklab, var(--brand-bg) 12%, transparent);
  border: 1px solid var(--border-subtle);
}
```

---

## 14) 小结

- 用**基础→语义→组件**三段式，把 UI 决策变成“可版本化的配置”。  
- 用 **Style Dictionary 等管道**产出多平台制品；Web 端配合 `@layer settings` 与根 data 属性做到“一处切换，处处生效”。  
- 把**可访问性与对比度**前置到令牌层，组件层只做“引用”，而非“发明新色”。  
- 收敛刻度、限制自由度、严控硬编码，是长期可维护的关键。
