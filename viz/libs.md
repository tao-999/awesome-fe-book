# 20.1 D3 / ECharts / AntV 体系 🎯📊

目标：用一章把三大可视化阵营的**选型、心智模型、集成套路、性能与可维护**说清楚，给到能直接抄的代码与流程。口号：**“图形语法定规则，声明配置出成图，工程能力保交付。”**

---

## 0) TL;DR（先抄三条）⚡
1. **产品快速上线 / 大而全组件** → 选 **ECharts**；  
   **高自由度 / 定制交互 / 研究型** → 选 **D3**；  
   **语法化 + 生态齐全 + 商业级工程化** → 选 **AntV（G2/Plot/G6/L7/S2）**。
2. React/Vue 项目优先用**官方/社区封装**（`@ant-design/plots`、`echarts-for-react`、`vue-echarts`），**状态只进不出**（数据→图表），交互事件再回流到应用层。
3. 性能守则：**SVG < Canvas < WebGL**；>10k 点用 Canvas/下采样，>100k 点上 WebGL/瓦片/抽稀；长动画进 **OffscreenCanvas + Worker**。

---

## 1) 三家快速画像 🧭
| 维度 | D3 | ECharts | AntV |
|---|---|---|---|
| 心智模型 | **底层工具集**（数据→DOM/Canvas，行为自己拼） | **配置驱动**（一个 `option` 出图） | **图形语法 / 语义层**（G2/Plot），再向图/地理/表扩展 |
| 出图效率 | 中（自由度高，成本也高） | 高（开箱即用 80% 场景） | 中高（Plot/G2 快，深定制也能打） |
| 自定义能力 | 极高（编译器级别自由） | 中（可写自定义系列/插件） | 高（自定义标记/视图/主题） |
| 渲染 | SVG/Canvas | Canvas/部分SVG/WebGL(扩展) | Canvas/SVG/WebGL(子库) |
| 生态 | d3-* 模块海，自由拼装 | 地图、GL、数据集、富交互组件全 | G2(统计图) / Plot(语法糖) / G6(关系图) / L7(地图) / S2(透视表) |
| 学习曲线 | 陡峭（但一劳永逸） | 平缓 | 适中 |

---

## 2) 选型清单（场景→方案）🧪
- **仪表盘 / 监控 / BI**：ECharts / AntV Plot（上手快、主题统一、联动方便）  
- **论文复现 / 可解释可视化 / 创意叙事**：D3（自定义标记、刻度、过渡）  
- **地图/空间可视**：AntV L7 / ECharts GL（瓦片、热力、矢量）  
- **关系网络**：AntV G6（布局丰富）  
- **明细+指标联动**：AntV S2（轻量透视表）  
- **超大点线面**（>100k 点）：ECharts（`large: true`）/ AntV + GPGPU / WebGL 专项

---

## 3) 统一心智模型（你在“告诉”系统什么）🧠
- **D3（Imperative）**：我来**怎么画**（尺度/轴/布局/过渡）；像写渲染引擎。  
- **ECharts（Declarative）**：我来**画什么**（类型+数据+样式+交互），引擎自己排布。  
- **AntV（Grammar of Graphics）**：我来**表达关系**（数据→编码：`x/y/color/shape/size`），引擎据“语法”出图。

> 结论：**先用声明式拿结果**，再用 D3 做“边界突破”。

---

## 4) 极速上手三连发（可直接抄）🚀

### 4.1 D3：最小柱状图 + 过渡
```js
import * as d3 from "d3";

const data = [12, 36, 28, 50, 18];
const w = 400, h = 160, m = {l:30, r:10, t:10, b:24};

const x = d3.scaleBand().domain(d3.range(data.length)).range([m.l, w-m.r]).padding(0.2);
const y = d3.scaleLinear().domain([0, d3.max(data)]).range([h-m.b, m.t]);

const svg = d3.select("#app").append("svg").attr("width", w).attr("height", h);
svg.append("g").attr("transform", `translate(0,${h-m.b})`).call(d3.axisBottom(x).tickFormat(i=>`#${i+1}`));
svg.append("g").attr("transform", `translate(${m.l},0)`).call(d3.axisLeft(y).ticks(5));

svg.selectAll("rect").data(data).join("rect")
  .attr("x", (_,i)=>x(i)).attr("width", x.bandwidth()).attr("y", y(0)).attr("height", 0)
  .transition().duration(600).attr("y", d=>y(d)).attr("height", d=>y(0)-y(d));
