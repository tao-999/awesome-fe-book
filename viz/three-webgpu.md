# 21.1 Canvas / WebGL / WebGPU 与 Three.js 🧱🧭🚀

目标：搞清 **2D Canvas → WebGL → WebGPU** 的演进、各自适用场景与**性能/工程化**打法；并给出 **Three.js** 的高质量落地套路（加载、PBR、动画、后处理、挑选等）。  
口号：**“像素谁画？CPU 画是 Canvas，GPU 画是 *WebGL/WebGPU*；高效做 3D，上 Three.js。”**

---

## 0) TL;DR（三件事先记住）🎯
1. **选型**：  
   - 2D UI/轻量图形/截图渲染 → **Canvas 2D**。  
   - 跨端覆盖最稳的 3D/大数据点/自定义着色器 → **WebGL**（或 Three.js）。  
   - 现代高性能/计算着色器/可编排管线 → **WebGPU**（可回退 WebGL）。  
2. **性能**：**批处理/实例化/纹理图集/降分辨率/离屏 + Worker**；移动端优先 **≤ 2K 分辨率**、纹理用 **KTX2**。  
3. **工程化**：**glTF + Draco + KTX2** 资源管线；**React Three Fiber** 封装；**Raycaster** 拾取；**postprocessing** 做后效。

---

## 1) 三者心智模型与能做什么 🧠
| 维度 | Canvas 2D | WebGL(1/2) | WebGPU |
|---|---|---|---|
| 抽象层级 | 立即模式 2D API | OpenGL ES 风格即时渲染 | 现代显卡管线（命令缓冲、绑定组、计算着色器） |
| 着色器 | 无 | GLSL（顶点/片元） | WGSL（Vertex/Fragment/Compute） |
| 并行计算 | 几乎无 | 可用片元/顶点着色器 | **Compute Shader** 一等公民 |
| 覆盖度 | 非常广 | 非常广（移动/桌面） | 新一代浏览器（Chrome/Edge/Safari/Firefox 逐步普及） |
| 典型场景 | 2D 图形、富文本截图、简单游戏 | 3D 场景/海量点线面、可视化、游戏 | PBR 大场景、高复杂后效、GPGPU、超大数据 |
| 上手成本 | 低 | 中（原生）/低（Three.js） | 中-高 |

> 选择顺序：**WebGPU 有则用、否则 WebGL**；**非 3D/轻量可视 → Canvas**；**优先 Three.js 封装**，除非你要完全定制图形管线。

---

## 2) 从零到一：最小可跑示例 🧪

### 2.1 Canvas 2D（条形图草稿）
```html
<canvas id="c" width="400" height="160"></canvas>
<script>
const ctx = document.getElementById('c').getContext('2d');
const data = [12,36,28,50,18], w=400,h=160,m=24;
const max = Math.max(...data);
ctx.font = '12px system-ui';
data.forEach((v,i)=>{
  const x = m + i * ((w-2*m)/data.length);
  const bw = (w-2*m)/data.length - 10;
  const bh = (h-2*m) * (v/max);
  ctx.fillStyle = '#4e79a7';
  ctx.fillRect(x, h-m-bh, bw, bh);
  ctx.fillStyle = '#333';
  ctx.fillText(String(v), x, h-m-bh-4);
});
</script>
```

### 2.2 WebGL（Hello Triangle：GLSL）
```html
<canvas id="gl" width="400" height="300"></canvas>
<script>
const gl = document.getElementById('gl').getContext('webgl');
const vs = gl.createShader(gl.VERTEX_SHADER);
gl.shaderSource(vs, `
attribute vec2 pos;
void main(){ gl_Position = vec4(pos, 0.0, 1.0); }
`); gl.compileShader(vs);
const fs = gl.createShader(gl.FRAGMENT_SHADER);
gl.shaderSource(fs, `
precision mediump float;
void main(){ gl_FragColor = vec4(0.3,0.6,0.9,1.0); }
`); gl.compileShader(fs);
const prog = gl.createProgram();
gl.attachShader(prog, vs); gl.attachShader(prog, fs); gl.linkProgram(prog); gl.useProgram(prog);

const buf = gl.createBuffer();
gl.bindBuffer(gl.ARRAY_BUFFER, buf);
gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([ -0.6,-0.6, 0.6,-0.6, 0.0,0.6 ]), gl.STATIC_DRAW);
const loc = gl.getAttribLocation(prog, 'pos');
gl.enableVertexAttribArray(loc);
gl.vertexAttribPointer(loc, 2, gl.FLOAT, false, 0, 0);

gl.clearColor(1,1,1,1); gl.clear(gl.COLOR_BUFFER_BIT);
gl.drawArrays(gl.TRIANGLES, 0, 3);
</script>
```

