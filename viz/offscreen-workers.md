# 21.2 OffscreenCanvas / Workers 协同 🧵🖼️⚙️

目标：把**渲染从主线程解放**，把**计算从渲染解耦**。用 `OffscreenCanvas` + `Web Worker` 做到**不卡 UI、不卡输入、不卡动画**。口号：**“UI 在前、像素在后，消息是血。”**

---

## 0) TL;DR（三条背熟）🎯
1. **管线分层**：主线程只管 UI/输入/合成；Worker 负责绘制（`OffscreenCanvas`）或重计算；消息是单一数据通道。  
2. **零拷贝优先**：`transferControlToOffscreen()`、`postMessage(..., [buffer])`、`ImageBitmap`、`SharedArrayBuffer+Atomics`（需跨源隔离）。  
3. **可降级**：没 `OffscreenCanvas` → Worker 只做计算，主线程 `CanvasRenderingContext2D`（或 WebGL/Three.js）渲染；性能开关：降分辨率/降帧/抽稀。

---

## 1) 体系结构（谁干什么）🧭
- **主线程（Window）**  
  - 负责：布局、事件（pointer/keyboard）、合成（CSS/DOM）、状态管理、rAF 驱动。  
  - 做法：  
    - 把 `<canvas>` **转移控制权**：`const off = canvas.transferControlToOffscreen()` → 通过 `postMessage` 传给 Worker（可转移对象）。  
    - 用 `ResizeObserver` 同步尺寸；用 rAF 发“节拍”或“新状态”到 Worker。  
- **Worker（Dedicated / Shared）**  
  - 负责：绘制（`OffscreenCanvas` 上下文 `2d` / `webgl(2)` / *可能* `webgpu`）、重计算（布局、采样、物理、聚合）。  
  - 做法：  
    - 接收 OffscreenCanvas、资源句柄、状态增量；渲染后无需回传像素（画面已在 GPU/合成器）。  
    - 监听主线程消息，**无共享可变 UI 状态**，只拉快照或补丁。

> 口诀：**ViewModel 在主、Renderer 在工**；**消息不可变，渲染幂等**。

---

## 2) 最小实战：2D 渲染搬进 Worker（可直接抄）🧪

### 2.1 主线程（window.ts）
```ts
const canvas = document.getElementById('c') as HTMLCanvasElement;
const offscreen = (canvas as any).transferControlToOffscreen();
const worker = new Worker(new URL('./worker.js', import.meta.url), { type: 'module' });
worker.postMessage({ type: 'init', canvas: offscreen, dpr: devicePixelRatio }, [offscreen]);

// 驱动节拍（更稳：用 rAF 把时间戳发给 Worker）
function tick(t: number) {
  worker.postMessage({ type: 'tick', t });
  requestAnimationFrame(tick);
}
requestAnimationFrame(tick);

// 同步尺寸
const ro = new ResizeObserver(([entry]) => {
  const { inlineSize, blockSize } = (entry.contentBoxSize?.[0] ?? entry.contentRect);
  worker.postMessage({ type: 'resize', w: Math.floor(inlineSize), h: Math.floor(blockSize) });
});
ro.observe(canvas);

// 输入事件（仅传必要字段）
canvas.addEventListener('pointermove', e => {
  worker.postMessage({ type: 'pointer', x: e.offsetX, y: e.offsetY, pressure: e.pressure }, { transfer: [] as any });
});
```

### 2.2 Worker（worker.ts）
```ts
let ctx: OffscreenCanvasRenderingContext2D;
let W = 300, H = 150, DPR = 1;
let state = { x: 0, y: 0 };

self.onmessage = (e: MessageEvent) => {
  const msg = e.data;
  if (msg.type === 'init') {
    const cvs: OffscreenCanvas = msg.canvas;
    DPR = msg.dpr ?? 1;
    cvs.width = Math.floor(cvs.width * DPR);
    cvs.height = Math.floor(cvs.height * DPR);
    ctx = cvs.getContext('2d')!;
    ctx.scale(DPR, DPR);
  } else if (msg.type === 'resize') {
    W = msg.w; H = msg.h;
    const cvs = ctx.canvas as OffscreenCanvas;
    cvs.width = Math.floor(W * DPR);
    cvs.height = Math.floor(H * DPR);
    ctx.setTransform(DPR,0,0,DPR,0,0);
  } else if (msg.type === 'pointer') {
    state.x = msg.x; state.y = msg.y;
  } else if (msg.type === 'tick') {
    render(msg.t);
  }
};

function render(t: number) {
  if (!ctx) return;
  ctx.clearRect(0,0,W,H);
  ctx.fillStyle = '#111'; ctx.fillRect(0,0,W,H);

  // demo: 跟随指针的圆 + 时间驱动
  const r = 10 + 6 * Math.sin(t/300);
  ctx.fillStyle = '#4e79a7';
  ctx.beginPath(); ctx.arc(state.x, state.y, r, 0, Math.PI*2); ctx.fill();

  ctx.fillStyle = '#fff'; ctx.font = '12px system-ui';
  ctx.fillText(`t=${(t/1000).toFixed(2)}s @ ${W}x${H}`, 8, 16);
}
```

