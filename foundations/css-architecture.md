# 2.2 CSS 架构：BEM / ITCSS / Utility-first

本章目标：用 **BEM 命名**、**ITCSS 分层** 与 **Utility-first（工具类优先）** 的组合拳，构建**可预测的级联**、**可控的特异性**、**可复用的组件**，并保持团队协作时的可演进性。

---

## 0) 架构心法（先有边界，再有自由）

- **语义归位**：结构语义交给 HTML；视觉语义交给 CSS；交互语义交给 JS。
- **特异性预算**：优先 `class`，避免 `id` 与嵌套选择器；把特异性锁在可控范围。
- **分层与收敛**：全局越少越好；把变化集中在组件层与工具类层；跨页规则在“对象/工具层”沉淀。
- **约束即生产力**：命名规则、目录结构、Lint 规则、评审清单，都是“护栏”。

---

## 1) BEM 命名（Block / Element / Modifier）

### 1.1 规则速记

- **Block（块）**：可独立复用的组件（如 `card`、`btn`）。
- **Element（元素）**：隶属于 Block 的组成部分，用双下划线：`card__title`。
- **Modifier（修饰）**：状态或外观变体，用双连字符：`card--featured`；键值式：`card--size-lg`。

> **状态类**建议统一前缀 `is-`/`has-`（如 `is-active`），与 BEM 修饰区分职责（UI 状态 vs 变体）。

### 1.2 示例

```html
<article class="card card--featured">
  <img class="card__cover" src="..." alt="">
  <h3 class="card__title">标题</h3>
  <p class="card__excerpt">摘要……</p>
  <a class="btn btn--primary card__action" href="/post/42">阅读全文</a>
</article>
```

```css
.card { display:grid; gap:.75rem; }
.card--featured { border:2px solid var(--brand-500); }
.card__cover { border-radius:.5rem; aspect-ratio:16/9; object-fit:cover; }
.card__title { font-size:1.125rem; line-height:1.4; }
.card__action { justify-self:start; }
```

### 1.3 组合与复合

- **Block 不彼此嵌套命名**：不要出现 `list__item__title`；改为 `list__item` 内放 `card`。
- **Modifier 不越级**：`card--featured` 影响 `card` 的风格，不直接改 `.card__title`，而是 `.card--featured .card__title{...}`。
- **跨 Block 组合**：通过工具类或布局对象（见 ITCSS 对象层）协调，而非让 Block 知晓彼此。

---

## 2) ITCSS 分层（从低变动到高变动）

**术语映射**：Settings → Tools → Generic → Elements → Objects → Components → Utilities  
**目标**：把“稳定的基础”放底层，频繁变化的样式放上层；自下而上，特异性逐渐升高。

### 2.1 目录结构模板

```
styles/
  0-settings/    # 设计令牌（Design Tokens）：颜色、间距、字体尺⼨
  1-tools/       # 函数、mixin、@custom-media、自定义属性工具
  2-generic/     # Reset/Normalize，基础 Box-sizing、打印规则
  3-elements/    # 原生元素样式：a, h1-h6, p, ul/ol, table, form...
  4-objects/     # 布局/模式对象：.o-stack, .o-cluster, .o-media 等
  5-components/  # 具体组件（BEM）：.card, .btn, .modal, .nav ...
  6-utilities/   # 单一职责工具类：.u-mt-4, .u-text-center, .u-hide ...
```

> 可以用 **Cascade Layers（@layer）** 映射上述层级，强化顺序与覆盖关系。

### 2.2 用 @layer 实现分层