### 2.3 WebGPU（Hello Triangle：WGSL）
```html
<canvas id="gpu" width="400" height="300"></canvas>
<script type="module">
if (!('gpu' in navigator)) { alert('WebGPU 不可用，降级到 WebGL/Three.js'); }
const canvas = document.getElementById('gpu');
const adapter = await navigator.gpu.requestAdapter();
const device = await adapter.requestDevice();
const context = canvas.getContext('webgpu');
const format = navigator.gpu.getPreferredCanvasFormat();
context.configure({ device, format, alphaMode: 'premultiplied' });

const vertices = new Float32Array([ -0.6,-0.6, 0.6,-0.6, 0.0,0.6 ]);
const vbuf = device.createBuffer({ size: vertices.byteLength, usage: GPUBufferUsage.VERTEX|GPUBufferUsage.COPY_DST });
device.queue.writeBuffer(vbuf, 0, vertices);

const shader = device.createShaderModule({ code: `
@vertex fn vs(@location(0) pos: vec2f) -> @builtin(position) vec4f {
  return vec4f(pos, 0.0, 1.0);
}
@fragment fn fs() -> @location(0) vec4f {
  return vec4f(0.2, 0.7, 0.5, 1.0);
}`});

const pipeline = device.createRenderPipeline({
  layout: 'auto',
  vertex: { module: shader, entryPoint: 'vs', buffers: [{ arrayStride: 8, attributes: [{ shaderLocation: 0, format: 'float32x2', offset: 0 }]}]},
  fragment: { module: shader, entryPoint: 'fs', targets: [{ format }] },
  primitive: { topology: 'triangle-list' }
});

function frame(){
  const encoder = device.createCommandEncoder();
  const pass = encoder.beginRenderPass({
    colorAttachments: [{ view: context.getCurrentTexture().createView(),
      clearValue: {r:1,g:1,b:1,a:1}, loadOp: 'clear', storeOp: 'store' }]
  });
  pass.setPipeline(pipeline); pass.setVertexBuffer(0, vbuf); pass.draw(3); pass.end();
  device.queue.submit([encoder.finish()]);
  requestAnimationFrame(frame);
}
frame();
</script>
```

### 2.4 Three.js（旋转立方体 + 阴影）
```html
<div id="app"></div>
<script type="module">
import * as THREE from 'https://unpkg.com/three@0.161.0/build/three.module.js';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.shadowMap.enabled = true;
document.getElementById('app').appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color('#202225');

const camera = new THREE.PerspectiveCamera(60, innerWidth/innerHeight, 0.1, 100);
camera.position.set(2.2, 1.8, 2.2);

const geo = new THREE.BoxGeometry(1,1,1);
const mat = new THREE.MeshStandardMaterial({ color: '#4e79a7', roughness: 0.5, metalness: 0.2 });
const cube = new THREE.Mesh(geo, mat);
cube.castShadow = true; scene.add(cube);

const plane = new THREE.Mesh(new THREE.PlaneGeometry(6,6), new THREE.ShadowMaterial({ opacity: 0.25 }));
plane.rotation.x = -Math.PI/2; plane.position.y = -0.6; plane.receiveShadow = true; scene.add(plane);

const light = new THREE.DirectionalLight('#ffffff', 1.0);
light.position.set(3, 4, 2); light.castShadow = true; scene.add(light);

function resize(){ camera.aspect = innerWidth/innerHeight; camera.updateProjectionMatrix(); renderer.setSize(innerWidth, innerHeight); }
addEventListener('resize', resize);

const clock = new THREE.Clock();
function animate(){
  const t = clock.getElapsedTime();
  cube.rotation.x = t * 0.6; cube.rotation.y = t * 0.8;
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
animate();
</script>
```

---

## 3) Three.js 实战要点（PBR/资源/后效/拾取）🧰

### 3.1 资源加载管线（**glTF 首选**）
- **模型**：`glTF 2.0`（`.gltf/.glb`），内含网格、材质（PBR）、骨骼动画。  
- **压缩**：几何 **Draco**；纹理 **KTX2（BasisU）** → 显著减体积 + 显存友好。  
- **代码**
```js
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { KTX2Loader } from 'three/examples/jsm/loaders/KTX2Loader.js';
const gltfLoader = new GLTFLoader();
const ktx2 = new KTX2Loader().setTranscoderPath('/basis/').detectSupport(renderer);
gltfLoader.setKTX2Loader(ktx2);

gltfLoader.load('/models/scene.glb', (gltf)=>{
  scene.add(gltf.scene);
});
```

