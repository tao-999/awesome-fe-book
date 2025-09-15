# 5.3 动画与交互：Framer Motion / GSAP 🌀

本章目标：用 **Framer Motion（React 声明式动画）** 与 **GSAP（命令式时间轴与滚动神器）**，在“**性能**、**可访问性**、**可维护性**”三线齐飞的前提下，落地常见交互：**页面转场、共享元素、列表重排、滚动触发、微交互、模态与手势**。

---

## 0) 选型心法（先拍板）

- **React 组件内的状态驱动、共享元素、手势** → **Framer Motion**（声明式、变体 `variants`、`layoutId` 超顺手）。
- **复杂时间线、跨组件/非 React DOM、滚动联动、像素级掌控** → **GSAP**（时间轴 `Timeline`、`ScrollTrigger` 一把梭）。
- **页面=路由 + 动画**：路由栈与数据见 5.1；动画层建议**最小侵入**——UI 状态驱动动画，而非动画驱动状态。

---

## 1) Framer Motion 快速上手

```bash
pnpm add framer-motion
```

### 1.1 最小微交互
```tsx
import { motion } from 'framer-motion';

export function FancyButton() {
  return (
    <motion.button
      initial={{ scale: 1 }}
      whileHover={{ scale: 1.05 }}
      whileTap={{ scale: 0.97 }}
      transition={{ type: 'spring', stiffness: 400, damping: 20 }}
      className="btn"
    >
      点我
    </motion.button>
  );
}
```

### 1.2 变体（Variants）与父子协同
```tsx
const list = {
  hidden: { opacity: 0 },
  show: { opacity: 1, transition: { staggerChildren: 0.06 } }
};
const item = {
  hidden: { opacity: 0, y: 6 },
  show:   { opacity: 1, y: 0 }
};

<motion.ul variants={list} initial="hidden" animate="show">
  {data.map(x =>
    <motion.li key={x.id} variants={item}>{x.name}</motion.li>
  )}
</motion.ul>
```

### 1.3 出场/入场（`AnimatePresence`）
```tsx
import { AnimatePresence, motion } from 'framer-motion';

<AnimatePresence initial={false}>
  {open && (
    <motion.aside
      key="panel"
      initial={{ x: 24, opacity: 0 }}
      animate={{ x: 0,  opacity: 1 }}
      exit={{ x: 24, opacity: 0 }}
      transition={{ duration: .18 }}
    />
  )}
</AnimatePresence>
```

---

## 2) 布局动画与共享元素（Layout / FLIP）

### 2.1 自动布局动画（`layout`）
```tsx
<motion.div layout className={expanded ? 'card card--xl' : 'card'} />
```
- 让 **布局变化**（尺寸/位置）自动平滑过渡；慎用深嵌套，必要时给关键节点加 `layout`。

### 2.2 共享元素（`layoutId`）做页面转场
```tsx
// 列表页
<motion.img layoutId={`photo-${id}`} src={thumb} onClick={()=>nav(`/p/${id}`)} />

// 详情页
<motion.img layoutId={`photo-${id}`} src={full} />
```
> 两处元素 `layoutId` 相同，切换路由时自动跨页面变形过渡（Frame-Match 魔术）。

---

## 3) 手势：拖拽/滚动/视差

```tsx
// 拖拽卡片（边界 + 惯性）
<motion.div
  drag
  dragConstraints={{ left: -50, right: 50, top: -20, bottom: 20 }}
  dragElastic={0.2}
  whileDrag={{ scale: 1.03 }}
/>
```

### 3.1 useScroll + useTransform（滚动驱动）
```tsx
import { useScroll, useTransform, motion } from 'framer-motion';

export function ParallaxHeader(){
  const { scrollYProgress } = useScroll();
  const y  = useTransform(scrollYProgress, [0, 1], [0, -120]);
  const op = useTransform(scrollYProgress, [0, 1], [1, 0]);
  return <motion.h1 style={{ y, opacity: op }}>前端宇宙</motion.h1>;
}
```

### 3.2 视口进入（一次性显现）
```tsx
<motion.div
  initial={{ opacity: 0, y: 12 }}
  whileInView={{ opacity: 1, y: 0 }}
  viewport={{ once: true, margin: '-10% 0px -10% 0px' }}
/>
```