```

### 4.2 ECharts：配置出图（带缩放+提示）
```js
import * as echarts from "echarts";

const chart = echarts.init(document.getElementById("app"));
chart.setOption({
  tooltip: { trigger: "axis" },
  dataZoom: [{ type: "inside" }, { type: "slider" }],
  xAxis: { type: "category", data: ["Mon","Tue","Wed","Thu","Fri"] },
  yAxis: { type: "value" },
  series: [{ type: "bar", data: [120, 200, 150, 80, 70], emphasis: { focus: "series" }, animationDuration: 500 }]
});
window.addEventListener("resize", () => chart.resize());
```

### 4.3 AntV Plot（G2 语法糖）：一行一个图
```js
import { Column } from "@ant-design/plots";

new Column("app", {
  data: [
    { day: "Mon", value: 120 }, { day: "Tue", value: 200 },
    { day: "Wed", value: 150 }, { day: "Thu", value: 80 }, { day: "Fri", value: 70 },
  ],
  xField: "day",
  yField: "value",
  interactions: [{ type: "element-highlight" }],
  animation: { appear: { animation: "scale-in-y", duration: 400 } },
  theme: "classic"
});
```

---

## 5) 与 React / Vue / Svelte 集成 🔌
- **优先封装**：  
  - React：`echarts-for-react`、`@ant-design/plots`、`@antv/g2-react`  
  - Vue：`vue-echarts`、`@antv/g2plot-vue`  
- **单向数据流**：应用状态 → 图表配置/数据，交互事件（`onEvents`/`onPlotClick`）再回流。  
- **避免重复 init**：在 `useEffect`/`onMounted` 只 `init` 一次，后续用 `setOption`/`update()`。  
- **尺寸自适应**：容器变化监听（`ResizeObserver`）→ `chart.resize()`；SSR/同构时延迟到 `mounted` 执行。

---

## 6) 性能工程（从 1k 到 1M 点）🏎️
- **渲染选择**：  
  - <3k 元素：SVG（开发舒服，交互易）  
  - 3k–100k：Canvas（ECharts/AntV 默认已经优化）  
  - >100k：WebGL/瓦片/抽稀/聚合（ECharts GL / AntV L7 / 自研）  
- **数据瘦身**：下采样（LTTB/最大最小法）、时间窗口聚合（minute→5min）、分桶（histogram）。  
- **动画策略**：进入动画一次性；交互动画不超过 300–500ms；**禁用全量重绘**（增量更新）。  
- **离屏与并行**：`OffscreenCanvas + Worker` 做重计算（布局/采样）；主线程只“呈现”。  
- **事件门限**：密集滚动/拖拽节流（`requestAnimationFrame`），工具提示走命中测试而非遍历。

---

## 7) 交互与图联动（Drilldown / Brush / Link）🪄
- **交互原子**：`hover / select / brush / zoom / pan / tooltip / contextmenu`  
- **联动协议**：统一事件总线（`eventBus.emit("filter:update", payload)`），所有图订阅；**不要相互直接调用**。  
- **跨图表高亮**：使用共享 `seriesId/itemId` 或统一键（如 `userId`），两边互相 `highlight/dispatchAction`。  
- **钻取**：Click → 更新全局 filter → 上游 API 拉子数据 → 子视图刷新。

---

## 8) 主题与设计系统（视觉一致性）🎨
- **色板**：主色+语义色（正/负/中性），顺序色与定性色分开管理。  
- **暗色模式**：背景、网格、文本对比度 ≥ 4.5:1；ECharts `theme` / G2 `theme` 一处切换。  
- **状态样式**：`selected/hover/disabled/emphasis` 统一定义。  
- **单位/格式**：千分位、时间/货币本地化；同比/环比箭头与色彩统一。

---

## 9) a11y 与国际化（别忘了）🧏🌍
- **可达性**：给图表容器 `role="img"` 与 `aria-label`；提供**数据表格视图**与**下载 CSV**。  
- **键盘操作**：可切换系列/高亮数据点；工具提示可用键盘触达。  
- **文本与数字**：本地化日期/货币；数字缩写（K/M/B）与中文万/亿切换。

---

## 10) 地图/关系/表格扩展（AntV 家族）🧩
- **L7（地图）**：热力、格网、流向、3D 柱；数据量大走 WebGL。  
- **G6（关系）**：力导/层次/径向布局；边样式、团簇、路径查找。  
- **S2（明细表/透视）**：百万级单元格虚拟化、冻结行列、数值汇总；与图表联动做“图表+表格”。

---

## 11) 测试与可维护性 🧪
- **快照**：配小数据集渲染静态 SVG/Canvas 快照（像素近似比较）。  
- **契约**：封装“绘图函数”接收**标准化数据结构**（例如 `{columns, rows}`），便于替换底座。  
- **监控**：埋点记录**渲染耗时/卡顿率/交互错误**；首屏图表延后挂载以保证 LCP。  
- **Storybook**：每种图一个 story，含交互态（selected/hover/empty/error）。

---

## 12) 反模式与纠偏 🧨
| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 直接把原始百万点全画 | 页面卡死 | 下采样/聚合/视窗+虚拟化/WebGL |
| 双向杂耦 | 一个图改动另一个图炸 | 事件总线 + 单向数据流 |
| 多次 init / 泄漏 | 切页内存暴涨 | 只 `init` 一次，记得 `dispose` |
| 所有样式写在 option | 难统一 | 主题集中配置 + 语义色 |
| 图表逻辑写在组件里 | 重复难测 | 提炼“制图器”纯函数（输入→配置） |
| 高频刷新 setOption 全量 | 抖动卡顿 | 增量更新/节流/`lazyUpdate` |

---

## 13) 验收清单 ✅
- [ ] 选型依据与“退路”（要不要 D3 兜底）已写入设计说明。  
- [ ] 主题与色板统一，暗黑模式对比度合规。  
- [ ] 数据量评估与降采样策略落地（>10k Canvas，>100k WebGL/抽稀）。  
- [ ] 事件总线与联动协议完成，组件无环依赖。  
- [ ] 渲染/交互性能监控与告警上线。  
- [ ] a11y：ARIA 文案 + 表格备选 + 键盘路径。  
- [ ] 单元/可视回归测试通过；`dispose()` 无泄漏。

---

## 14) 练习 🏋️
1. 用 **ECharts** 做折线+柱状双轴，支持缩放、区域刷选、高亮联动。  
2. 用 **D3** 实现自定义散点图，附带四象限背景与拖拽选框。  
3. 用 **AntV Plot** 复现“漏斗 + 环比标注”，并封装成 `useChart` 钩子。  
4. 同一数据源在三家各实现一次，对比：代码行数/重构难度/性能与交互。

---

## 15) 模板片段（工程可复用）🧰

### 15.1 React + ECharts 封装
```tsx
import React, { useEffect, useRef } from "react";
import * as echarts from "echarts";

