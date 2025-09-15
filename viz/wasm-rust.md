# 23.1 Rust / wasm-pack / 性能实践 🦀⚙️⚡

目标：用 **Rust + WebAssembly** 把前端的“重计算/高并发/可移植”三件事一次吃透：**更快**（SIMD/多线程）、**更稳**（类型安全/内存安全）、**更省**（体积/能耗）。  
口号：**“把 CPU 卷起来，让 JS 轻一点。”**

---

## 0) TL;DR（三件必做）🎯
1. **工程基线**：`crate-type = ["cdylib"]` + `wasm-bindgen` + `wasm-pack build --release`；用 `vite`/`webpack` 挂载；**接口细粒度**避免频繁跨边界。
2. **性能三板斧**：**零拷贝**（TypedArray 视图）+ **SIMD**（`+simd128`）+ **并行**（`wasm-bindgen-rayon`，需跨源隔离 COOP/COEP）。
3. **体积与健壮**：`panic=abort`、LTO、`opt-level=z/s`、KTX2/Draco 等资源压缩；`console_error_panic_hook` 定位崩溃。

---

## 1) 基础脚手架（wasm-pack）🧱

### 1.1 新建 crate
```bash
cargo new image-lab --lib
cd image-lab
```

`Cargo.toml`
```toml
[package]
name = "image_lab"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]   # 关键：生成 wasm 供外部调用

[dependencies]
wasm-bindgen = "0.2"
# 调试更友好（可选）
console_error_panic_hook = "0.1"
wee_alloc = { version = "0.4", optional = true }

[features]
default = []
small-alloc = ["wee_alloc"]

[profile.release]
lto = "fat"
codegen-units = 1
opt-level = "z"     # 或 s：体积与速度权衡
panic = "abort"     # 省体积
```

`src/lib.rs`
```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen(start)]
pub fn start() {
    #[cfg(debug_assertions)]
    console_error_panic_hook::set_once();
}

#[wasm_bindgen]
pub fn sum(a: u32, b: u32) -> u32 {
    a + b
}
```

### 1.2 构建
```bash
# 安装 wasm-pack（如未安装）
cargo install wasm-pack
# 目标 web（也可 bundler/nodejs）
wasm-pack build --target web --release
```

输出在 `pkg/`：包含 `*.wasm` 与包装的 `*.js`。

---

## 2) 前端集成（Vite / Webpack）🔌

### 2.1 Vite（推荐）
```ts
// main.ts
import init, { sum } from './pkg/image_lab.js'; // wasm-pack 产物
await init();  // 异步初始化
console.log(sum(3, 9));
```

`vite.config.ts` 一般无需特殊配置；若要线程（SAB），需**跨源隔离**（见 §6.2）。

### 2.2 Web Worker + Offscreen（计算不阻塞 UI）
```ts
// worker.ts
import init, { heavy } from './pkg/image_lab.js';
self.onmessage = async (e) => {
  if (e.data.type === 'init') await init();
  if (e.data.type === 'run') {
    const out = heavy(e.data.n);
    postMessage(out);
  }
};
```

---

## 3) JS ↔ Rust 互操作：别被“边界成本”坑了 🔁

### 3.1 传文本 vs 传二进制
- 文本（`String`）跨边界**要拷贝**，尽量少传、传短。
- 图像/音频等**用 TypedArray**：`&[u8]`/`&mut [u8]` 在 Rust 侧零拷贝读取/写入。

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn grayscale_inplace(buf: &mut [u8]) {
    // 假设 RGBA
    for px in buf.chunks_mut(4) {
        let g = (0.299*px[0] as f32 + 0.587*px[1] as f32 + 0.114*px[2] as f32) as u8;
        px[0] = g; px[1] = g; px[2] = g;
    }
}
```

```ts
// JS：把 ImageData.data 直接传给 Rust（零拷贝视图）
import { grayscale_inplace } from './pkg/image_lab.js';
const ctx = canvas.getContext('2d')!;
const img = ctx.getImageData(0,0,w,h);
grayscale_inplace(img.data); // in-place 修改
ctx.putImageData(img, 0, 0);
```

> 关键点：**就地修改** + **一次调用处理尽可能多的数据**，减少来回穿梭。

### 3.2 结构化数据
- 数组/切片：用 `Vec<T>` 返回（会拷贝），或**传入缓冲区**让 Rust 填充。
- 对象：在 JS 构造 JSON，Rust 用 `serde_wasm_bindgen` 解析（注意开销）。

---

## 4) SIMD：打开“并行寄存器”🚀

### 4.1 开启方式
- 编译开启 `+simd128`（现代浏览器普遍支持 WebAssembly SIMD）。
```bash
RUSTFLAGS="-C target-feature=+simd128" wasm-pack build --target web --release
```

### 4.2 实战：SIMD 灰度（简化示例）
```rust
#[cfg(target_feature = "simd128")]
use core::arch::wasm32::*;

