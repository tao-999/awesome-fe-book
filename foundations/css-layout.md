# 2.1 Flex/Grid/Container Queries

本章目标：掌握 **Flexbox（轴向排布）**、**CSS Grid（二维栅格）** 与 **Container Queries（容器查询）** 的使用边界与组合套路，在**组件层**实现自适应，而非一味依赖全局媒体查询。

---

## 0) 选型心法（一句话判断）

- **单轴、一行自适应、内容驱动** → 用 **Flex**  
- **二维布局、对齐与轨道（Tracks）控制** → 用 **Grid**  
- **同一组件在不同父容器宽度下切换样式** → **Container Queries**（容器查询）  
- **能在组件层解决的响应式，不要上媒体查询**；媒体查询只兜全局断点与排版大纲。

---

## 1) Flex 速记卡

### 1.1 最常用属性组合
```css
/* 行内导航/标签栏 */
.nav {
  display: flex;
  gap: .75rem;
  align-items: center;          /* 交叉轴对齐 */
  justify-content: space-between; /* 主轴分布 */
  flex-wrap: wrap;              /* 小屏换行 */
}

/* “媒体对象”模式：左图右文 */
.media { display: flex; gap: 1rem; align-items: start; }
.media__img { flex: 0 0 auto; }
.media__body { flex: 1 1 auto; min-width: 0; } /* 防止文本溢出 */
```

### 1.2 常见坑
- **min-width: 0**：子项含长文本/代码时记得清零，避免溢出不收缩。  
- **gap** 优于 margin 撑开；避免“最后一个元素要去掉间距”的判断。  
- `flex: 1` 等价 `1 1 0%`，而 `flex: auto` 是 `1 1 auto`；有时会影响初始尺寸参与分配。

---

## 2) Grid 速记卡

### 2.1 一行写出“自适应卡片墙”
```css
.cards {
  display: grid;
  gap: 1rem;
  grid-template-columns: repeat(auto-fit, minmax(16rem, 1fr));
}
```
- `auto-fit` 会“折叠”多余轨；`minmax(16rem, 1fr)` 让卡最小 16rem，空间富余则均分。

### 2.2 圣杯/双栏/三栏
```css
.layout {
  display: grid;
  grid-template-columns: 1fr min(72ch, 100%) 1fr; /* 中心列最大 72ch */
}
.layout > * { grid-column: 2; }   /* 默认放中栏 */
.layout--full-bleed { grid-column: 1 / -1; } /* 横跨全宽的横幅 */
```

### 2.3 子网格（Subgrid）做内容对齐
```css
.page {
  display: grid;
  grid-template-columns: 16rem 1fr;
  gap: 1rem;
}
.article {
  display: grid;
  grid-template-columns: subgrid; /* 继承父网格的列轨 */
  grid-column: 1 / -1;
}
```
> 子网格让“标题/图文/元信息”在不同模块里也能对齐到同一列。

---

## 3) Container Queries（容器查询）

组件在**父容器尺寸**变化时改变样式（而非全局视口）。核心三步：

### 3.1 标记容器
```css
.card-list { 
  container-type: inline-size;   /* 只关心行内尺寸（宽度） */
  container-name: cards;         /* 可选命名，便于选择特定容器 */
}
```

### 3.2 写查询
```css
/* 未命名：最近的有 container-type 的祖先即为容器 */
@container (min-width: 36rem) {
  .card { display: grid; grid-template-columns: 10rem 1fr; gap: 1rem; }
}

/* 命名容器：只对 name=cards 生效 */
@container cards (min-width: 64rem) {
  .card { grid-template-columns: 14rem 1fr; }
}
```

### 3.3 容器查询单位（CQ Units）
```css
.card { padding: 2cqi; } /* 2% of container inline size（相对容器宽度） */
```
- `cqw/cqh/cqi/cqb/cqmin/cqmax`：相对容器宽/高/内联/块维度，做“组件内流体布局”。

> 与媒体查询搭配思路：**大纲结构**（导航从顶部到抽屉）用媒体查询；**组件内布局**（卡片单/双列、字体大小、间距）交给容器查询。

---

## 4) 组件级响应式：三连例

### 4.1 “卡片在窄容器变纵向，在宽容器变左右布局”
```html
<ul class="card-list">
  <li class="card">
    <img class="card__cover" src="..." alt="">
    <div class="card__body">
      <h3 class="card__title">标题</h3>
      <p class="card__desc">摘要……</p>
    </div>
  </li>
  <!-- ... -->
</ul>
```

```css
.card-list { container-type: inline-size; }

.card {
  display: grid;
  gap: .75rem;
  grid-template-columns: 1fr;     /* 窄时纵向 */
  align-items: start;
}
.card__cover { aspect-ratio: 16 / 9; object-fit: cover; border-radius: .5rem; }

@container (min-width: 36rem) {
  .card { grid-template-columns: 12rem 1fr; } /* 宽时左右 */
}
```