---

## 4) 动画学参数速记（Motion）

- **插值**：`spring`（弹性，`stiffness`/`damping`/`mass`）、`tween`（`ease`、`duration`）、`inertia`（基于速度衰减）。  
- **关键帧**：`animate={{ x: [0, 50, 0] }}` + `times` 控制节奏。  
- **优先 transform**（`x/y/scale/rotate`）而非 `top/left`；避免频繁 `box-shadow`/`filter: blur()`。

---

## 5) GSAP：时间轴与滚动触发

```bash
pnpm add gsap
```

### 5.1 基础语法
```ts
import gsap from 'gsap';

// to / from / fromTo
gsap.to('.box', { x: 200, duration: .6, ease: 'power2.out' });

const tl = gsap.timeline({ defaults: { ease: 'power3.out' } });
tl.from('.title', { y: 30, opacity: 0 })
  .from('.desc',  { y: 20, opacity: 0 }, '<0.06')
  .from('.cta',   { y: 10, opacity: 0 }, '<');
```

### 5.2 ScrollTrigger（滚动就是进度条）
```ts
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);

gsap.to('.hero', {
  yPercent: -20,
  scrollTrigger: {
    trigger: '.section-hero',
    start: 'top top',
    end: 'bottom top',
    scrub: true
  }
});
```

### 5.3 React 集成（`useLayoutEffect + context`）
```tsx
import { useLayoutEffect, useRef } from 'react';
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);

export function Hero(){
  const root = useRef<HTMLDivElement>(null);
  useLayoutEffect(() => {
    const ctx = gsap.context(() => {
      gsap.from('.hero-title', { y: 24, opacity: 0, duration: .4 });
      gsap.to('.hero-img', {
        yPercent: -15,
        scrollTrigger: { trigger: root.current, start: 'top top', end: 'bottom top', scrub: true }
      });
    }, root);
    return () => ctx.revert(); // 组件卸载时撤销所有动画
  }, []);
  return <section ref={root}><h1 className="hero-title" /><img className="hero-img" /></section>;
}
```

---

## 6) Framer vs GSAP：怎么搭？

- **组件内微交互/共享元素/手势** → Framer。  
- **跨区块时间编排/滚动联动/精细时间线** → GSAP。  
- 组合姿势：**Framer 管元素状态**，**GSAP 管全局时间轴**。避免双管同一属性；若必须，**拆分维度**（如 Framer 管 `opacity`，GSAP 管 `x`）。

---

## 7) 可访问性与“减弱动效”♿

### 7.1 尊重系统设置
```tsx
// Framer：统一关闭或降级
const prefersReduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
<motion.div animate={prefersReduced ? { opacity: 1 } : { opacity: [0,1] }} />;
```

```css
/* 纯 CSS 开关（兜底） */
@media (prefers-reduced-motion: reduce) {
  * { animation-duration: .001ms !important; animation-iteration-count: 1 !important; transition-duration: .001ms !important; }
}
```

### 7.2 无障碍细节
- 动效用于**强化反馈**（状态变化、层级关系），避免无意义炫技。  
- 关闭/跳过入口：长流程滚动动效提供“跳过动画”。  
- 模态出现时**先聚焦**内容（见 4.2 的焦点管理），动画不阻碍键盘导航。

---

## 8) 性能三板斧 🏎️

- **合成层**：只动 `transform/opacity`；大元素加 `will-change: transform`（*短时*使用）。  
- **避免强制同步布局**：不要在动画回调里读写同一元素布局；GSAP/Framer 已做 raf 节流，逻辑层也要自律。  
- **图片/文字渲染**：SVG 复杂路径尽量栅格化；文字动效避免过度 `filter`/模糊；滚动视差限制在移动端低端机。

---

## 9) 常见食谱（拷走即用）

### 9.1 页面的共享元素转场（Framer）
```tsx
// Layout.tsx
import { AnimatePresence } from 'framer-motion';
export function Layout({ location, children }: any){
  return (
    <AnimatePresence mode="wait">
      <motion.main key={location.pathname} initial={{ opacity: 0 }} animate={{ opacity: 1 }} exit={{ opacity: 0 }} transition={{ duration: .15 }}>
        {children}
      </motion.main>
    </AnimatePresence>
  );
}
```