> **为何用主线程 rAF 驱动？** Worker 里的 rAF 支持在各家浏览器并不完全一致；用主线程 rAF 作为“时钟”更稳定，也能随页面节流/暂停。

---

## 3) WebGL/Three.js 进 Worker（不卡 UI 的 3D）🧊

### 3.1 主线程
```ts
const off = (document.getElementById('gl') as HTMLCanvasElement).transferControlToOffscreen();
const worker = new Worker(new URL('./gl-worker.js', import.meta.url), { type: 'module' });
worker.postMessage({ canvas: off, dpr: devicePixelRatio, kind: 'three' }, [off]);

// 仍由主线程 rAF 发“tick”（也可只在交互/状态变化时发补丁）
const tick = (t:number)=>{ worker.postMessage({ type: 'tick', t }); requestAnimationFrame(tick); };
requestAnimationFrame(tick);
```

### 3.2 Worker（Three.js 渲染）
```ts
import * as THREE from 'three';

let renderer: THREE.WebGLRenderer, scene: THREE.Scene, camera: THREE.PerspectiveCamera;
let ready = false;

self.onmessage = (e: MessageEvent) => {
  const { canvas, dpr, type, t } = e.data;
  if (canvas) {
    renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setPixelRatio(Math.min(dpr, 2));
    renderer.setSize(800, 450, false);

    scene = new THREE.Scene();
    camera = new THREE.PerspectiveCamera(60, 800/450, 0.1, 100);
    camera.position.set(2.2, 1.8, 2.2);
    scene.add(new THREE.AmbientLight(0xffffff, 0.3));
    const light = new THREE.DirectionalLight(0xffffff, 1); light.position.set(3,4,2); scene.add(light);

    const cube = new THREE.Mesh(new THREE.BoxGeometry(1,1,1), new THREE.MeshStandardMaterial({ color: '#4e79a7' }));
    scene.add(cube);
    (self as any).__cube = cube; // debug

    ready = true;
    return;
  }
  if (ready && typeof t === 'number') {
    (self as any).__cube.rotation.y = t * 0.001;
    renderer.render(scene, camera);
  }
};
```

> 说明：Three.js **可以**在 Worker 里用 `OffscreenCanvas` 获取 WebGL 上下文。统一由主线程发“tick”可避免 Worker 自转导致后台标签页不节流的问题。

---

## 4) 高效消息传输（别把总线塞爆）📦
- **Transferable**：通过 `postMessage(payload, [arrayBuffer])` **转移**所有权，避免拷贝（如点云 `Float32Array.buffer`）。  
- **ImageBitmap**：主线程 `createImageBitmap(blob)` → 传给 Worker → 2D/WebGL 里 `drawImage()` / `texImage2D()`；或 Worker 产 `ImageBitmap` 给主线程用 `bitmaprenderer` 显示。  
- **SharedArrayBuffer + Atomics**（需跨源隔离：`COOP: same-origin` + `COEP: require-corp`）  
  - 环形队列：Worker 写、主线程读；用 `Atomics.store/add/notify/wait` 做轻量同步。  
  - 适合高频小批量数据（如游标/传感器）。  
- **批处理消息**：把每帧多次更新**合并为一次**（微队列），或仅发“脏矩形/脏对象”列表。

---

## 5) 图片/字体/解码在 Worker（CPU 繁重搬家）🧩
- **图片**：在 Worker 中 `fetch()` → `createImageBitmap()` → 直接绘制到 OffscreenCanvas（避免主线程解码卡顿）。  
- **字体**：`FontFace` 可在主线程加载并通过 CSSOM 生效；WebGL 文字建议使用 **SDF 文本纹理**，在 Worker 构建字图集。  
- **数据处理**：排序、聚合、空间索引（R-Tree）、抽稀（LTTB）在 Worker 完成，主线程只拿结果集。