### 3.2 PBR 与环境贴图
```js
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js';
new RGBELoader().load('/hdr/venice_sunset_1k.hdr', (tex)=>{
  tex.mapping = THREE.EquirectangularReflectionMapping;
  scene.environment = tex; // 反射/折射环境光
  scene.background = tex;  // 如需背景同 HDR
});
```

### 3.3 后处理（Bloom/FXAA/TAA）
```js
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';
const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
composer.addPass(new UnrealBloomPass(new THREE.Vector2(innerWidth, innerHeight), 0.8, 0.4, 0.85));

function animate(){ /* ...更新... */ composer.render(); requestAnimationFrame(animate); }
```

### 3.4 拾取（Raycaster）
```js
import { Raycaster, Vector2 } from 'three';
const ray = new Raycaster(), mouse = new Vector2();
renderer.domElement.addEventListener('pointermove', (e)=>{
  const r = renderer.domElement.getBoundingClientRect();
  mouse.x = ((e.clientX - r.left)/r.width)*2 - 1;
  mouse.y = -((e.clientY - r.top)/r.height)*2 + 1;
  ray.setFromCamera(mouse, camera);
  const hit = ray.intersectObjects(scene.children, true)[0];
  document.body.style.cursor = hit ? 'pointer' : 'default';
});
```

---

## 4) 性能工程（移动端优先）🏎️

### 4.1 预算与分辨率
- 60 FPS 帧预算 ≈ **16.67ms**（渲染只占 8–10ms 更稳）；120 FPS → **8.33ms**。  
- 屏幕像素密度限制：`renderer.setPixelRatio(Math.min(devicePixelRatio, 2))`；UI 模块使用 **动态分辨率**。

### 4.2 批处理/实例化/图集
- **实例化**：百万草地/粒子 → `THREE.InstancedMesh`。  
- **合并**：静态几何 `BufferGeometryUtils.mergeVertices/mergeBufferGeometries`。  
- **图集**：多小贴图合一，减少 **texture bind**。

### 4.3 材质与阴影
- 阴影昂贵：优先 **PCFSoftShadowMap**，限制投射数量与 `shadow.mapSize`。  
- 材质层级：尽量共享 `MeshStandardMaterial`；禁用未用特性（`normalMap`/`envMap`）。

### 4.4 纹理与显存
- 用 **KTX2**；对大贴图加 **mipmap** 与合理 **anisotropy**（≤ 8）；颜色空间 `sRGBEncoding`。  
- 尽量使用 **power-of-two** 尺寸（兼容性+采样效果）。

### 4.5 线程与离屏
- 复杂解析/物理/路径计算放 **Worker**；渲染用 **OffscreenCanvas**（需浏览器支持）。  
- 大量点/线可用 **WebGL2 Transform Feedback** 或 WebGPU **Compute** 做 GPGPU。

---

## 5) WebGPU 进阶（什么时候值得用）🧪➡️⚙️
- **你需要**：计算着色器（粒子/GPGPU）、多渲染目标、高级同步/管线缓存、极致可控的绑定与命令缓冲。  
- **迁移策略**：封装渲染后端接口 → **WebGPU 实现** + **WebGL 回退**；资源管线不变（glTF/KTX2 继续用）。  
- **注意**：Safari/Firefox 的可用性与权限策略；CI 中跑 **can-i-use** 检测，动态切换渲染后端。

---

## 6) 交互、坐标与相机（不迷路）🧭
- 坐标系（Three.js）：右手系，+X 右、+Y 上、+Z 出屏。  
- 相机：`PerspectiveCamera`（大多数 3D）、`OrthographicCamera`（UI/等距）。  
- 控制：`OrbitControls` 适合浏览器 3D 浏览；FPS/轨迹球按需替换。  
- **世界单位**统一：米/厘米随项目固定；避免动态缩放导致的 z-fighting。

---

## 7) 框架集成（React/Vue/Svelte）🔌
- **React**：`react-three-fiber`（R3F）+ `drei`（辅助组件），声明式组合场景。  
- **Vue**：`troisjs` / 自封装。  
- **R3F 简例**
```tsx
import { Canvas } from '@react-three/fiber';
import { OrbitControls, StatsGl } from '@react-three/drei';

function Box(){
  return (
    <mesh rotation={[0.4,0.2,0.0]}>
      <boxGeometry args={[1,1,1]} />
      <meshStandardMaterial color="#4e79a7" metalness={0.2} roughness={0.5}/>
    </mesh>
  );
}

export default function App(){
  return (
    <Canvas dpr={[1,2]} shadows camera={{ position: [2.5,1.8,2.5], fov: 60 }}>
      <ambientLight intensity={0.2} />
      <directionalLight position={[3,4,2]} castShadow />
      <Box />
      <OrbitControls />
      <StatsGl />
    </Canvas>
  );
}
```

---