```css
@layer settings, tools, generic, elements, objects, components, utilities;

/* 0-settings */
@layer settings {
  :root {
    --brand-500:#2563eb; --brand-600:#1d4ed8;
    --space-1:.25rem; --space-2:.5rem; --space-3:.75rem; --space-4:1rem;
    --radius-md:.5rem; --text-md:1rem; --text-lg:1.125rem;
  }
}

/* 2-generic */
@layer generic {
  *,*::before,*::after{ box-sizing:border-box; }
  html:focus-within{ scroll-behavior:smooth; }
}

/* 4-objects */
@layer objects {
  .o-stack{ display:flex; flex-direction:column; gap:var(--space-3); }
  .o-cluster{ display:flex; flex-wrap:wrap; gap:var(--space-2); align-items:center; }
}

/* 5-components */
@layer components {
  .card{ display:grid; gap:var(--space-3); }
  .card__cover{ border-radius:var(--radius-md); aspect-ratio:16/9; object-fit:cover; }
  .btn{ display:inline-flex; align-items:center; gap:.5em; padding:.5em 1em; border-radius:.375rem; }
  .btn--primary{ background:var(--brand-600); color:#fff; }
}

/* 6-utilities */
@layer utilities {
  .u-text-center{ text-align:center; }
  .u-mt-4{ margin-top:var(--space-4); }
  .u-hide{ display:none !important; }
}
```

> 经验：**组件层尽量不嵌套超过 2 层**；需要跨组件对齐时，抽到 **对象层**（如 `.o-cluster`）。

---

## 3) Utility-first（工具类优先）

### 3.1 思想与适用场景

- 用原子/工具类快速构建界面；**类名即意图**（`flex`、`gap-4`、`items-center`）。
- **小范围改动**与**视觉实验**极快；但容易“类名爆炸”，因此需要“抽取语义外壳”。

### 3.2 用“工具类 → 语义外壳”降噪

> 在组件稳定后，把工具类“固化”为语义化外壳，避免在每个使用点重复长串类名。

**HTML（过渡期）：**
```html
<button class="inline-flex items-center gap-2 px-4 py-2 rounded-md text-white bg-blue-600 hover:bg-blue-700">
  提交
</button>
```

**CSS Modules 语义外壳（示例）：**
```css
/* button.module.css */
.button{ composes: inline-flex items-center gap-2 px-4 py-2 rounded-md from global; }
.button--primary{ composes: text-white bg-blue-600 hover:bg-blue-700 from global; }
```

**HTML（稳定期）：**
```html
<button class="button button--primary">提交</button>
```

> 若工具链支持 `@apply`（例如某些 utility 框架），也可在组件样式中用 `@apply` 汇聚工具类。

### 3.3 “工具类 + BEM” 共存策略

- **结构 = BEM**，**微调 = 工具类**。  
- 例如：`<article class="card u-mt-4">`；内部间距/对齐用 `.o-` 对象或少量工具类。

---

## 4) 设计令牌（Design Tokens）与主题化

### 4.1 令牌定义（Settings 层）

```css
@layer settings {
  :root {
    /* 色板 */
    --brand-50:#eff6ff; --brand-500:#3b82f6; --brand-600:#2563eb;
    --fg-1:#111827; --fg-2:#374151; --bg-1:#ffffff; --bg-2:#f9fafb;

    /* 尺寸与排版 */
    --space-1:.25rem; --space-2:.5rem; --space-3:.75rem; --space-4:1rem; --space-6:1.5rem;
    --radius-sm:.375rem; --radius-md:.5rem;
    --text-sm:.875rem; --text-md:1rem; --text-lg:1.125rem;
  }

  [data-theme="dark"]{
    --bg-1:#0b1020; --bg-2:#111827; --fg-1:#e5e7eb; --fg-2:#cbd5e1;
    --brand-500:#60a5fa; --brand-600:#3b82f6;
  }
}
```

### 4.2 组件消费令牌（Components 层）

```css
@layer components {
  .card{ background:var(--bg-1); color:var(--fg-1); border:1px solid color-mix(in oklab, var(--fg-2) 10%, transparent); }
  .btn--primary{ background:var(--brand-600); }
  .btn--primary:hover{ background:var(--brand-500); }
}
```

> 主题切换只需切换 `data-theme` 属性；组件不感知主题细节，**只消费令牌**。

---

## 5) React/Vue 等组件化框架的落地

### 5.1 CSS Modules（推荐在中小型团队）

- **优点**：类名局部化、避免全局污染；可与 BEM 结合（对外暴露语义类）。
- **做法**：组件内用 Modules 组织细节；跨组件复用的布局放到 **对象层**（全局）。