export function EChart({ option, style }) {
  const ref = useRef(null);
  const inst = useRef(null);

  useEffect(() => {
    inst.current = echarts.init(ref.current);
    const ro = new ResizeObserver(() => inst.current?.resize());
    ro.observe(ref.current);
    return () => { ro.disconnect(); inst.current?.dispose(); };
  }, []);

  useEffect(() => { inst.current?.setOption(option, { notMerge: true, lazyUpdate: true }); }, [option]);
  return <div ref={ref} style={{ width: "100%", height: 300, ...style }} />;
}
```

### 15.2 数据下采样（LTTB 简化版）
```js
export function lttb(points, threshold=2000) {
  if (points.length <= threshold) return points;
  const bucketSize = (points.length - 2) / (threshold - 2);
  const sampled = [points[0]];
  let a = 0;
  for (let i=0; i<threshold-2; i++) {
    const start = Math.floor((i+1)*bucketSize) + 1;
    const end = Math.floor((i+2)*bucketSize) + 1;
    const bucket = points.slice(start, end);
    const avgX = bucket.reduce((s,p)=>s+p.x,0)/bucket.length;
    const avgY = bucket.reduce((s,p)=>s+p.y,0)/bucket.length;
    let maxArea=-1, next=a+1;
    for (let j=start;j<end;j++){
      const area = Math.abs((points[a].x-avgX)*(points[j].y-points[a].y) - (points[a].x-points[j].x)*(avgY-points[a].y));
      if (area>maxArea){maxArea=area;next=j;}
    }
    sampled.push(points[next]); a = next;
  }
  sampled.push(points[points.length-1]);
  return sampled;
}
```

---

**小结**：**ECharts** 解决“快”和“稳”，**D3** 解决“难”和“新”，**AntV** 在“语法化 + 体系化”之间取平衡。把**选型→语法→工程→性能→联动→测试**一条龙打通，你的可视化就不再是“画图”，而是**产品能力**。🛠️✨