## 8) 资源与构建流水线（实操）🛠️
- **转换**：DCC（Blender/Maya）→ glTF；`gltfpack`/`Draco` 压缩；`ktx2ktx2` 转纹理。  
- **版本控制**：二进制资源走对象存储（CDN），代码仓只存 **指针/manifest**。  
- **预加载**：**顶层路由 prefetch** 核心模型；进入场景后懒加载次要对象。  
- **缓存**：CDN + `Cache-Control`；版本戳 hash；Service Worker 可做离线缓存。

---

## 9) 光照、阴影与后效（视觉升级）💡
- **IBL**（基于图像的光照）提升 PBR 质感（HDR 环境纹理）。  
- **阴影**：方向光 + 级联（CSM）适合大场景；点光影子慎用。  
- **后效链**：FXAA/TAA → Bloom → LUT/色调映射（Filmic/ACES） → Vignette。  
- **抗锯齿**：WebGL 多用 **MSAA（离屏不可用时 → FXAA/TAA）**；WebGPU 原生 MSAA 友好。

---

## 10) 拾取与选择（WebGL 原生思路）🎯
- **颜色编码拾取**：离屏渲染每对象唯一颜色 → 读回像素 → ID。  
- **深度排序**：透明对象后渲染，或按深度分层；尽量减少混合对象数量。  
- **Three.js** 优先用 `Raycaster`，性能不足时再换颜色拾取方案。

---

## 11) 移动端适配与 WebXR 🌐
- **移动限制**：纹理 ≤ 4096、`precision mediump`、谨慎使用高阶后效。  
- **电源策略**：可降帧（30 FPS）、降低分辨率、暂停后台渲染（`visibilitychange`）。  
- **WebXR**：Three.js 内置；交互参考 `XRControllerModelFactory`；注意空间映射权限与 FOV 变化。

---

## 12) 安全与可访问性 🔒🧏
- **跨域**：纹理需 `crossOrigin` + CORS；否则无法读像素/导出。  
- **隐私**：避免将用户数据嵌在着色器源码或材质 URL 中。  
- **a11y**：提供**数据表/文本替代**、**键盘操作路径**、**低性能模式开关**；Canvas 输出可配 **Offscreen 渲染 + ARIA label**。

---

## 13) 反模式与纠偏 🧨
| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 逐帧全场景重建 | FPS 过山车 | 复用网格/材质；只更新变动对象 |
| 过高像素比（dpr=3/4） | 手机发烫掉帧 | `setPixelRatio(min(dpr,2))` + 动态分辨率 |
| 阴影铺满 + 高分辨率 | 阴影模糊/卡顿 | 控制投射者数量、CSM、合理 mapSize |
| 纹理 png/jpg 直传 | 包体/显存暴涨 | **KTX2** + mipmap + POT |
| 全靠 CPU 计算粒子 | 主线程爆炸 | GPGPU（WebGL TF/WebGPU Compute） |
| 在渲染循环里 `new` 对象 | GC 抖动 | 预分配、对象池 |
| 混合模式滥用 | 排序错误、闪烁 | 透明对象后绘，减少混合数量 |

---

## 14) 验收清单 ✅
- [ ] 设备检测（WebGPU → WebGL → Canvas）与**回退**路径可用。  
- [ ] Three.js：**glTF + Draco + KTX2** 管线稳定，资源按需/预加载。  
- [ ] FPS 与帧预算可视化（`Stats`/自定义指标），移动端热/功耗达标。  
- [ ] 批处理与实例化落地；阴影与后效有开关。  
- [ ] 纹理与颜色空间正确（`sRGBEncoding`、toneMapping）。  
- [ ] 拾取/交互流畅，读取像素时已配置 CORS。  
- [ ] 低性能模式（降分辨率/降帧/关闭 bloom）可切换。

---

## 15) 练习 🏋️
1. 用 **Three.js** 加载一个 `glb`，启用 **HDR IBL + 阴影 + Bloom**，提供移动端“低性能模式”开关。  
2. 把 5 万个实例化方块做成“音频频谱体素”，使用 `InstancedMesh` + `dynamic draw` 更新。  
3. 用 **WebGPU** 实现 10 万粒子的 **Compute Shader** 更新 + 渲染；回退 WebGL 用 Transform Feedback。  
4. 为你的项目接入 **KTX2** 纹理与 **Draco** 模型压缩，比较包体/显存/加载时长。

---

**小结**：Canvas 做“手起刀落”的 2D，WebGL 是现阶段**最通用的 GPU 渲染底座**，WebGPU 则把“现代管线”和“计算”带到浏览器。多数场景选 **Three.js** 直上，把 **资源管线/性能/工程化**打全套，你的 3D 前端就既好看又好跑。🎮✨