### 4.2 “工具栏：小容器堆叠，大容器一行并自动留白”
```css
.toolbar { 
  display: flex; gap: .5rem; flex-wrap: wrap; 
  container-type: inline-size;
}
@container (min-width: 28rem) {
  .toolbar { justify-content: space-between; align-items: center; }
}
```

### 4.3 “卡片网格：列数跟随容器，不受页面其它栏影响”
```css
.gallery { 
  display: grid; gap: 1rem; container-type: inline-size; 
  grid-template-columns: repeat(auto-fit, minmax(12rem, 1fr));
}
@container (min-width: 60rem) {
  .gallery { grid-template-columns: repeat(auto-fit, minmax(16rem, 1fr)); }
}
```

---

## 5) Grid × Flex × CQ 的组合套路

- **外层 Grid 定轨、内层 Flex 排轴**：Grid 定宏观列，对齐精度高；卡片内部用 Flex 布小部件。  
- **子网格做“跨模块对齐”**：标题、按钮、计价等跨组件竖线对齐。  
- **容器查询控制“组件形态”**：单列 → 双列、图文上下 → 左右、信息密度从紧到疏。  
- **流体单位**：`clamp()` + `cqi` 做组件内排版比例：
  ```css
  .card__title { font-size: clamp(1rem, 2cqi, 1.5rem); }
  .card { padding: clamp(.75rem, 1cqi, 1.25rem); }
  ```

---

## 6) 常见版式食谱（拷走即用）

### 6.1 顶部导航自适应（Logo + 菜单 + 右侧动作）
```css
.header {
  display: grid;
  grid-template-columns: auto 1fr auto;
  align-items: center; gap: .75rem;
}
.nav { display: flex; gap: .75rem; flex-wrap: wrap; justify-content: center; }
.actions { display: flex; gap: .5rem; justify-content: end; }
```

### 6.2 双栏文章（中栏 72ch，侧栏贴网格）
```css
.page {
  display: grid; gap: 2rem;
  grid-template-columns: 1fr min(72ch, 100%) 16rem;
}
.page > main { grid-column: 2; }
.page > aside { grid-column: 3; align-self: start; position: sticky; top: 2rem; }
@media (max-width: 64rem) { .page { grid-template-columns: 1fr; } .page > * { grid-column: 1; } }
```

### 6.3 数据表头与行对齐（Subgrid）
```css
.table { 
  display: grid; grid-template-columns: 2fr 1fr 1fr 1fr; gap: .5rem; 
}
.thead, .tbody { display: grid; grid-template-columns: subgrid; }
.thead { font-weight: 600; }
.trow { display: contents; } /* 每行的单元格参与父网格 */
```

---

## 7) 性能与可维护性

- **少量但明确的断点**：用语义化常量集中管理：
  ```css
  @custom-media --bp-m (min-width: 48rem);
  @media (--bp-m) { /* ... */ }
  ```
- **优先容器查询**：组件内部自适应不会“误伤”其它布局，减少断点联动复杂度。  
- **避免魔法数**：`minmax()`/`clamp()`/`aspect-ratio` 优于硬编码宽高。  
- **Debug 小技巧**：
  ```css
  * { outline: 1px dashed hsl(210 10% 70% / .5); }
  ```

---

## 8) 反模式与纠偏

| 反模式 | 问题 | 纠偏 |
|---|---|---|
| 纯 Flex 拼二维网格 | 复杂嵌套、对齐困难 | 用 Grid 定轨，内部用 Flex |
| 只用媒体查询 | 组件在不同父容器下不可控 | 给组件父级加 `container-type`，改用 CQ |
| 固定像素断点 | 设备碎片化失效 | 用 `em/rem` 或语义断点、`clamp()` |
| Grid 无最小宽度 | 卡片过窄破版 | `minmax(16rem, 1fr)` 设下限 |
| 卡片文字溢出 | 默认 min-content 影响收缩 | `min-width: 0; overflow-wrap: anywhere;` |

---

## 9) 提交前检查清单

- [ ] 单轴列表/工具栏用 Flex，二维区块用 Grid。  
- [ ] 组件父级声明 `container-type`，组件内样式首选容器查询。  
- [ ] 网格列使用 `repeat(auto-fit, minmax())`，设置合理最小宽度。  
- [ ] 关键元素 `min-width: 0` 防止溢出；图片设置 `aspect-ratio` 与 `object-fit`。  
- [ ] 排版采用 `clamp()` 或 CQ 单位实现流体比例。  
- [ ] 媒体查询仅用于全局版式切换或大纲层面改变。

---

## 10) 练习

1. 把“产品卡片”改造成 **容器查询驱动**：窄容器竖排，宽容器图文左右。  
2. 用 **Grid + Subgrid** 对齐博客列表里“标题/日期/标签”，让不同模块精确对齐到同一列。  
3. 将一个媒体查询驱动的卡片墙改成 `repeat(auto-fit, minmax())`，并用 `clamp()` 做流体排版。

---