#[wasm_bindgen]
pub fn grayscale_simd(buf: &mut [u8]) {
    let len = buf.len() & !15; // 每次处理16字节（4像素 RGBA）
    let w_r = f32x4_splat(0.299);
    let w_g = f32x4_splat(0.587);
    let w_b = f32x4_splat(0.114);

    let ptr = buf.as_mut_ptr();
    let mut i = 0;
    while i < len {
        unsafe {
            let p = v128_load(ptr.add(i) as *const v128);
            // 解包 RGBA → 分量（示范思路，真实代码需 shuffle 提取 rgb）
            let r = f32x4_convert_i32x4(i32x4_extend_low_u16x8(u16x8_swizzle(p, i8x16_splat(0))));
            let g = f32x4_convert_i32x4(i32x4_extend_low_u16x8(u16x8_swizzle(p, i8x16_splat(1))));
            let b = f32x4_convert_i32x4(i32x4_extend_low_u16x8(u16x8_swizzle(p, i8x16_splat(2))));
            let gray = f32x4_add(f32x4_mul(r, w_r), f32x4_add(f32x4_mul(g, w_g), f32x4_mul(b, w_b)));
            // …再 pack 回 u8 并写回 p
            // 这里省略细节：生产建议用 crate `wide` / `simdeez` 或直接像素块循环 + LUT
        }
        i += 16;
    }
}
```

> 小结：**SIMD 有门槛**，但在图像/信号/向量计算上往往能给出 **2–8×** 提升。也可以用 **`std::simd`（portable_simd）**（需 nightly 或稳定版对应进度）。

---

## 5) 多线程（Rayon）与跨源隔离 🧵

### 5.1 何时用
- 大数组并行（排序/卷积/聚合）、路径规划/物理仿真、解析/压缩等 CPU 密集场景。

### 5.2 启用步骤
1. 服务器/本地 Dev 开**跨源隔离**（否则 SharedArrayBuffer 不可用）：  
   - 响应头：`Cross-Origin-Opener-Policy: same-origin`  
   - `Cross-Origin-Embedder-Policy: require-corp`  
2. 依赖：`wasm-bindgen-rayon`。
3. 初始化线程池（JS 调用一次）。

`Cargo.toml`
```toml
[dependencies]
wasm-bindgen = "0.2"
wasm-bindgen-rayon = "1.2"
rayon = "1.10"
```

`src/lib.rs`
```rust
use wasm_bindgen::prelude::*;
use rayon::prelude::*;

#[wasm_bindgen]
pub async fn init_threads(pool_size: Option<usize>) -> Result<(), JsValue> {
    wasm_bindgen_rayon::init_thread_pool(pool_size.unwrap_or_else(num_cpus::get))
        .await
        .map_err(|e| JsValue::from_str(&format!("{e:?}")))
}

#[wasm_bindgen]
pub fn sum_parallel(data: &[u32]) -> u64 {
    data.par_iter().map(|&x| x as u64).sum()
}
```

```ts
// JS
import init, { init_threads, sum_parallel } from './pkg/image_lab.js';
await init();
await init_threads(navigator.hardwareConcurrency); // 开 Worker 池
const arr = new Uint32Array(1e7);
console.log(sum_parallel(arr)); // 并行求和
```

> 多线程 + SIMD 叠加可达**数量级加速**；记得在 UI 上给**“低性能模式”**开关。

---

## 6) 体积优化与构建事项 📦

### 6.1 编译选项
- `panic=abort`、`lto="fat"`、`codegen-units=1`、`opt-level=z`（体积）/`s`（速度）。  
- 去除符号：`wasm-opt -Oz -g0`（Binaryen），生产再跑 `-O4` 探索；避免过度会破坏调试。

### 6.2 线程与 SAB（重要）
- 要用多线程：**必须** COOP/COEP。Vite 示例：
```ts
// vite.config.ts
export default defineConfig({
  server: {
    headers: {
      'Cross-Origin-Opener-Policy': 'same-origin',
      'Cross-Origin-Embedder-Policy': 'require-corp'
    }
  }
});
```

### 6.3 Source Map 与崩溃
- 开发留 `-g`，生产去；用 `console_error_panic_hook` 打印 Rust panic 栈。  
- JS 侧给 **降级**（比如无 SIMD/线程 → 单线程路径）。

---

## 7) 图像/视频处理管线（实战蓝本）🧪🖼️

### 7.1 Rust：卷积核（边缘检测/模糊）
```rust
#[wasm_bindgen]
pub fn convolve_rgba(src: &[u8], dst: &mut [u8], w: usize, h: usize, kernel: &[f32], k: usize) {
    let khalf = k / 2;
    for y in 0..h {
        for x in 0..w {
            let mut acc = [0.0f32; 3];
            for ky in 0..k {
                for kx in 0..k {
                    let ix = x as isize + kx as isize - khalf as isize;
                    let iy = y as isize + ky as isize - khalf as isize;
                    if ix < 0 || iy < 0 || ix >= w as isize || iy >= h as isize { continue; }
                    let idx = (iy as usize * w + ix as usize) * 4;
                    let wgt = kernel[ky * k + kx];
                    acc[0] += src[idx] as f32 * wgt;
                    acc[1] += src[idx+1] as f32 * wgt;
                    acc[2] += src[idx+2] as f32 * wgt;
                }
            }
            let o = (y * w + x) * 4;
            dst[o]   = acc[0].clamp(0.0,255.0) as u8;
            dst[o+1] = acc[1].clamp(0.0,255.0) as u8;
            dst[o+2] = acc[2].clamp(0.0,255.0) as u8;
            dst[o+3] = 255;
        }
    }
}
```

```ts
// JS：TypedArray 开两个缓冲区，避免反复分配
import { convolve_rgba } from './pkg/image_lab.js';
const img = ctx.getImageData(0,0,w,h);
const out = new Uint8ClampedArray(img.data.length);
const K = new Float32Array([0,-1,0,-1,5,-1,0,-1,0]); // 锐化
convolve_rgba(img.data, out, w, h, K, 3);
ctx.putImageData(new ImageData(out, w, h), 0, 0);
```

### 7.2 与 WebCodecs/OffscreenCanvas 组合
- `VideoFrame` → `copyTo` 得到 `Uint8Array` → Rust 处理 → 新建 `VideoFrame` 或绘制到 `OffscreenCanvas`。  
- 放在 Worker 里做：**零阻塞 UI**，配合 §5 的并行/§4 的 SIMD。

---

## 8) 测量与基准（别靠感觉）⏱️

```ts
const t0 = performance.now();
grayscale_inplace(img.data);
const t1 = performance.now();
console.log('wasm ms=', t1 - t0);

