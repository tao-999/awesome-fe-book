# 16.1 LCP / CLS / INP / TBT 优化 ⚡

目标：把 **Core Web Vitals** 的四大指标调到绿区，并在真实用户数据（RUM）里稳定达标。  
阈值（以“好”为目标线，Google 口径）：
- **LCP**（最大内容绘制） ≤ **2.5s**
- **CLS**（累计位移） ≤ **0.1**
- **INP**（交互到下一次绘制） ≤ **200ms**
- **TBT**（总阻塞时间，实验室指标） ≤ **200ms**

> 方法论：**先找“首屏瓶颈”（LCP/TTFB）→ 再稳布局（CLS）→ 再抹平交互长尾（INP/TBT）**。实验室（Lighthouse/WebPageTest）定位问题，现场（RUM/CrUX）验证成效。

---

## 1) LCP：最大内容绘制 🖼️

LCP 通常由 **首屏英雄图/标题/视频海报**触发。公式近似：**TTFB + 资源加载 + 首次绘制**。  
**优化清单（按收益排序）**：
1. **降 TTFB**：CDN/边缘渲染、静态化(SSG/ISR)、HTTP/2/3、压缩(gzip/br)、数据库走只读副本。
2. **给 LCP 资源最高优先级**  
   - HTML 内直出 `<img ... fetchpriority="high">`（或 `<link rel="preload" as="image">`）。  
   - CSS 首屏内联（Critical CSS 5–14KB），剩余 `media="print"` + `onload`/`rel=preload`。
3. **图片优化**：响应式 `srcset/sizes`、`width/height/aspect-ratio`、`loading="eager"`（仅 LCP）、`decoding="async"`、`AVIF/WEBP`。  
4. **字体策略**：首屏文本用系统字体或 `font-display: swap` + 子集化（避免 FOIT 抵消 LCP）。  
5. **减少阻塞脚本**：移除不必要 JS；其余 `defer`/`async`，首屏路由做**代码分割**；三方脚本延后。  
6. **网络暖身**：关键域名 `<link rel="preconnect" href="https://…">`；跨源图片可 `<link rel="dns-prefetch">`。  
7. **优先流式**：SSR/边缘 Streaming，尽早输出包含 LCP DOM 的 HTML 片段。

**示例：英雄图（响应式 + 高优先级）**
```html
<link rel="preconnect" href="https://cdn.example.com">
<img
  src="https://cdn.example.com/hero-1200.avif"
  srcset="
    https://cdn.example.com/hero-800.avif 800w,
    https://cdn.example.com/hero-1200.avif 1200w,
    https://cdn.example.com/hero-1600.avif 1600w"
  sizes="(max-width: 800px) 100vw, 1200px"
  width="1200" height="630" fetchpriority="high" decoding="async" alt="Hero">
```

---

## 2) CLS：累计布局位移 🧱

**根因**：尺寸未保留、异步内容插队、字体换行差异。  
**优化清单**：
- **保留空间**：所有媒体元素提供 `width/height` 或 `aspect-ratio`。广告/骨架/占位有**固定槽位**。  
- **字体稳定**：`font-display: swap` + `size-adjust`（或在本地预估 metrics），避免加载后回流。  
- **UI 改动只用 transform/opacity**：动画尽量用 `transform/opacity`，避免 `top/left/width/height`。  
- **不在已有内容上方插入**：通知条/懒组件在**容器内**展开，不要把页面向下推。  
- **图片比例容器**：
```css
.card-thumb { aspect-ratio: 4 / 3; object-fit: cover; }
```

---

## 3) INP：交互到下一次绘制 🎯

INP 代表**一次完整交互**（点击/输入/拖动）的**最慢体验**。  
**三段优化**：输入→处理→绘制。

- **输入阶段**：事件监听用 `passive`（滚动相关），降低主线程争用；关闭多余监听（事件委托）。  
- **处理阶段**：拆分长任务（< 50ms），把重活丢给 Web Worker；避免同步布局抖动（读写分离）。  
- **绘制阶段**：优先 `transform/opacity`，启用 `will-change`（谨慎）。对大更新做**虚拟列表/分页渲染**。

**将重活切片 & 让用户先得到响应**
```ts
// 伪代码：把重计算切片，保持每片 < 16ms
function chunkedWork(items: any[]) {
  let i = 0;
  function next() {
    const deadline = performance.now() + 10;
    while (i < items.length && performance.now() < deadline) {
      heavyCompute(items[i++]);
    }
    if (i < items.length) requestIdleCallback(next); // 或 setTimeout(next, 0)
  }
  next();
}
```