### 9.2 列表重排（Framer Reorder）
```tsx
import { Reorder, motion } from 'framer-motion';
export function Sortable({ items, setItems }:{ items:string[]; setItems:(v:string[])=>void }){
  return (
    <Reorder.Group axis="y" values={items} onReorder={setItems}>
      {items.map(x => (
        <Reorder.Item key={x} value={x} as={motion.li} whileDrag={{ scale: 1.02 }}>
          {x}
        </Reorder.Item>
      ))}
    </Reorder.Group>
  );
}
```

### 9.3 模态弹层（进出+背景虚化）
```tsx
<AnimatePresence>
  {open && (
    <>
      <motion.div className="backdrop"
        initial={{ opacity: 0 }} animate={{ opacity: .5 }} exit={{ opacity: 0 }} />
      <motion.dialog
        initial={{ y: 24, opacity: 0, scale: .98 }}
        animate={{ y: 0,  opacity: 1, scale: 1 }}
        exit={{ y: 12, opacity: 0, scale: .98 }}
        transition={{ type: 'spring', stiffness: 420, damping: 30 }}
      />
    </>
  )}
</AnimatePresence>
```

### 9.4 滚动显现（GSAP + ScrollTrigger）
```ts
gsap.utils.toArray<HTMLElement>('.reveal').forEach((el) => {
  gsap.from(el, {
    y: 20, opacity: 0, duration: .4,
    scrollTrigger: { trigger: el, start: 'top 80%', toggleActions: 'play none none reverse' }
  });
});
```

---

## 10) 在路由/数据流中的位置（协同）

- **页面数据就绪** → 再开始昂贵动画（避免“骨架 + 动画打架”）。  
- 路由切换中断时取消动画：Framer 由组件卸载自动结束；GSAP 用 `ctx.revert()` 清干净。  
- **React Query/SWR** 的加载态配微动效（例如淡入/骨架），但**不要**在每次 refetch 都做大动画。

---

## 11) 测试与可预测性

- 单测/E2E 中将动效**降级**：把 `prefers-reduced-motion` 设为 `reduce` 或将过渡时间置零。  
- 视觉回归（见 4.1）：冻结时间/随机，动画结束后截屏；或对中间帧用**关键帧快照**。

---

## 12) 反模式与纠偏

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 动画驱动状态 | 竞态、难维护 | 由**状态**驱动动画（声明式），动画只“跟随” |
| top/left 改位移 | 抖动、重排多 | 用 `transform: translate` |
| 滥用滤镜/阴影 | GPU/文字渲染吃紧 | 用不透明度与位置替代；阴影用伪元素静态图 |
| 动画阻断交互 | 键盘无法进入/退出 | `pointer-events` 控制 + 先聚焦可交互元素 |
| 未尊重 reduced-motion | 用户恶心眩晕 | 提供关闭/降级路径（见 §7） |

---

## 13) 提交前检查清单 ✅

- [ ] 仅动画 `transform/opacity`，重排最小化；必要时 `will-change` 短期启用。  
- [ ] Framer：变体合理、`AnimatePresence` 出场收尾；共享元素用 `layoutId`。  
- [ ] GSAP：`useLayoutEffect + ctx.revert()` 清理，`ScrollTrigger` 限制在需要的段落。  
- [ ] 尊重 `(prefers-reduced-motion)`；为用户提供“减少动效”开关。  
- [ ] 路由/数据加载与动画协同，无竞态（卸载即清）。  
- [ ] 在 Storybook / E2E 上有**稳定**的动效用例（冻结时间/随机）。

---

## 14) 练习 🏋️

1. 用 **`layoutId`** 实现“卡片 → 详情”的共享元素转场，并在路由切换中复用。  
2. 用 **`ScrollTrigger`** 完成三段式滚动叙事：标题视差、插图钉住（pin）、数据图表进度条联动。  
3. 将项目的“入场大动画”在 `prefers-reduced-motion` 下自动降级为**淡入**，并写一个 Storybook `play` 断言。

---