```tsx
// Card.tsx（React）
import s from './card.module.css';
export function Card({featured=false, children}) {
  return (
    <article className={`${s.card} ${featured ? s['card--featured'] : ''}`}>
      {children}
    </article>
  );
}
```

### 5.2 全局工具类（Utility-first）与 Modules 组合

- 页面级编排、间距微调用工具类；
- 复杂组件内部用 Modules + BEM，减少“工具类长串”。

---

## 6) 迁移与增量治理（从野生到可控）

1. **梳理 CSS 资产**：按“组件/对象/工具/杂项”打标签；统计选择器特异性与重复率。  
2. **先立分层**：把 Reset/Elements 抽到低层，把通用布局抽到对象层。  
3. **两类改造**：  
   - **热点组件**：BEM 化 + 令牌接入；建立语义外壳。  
   - **页面样式**：逐步替换为工具类或对象类，减少自定义散落样式。  
4. **门禁**：Stylelint + PR 模板 + 视觉回归测试（Playwright/Cypress + percy/snapshots）。  

---

## 7) Lint 与规范（拷走即用）

### 7.1 `.stylelintrc.json`

```json
{
  "extends": ["stylelint-config-standard", "stylelint-config-standard-scss"],
  "rules": {
    "selector-max-id": 0,
    "selector-max-compound-selectors": 3,
    "selector-no-qualifying-type": [true, { "ignore": ["attribute", "class"] }],
    "declaration-no-important": null, 
    "color-function-notation": "modern",
    "property-no-vendor-prefix": true,
    "selector-class-pattern": "^[a-z](?:[a-z0-9-]|__[a-z0-9]+|--[a-z0-9-]+)*$"
  }
}
```

> `selector-class-pattern` 支持 BEM；`selector-max-compound-selectors` 控制嵌套深度。

### 7.2 PostCSS（可选，启用嵌套与自定义媒体）

```js
// postcss.config.cjs
module.exports = {
  plugins: {
    'postcss-nesting': {},
    'postcss-custom-media': { importFrom: [{ customMedia: { '--bp-m': '(min-width: 48rem)' } }] }
  }
};
```

---

## 8) 常见对象（Objects）食谱

```css
/* 纵向栈：元素之间统一间距 */
.o-stack{ display:flex; flex-direction:column; gap:var(--space-3); }
/* 聚簇：水平对齐 + 自动换行 */
.o-cluster{ display:flex; flex-wrap:wrap; gap:var(--space-2); align-items:center; }
/* 媒体对象：左图右文 */
.o-media{ display:flex; gap:var(--space-3); align-items:flex-start; }
.o-media__img{ flex:0 0 auto; }
.o-media__body{ flex:1 1 auto; min-width:0; }
```

---

## 9) 反模式与纠偏

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 深嵌套选择器 `.a .b .c .d span` | 脆弱、覆盖链复杂 | 扁平化到 BEM：`.block__el`；用对象/工具类抽共性 |
| 滥用 `!important` | 覆盖战争 | 用 @layer/特异性预算；仅限工具类紧急兜底 |
| 全局工具类横飞 | 语义缺失、难复用 | 稳定后抽语义外壳：`.button--primary` |
| 组件互知彼此结构 | 紧耦合 | 用对象层协调布局，不让组件“跨墙” |
| 令牌缺失 | 难统一主题/品牌 | 先补 `:root` 令牌，再改组件消费令牌 |

---

## 10) 提交前检查清单

- [ ] 类名符合 BEM/状态前缀；无多级嵌套与 `id` 选择器。  
- [ ] 新样式落到正确层（Objects/Components/Utilities），不塞进“杂项”。  
- [ ] 组件只消费令牌，不直接写硬编码色值与间距。  
- [ ] 工具类用于微调；稳定后抽成语义外壳。  
- [ ] Stylelint 通过；视觉回归关键路径通过。  

---

## 11) 练习

1. 选一个页面，把“全局规则/组件规则/微调规则”按 ITCSS 分层重组。  
2. 把一个“工具类长串”的按钮沉淀为 `button` 组件（BEM），并用令牌驱动配色。  
3. 用 **对象层 + BEM** 重构“卡片列表”，让不同模块的标题/摘要/操作按钮对齐到同一列。

---