---

## 4) TBT：总阻塞时间 🧱⏱️（实验室代理 INP）

TBT ≈ 所有 **>50ms Long Task 超出的部分之和**（加载阶段）。  
**优化清单**：
- **少而精的 JS**：删依赖、按路由拆包、动态导入；图片/样式交给平台组件（如 next/image）。  
- **脚本延后**：`defer` 默认，非必要三方脚本延迟到 `requestIdleCallback`。  
- **Hydration 最小化**：岛屿架构/Server Components；组件粒度更细。  
- **预编译**：SSR/边缘预渲染 HTML，客户端只做交互绑定。  
- **避免同步阻塞**：大 JSON 解压/解析放 Worker；避免 `localStorage` 读写阻塞主线程。

---

## 5) 诊断：实验室与现场两条腿走路 🔬

- **实验室**：Lighthouse、WebPageTest、`Performance` 面板 + **Long Task**（>50ms）；`Coverage` 看未用代码。  
- **现场（RUM）**：使用 `PerformanceObserver` 采集 **LCP/CLS/INP**，上报到你的埋点系统；按**设备/网络/地域**分桶。  
- **追长尾**：记录**最慢 1% 的交互**（INP），抓取 `event.type/selector/stack/longtasks`。

**采集 INP（简化版）**
```js
let worst = 0;
new PerformanceObserver((list) => {
  for (const e of list.getEntries()) worst = Math.max(worst, e.duration);
  // 页面卸载前上报 worst 作为 INP 近似
}).observe({ type: 'event', buffered: true, durationThreshold: 40 });
```

---

## 6) 各框架/平台的落地手册 🧰

- **Next.js**：`next/image`（自动格式/尺寸/占位）、`next/font`（子集化 + 自动 preload）、`fetch` 的 `revalidate`/`cache` 控制首屏、`app/` 路由用 **RSC** 降 JS。  
- **Nuxt**：`@nuxt/image`、`routeRules` 配 ISR、`nitro` 预渲染，组件懒加载。  
- **SvelteKit/Astro**：岛屿/部分水合；`client:visible/idle`；`.astro` 输出静态结构。  
- **CDN/边缘**：给 LCP 资源单独缓存/就近下发；用 Early Hints/Preload 推进下载队列。

---

## 7) 核心反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 未标尺寸的图片/广告 | CLS 飙升 | `width/height` 或 `aspect-ratio` + 占位容器 |
| 全局巨包 | TBT/INP 高 | 路由级拆分、动态导入、删除死代码 |
| 阻塞式三方脚本 | 首屏慢、TBT 高 | `defer`/延迟加载/同意管理，能不用就不用 |
| 动画改 layout 属性 | 帧率抖 | 用 `transform/opacity`，避免回流 |
| FOIT（首屏字体“隐形”） | LCP 延迟、CLS | `font-display: swap` + 子集化 + `size-adjust` |
| 客户端渲染一切 | 首屏慢 | SSR/SSG/ISR + 流式 + 优先直出 LCP DOM |
| 读写混用导致布局抖动 | INP 受损 | 读写分离：先读再写，或 `requestAnimationFrame` 调度 |

---

## 8) 提交前检查清单 ✅

- [ ] 首屏 **LCP 资源**已 `fetchpriority="high"`/`preload`，并使用响应式图片与现代格式。  
- [ ] 所有媒体/广告/骨架**有固定尺寸或比例**；没有在顶部插入新内容。  
- [ ] 首屏 CSS 内联；非关键 JS **defer/懒加载**；路由级**代码分割**。  
- [ ] 长任务拆分（<50ms）；重计算移入 **Web Worker**；避免同步存储。  
- [ ] 框架层启用 **SSR/SSG/Streaming/岛屿**；减少水合范围。  
- [ ] RUM 采集 **LCP/CLS/INP**，并按设备/网络分桶监控；实验室基线 TBT ≤ 200ms。

---

## 9) 练习 🏋️

1. 将首页英雄图改成 **AVIF + fetchpriority**，测 LCP 改善幅度。  
2. 为所有图片/广告补充 **尺寸/比例占位**，对比 CLS。  
3. 把一个 300KB 的依赖做 **动态导入**，观察 TBT/INP 的 P95 变化；将计算搬进 **Worker** 再测。  

---

**小结**：把 LCP 当“首屏速度表”、CLS 当“布局稳定器”、INP/TBT 当“交互顺滑度”。用**小而快的首屏**、**可预测的布局**、**切片与并行的执行流**，四项指标自然变绿。🟢