---

## 6) 动态分辨率 / 动态帧率（自适应不掉帧）📉➡️📈
- FPS 目标 60 → 帧预算 **16.67ms**（渲染建议 ≤ 10ms）。  
- **动态分辨率**：测一秒均帧耗时，超标则 `canvas.width/height *= 0.85`，低于阈值缓慢上调。  
- **动态帧率**：根据交互状态（静止/连续拖拽）把 rAF 间隔改为 **30/60**（主线程控制“tick”频率）。  
- **节流输入**：`pointermove` 走 `requestAnimationFrame` 合帧（而非每次 move 都发）。

---

## 7) WebGPU 与 OffscreenCanvas（前瞻）🧪
- 现代浏览器正逐步支持 `OffscreenCanvas.getContext('webgpu')`。  
- 策略：**能力检测** → 优先 `webgpu`，失败回退 `webgl2`；资源/管线接口抽象一层，**渲染后端可替换**。  
- Compute 着色器适合放在 **Worker 的 WebGPU**，将结果写入 `GPUBuffer`/`Texture` 后直接渲染（零拷贝）。

> 兼容性仍需实测，CI 用 `@vitejs/plugin-legacy` + 自写 `feature-detect` 逻辑兜底。

---

## 8) 兼容与降级 🪂
- 无 `OffscreenCanvas`：  
  - Worker 只做**计算**（抽稀/布局/切片），主线程负责 Canvas/WebGL 渲染；  
  - 或使用 `createImageBitmap` 在 Worker 合成后传回主线程用 `bitmaprenderer` 绘制（有一次复制，但仍低卡顿）。  
- 无 Worker：  
  - 在主线程采用**分块渲染**（分帧处理） + `setTimeout(0)` 让出控制权；  
  - 降低分辨率/关闭特效，显示“性能模式”。

---

## 9) 调试与观测 🔍
- `console.log` 可在 Worker 直接输出；在 Vite/webpack 下**记得生成 source map**。  
- 统计指标：**渲染耗时/队列长度/消息吞吐/丢帧率**（在 Worker 内记录，按秒回传主线程上报）。  
- `performance.mark/measure` 两端都可以用；自定义面板显示“tick 延迟分布”。

---

## 10) 反模式与纠偏 🧨
| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 每个事件都 `postMessage` | 队列爆炸、延迟抖动 | 合帧/批量；仅发“状态增量” |
| 传大对象不转移 | GC 压力、卡顿 | 用 **Transferable** 或 **SAB** |
| Worker 自转 rAF | 后台不节流、电耗高 | 用主线程 rAF 作为“时钟” |
| 每 resize 全新建上下文 | 黑屏闪烁 | 只重设 `width/height` + `setTransform` |
| 主线程还做图像解码 | 滚动丢帧 | Worker 内 `createImageBitmap` |
| 渲染与计算绑死一帧 | 尾帧抖动 | 计算提前做、渲染只吃快照 |

---

## 11) 验收清单 ✅
- [ ] `OffscreenCanvas` 在 Worker 里拿到 `2d`/`webgl(2)` 上下文并稳定渲染。  
- [ ] 主线程 rAF 驱动或状态驱动；输入事件合帧/节流。  
- [ ] 大数据通过 **Transferable/SAB** 零拷贝传输；图片用 **ImageBitmap**。  
- [ ] 尺寸/DPR 同步正确；动态分辨率/帧率策略生效。  
- [ ] 降级路径可用（无 OffscreenCanvas / 无 Worker）。  
- [ ] 指标：平均帧耗时、p95、吞吐与丢帧告警接入。

---

## 12) 练习 🏋️
1. 把现有 Canvas 图表搬进 Worker（2D），接入 `ImageBitmap` 纹理缓存，对比输入流畅度与 CPU。  
2. 用 Three.js + OffscreenCanvas 在 Worker 渲染，主线程仅转发交互（拖拽/缩放），测量合成帧率。  
3. 实现 **SAB 环形队列**，把 60Hz 鼠标坐标和 120Hz 传感器数据同时喂给渲染端，验证原子同步。  
4. 做一套 **动态分辨率控制器**：根据 1s 滑动窗口帧耗时，自动上/下调渲染分辨率。

---

**小结**：`OffscreenCanvas + Worker` 就是把**像素流水线**从主线程拆出去：**UI 不被拖累、输入不卡、动画不抖**。配合零拷贝与降级策略，你的 2D/3D、海量点可视与复杂动画都会“稳到离谱”。🚀