// 比较 JS 版本 / SIMD 版本 / 并行版本；跑 10 次取中位数；预热一次。
```

指标建议：**吞吐（MB/s）**、**每帧耗时**、**p95**、**CPU 占用**、**能耗（移动端可粗略看温度/降频）**。

---

## 9) 错误处理与 API 设计 🧩

- Rust 侧返回 `Result<T, JsValue>`；出错时抛 JS 异常。  
- 公共 API **小而稳**：让一次调用处理尽可能多的数据，暴露纯函数式接口（方便测试与并行）。  
- 内部状态（缓存/工作缓冲区）在 Rust 侧**长驻**，暴露 `init(capacity)` 等方法。

---

## 10) 反模式与纠偏 🧨
| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 高频小函数跨边界 | CPU 时间花在胶水层 | 合并调用，批量处理 |
| 全靠返回 `Vec` | 大量复制/GC 压力 | 传入可写缓冲区，就地写 |
| 没开 SIMD/线程 | 性能打折 | `+simd128` + `wasm-bindgen-rayon` |
| 无跨源隔离却想并行 | 初始化报错 | 配 COOP/COEP（开发/生产都要） |
| panic 默认 | 体积大且难排错 | `panic=abort` + `console_error_panic_hook`（仅开发） |
| 过度优化 `-O4` | 调试灾难 | 分环境：开发保符号，生产再压 |

---

## 11) 验收清单 ✅
- [ ] `wasm-pack build --target web --release` 产物稳定加载（异步 init）。  
- [ ] API 接口以 **TypedArray in/out** 为主，跨边界调用**批量化**。  
- [ ] SIMD 开启并通过 feature detect 降级；有基准数据。  
- [ ] Rayon 线程池在 **COOP/COEP** 环境下可用；低端机有关闭并行的开关。  
- [ ] 体积优化：`panic=abort`、LTO、`wasm-opt -Oz`，SourceMap 只在开发。  
- [ ] 监控：关键路径耗时、p95、错误率上报。

---

## 12) 练习 🏋️
1. 用 Rust 实现 **Sobel 边缘**与**高斯模糊**，提供 JS 与 Wasm 版本基准对比（p50/p95）。  
2. 打开 **SIMD + Rayon**，在 4k 图像上对比单线程耗时；做“低性能模式”自适应开关。  
3. 把摄像头流接 **WebCodecs** + Rust 卷积，端到端延迟控制在 **< 16ms**。  
4. 用 `wasm-bindgen` 自定义 **内存池**（复用 `Vec<u8>`），统计 GC/alloc 次数变化。

---

**小结**：Rust+Wasm 的价值，不在“能不能做”，而在“**做同样的事更快更稳**”。记住三件事：**边界少**、**数据块大**、**硬件吃满**（SIMD/线程），再把工程化（构建/体积/监控）补齐，你的前端重计算就能跑得又冷又快。🧊🏎️
