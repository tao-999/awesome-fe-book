# 16.2 资源策略：图片 / 字体 / 预取 🖼️🔤🚀

目标：用一套**可复制**的工程策略，让图片、字体与预取做到——**少而精、按需来、尽早到、稳布局、好缓存**。口号：**把带宽花在刀刃上**。

---

## 0) 总原则（先立规矩）

- **就近与可缓存**：CDN/Edge 下发；文件名用**内容指纹**（`.[hash].*`）+ `Cache-Control: public, max-age=31536000, immutable`。
- **优先级**：LCP 资源使用 `fetchpriority="high"` 或 `<link rel="preload">`；其余懒加载或可见再拉。
- **稳布局**：所有媒体都给 `width/height` 或 `aspect-ratio`，广告/骨架**预留槽位**，抑制 CLS。
- **现代格式**：图片优先 AVIF/WebP；字体 WOFF2；动图尽量用视频（WebM/MP4）。
- **可观测**：RUM 上报 LCP 资源名/体积/TTFB、预取命中率；DevTools `Coverage` 清未用代码与字形。

---

## 1) 图片策略（从“像素”到“管线”）🖼️

### 1.1 格式选择

- **优先级**：AVIF → WebP → JPEG/PNG（少量需要透明/锐利边缘场景保留 PNG）。
- **动图**：用 `<video muted autoplay loop playsinline>` 代替 GIF，并设置 `poster`。
- **图标**：矢量优先（SVG），可内联以减少请求与保证颜色/大小可控。

### 1.2 响应式与密度（`<picture>` / `srcset` / `sizes`）

~~~html
<picture>
  <source type="image/avif"
          srcset="/img/hero-800.avif 800w, /img/hero-1200.avif 1200w, /img/hero-1600.avif 1600w"
          sizes="(max-width: 800px) 100vw, 1200px">
  <source type="image/webp"
          srcset="/img/hero-800.webp 800w, /img/hero-1200.webp 1200w, /img/hero-1600.webp 1600w"
          sizes="(max-width: 800px) 100vw, 1200px">
  <img src="/img/hero-1200.jpg" alt="主视觉"
       width="1200" height="630"
       decoding="async" fetchpriority="high">
</picture>
~~~

要点：LCP 图高优先级；**一定给尺寸/比例**锁盒。

### 1.3 懒加载与解码

~~~html
<img src="/img/card.webp" alt="示例"
     width="400" height="300"
     loading="lazy" decoding="async">
~~~

- 原生 `loading="lazy"` 足够；复杂联动再用 IntersectionObserver。
- `decoding="async"` 降低主线程阻塞，可配 `content-visibility: auto` 进一步减负。

### 1.4 CDN 变体与 DPR

- 通过查询生成变体：`/img/hero.jpg?w=1200&dpr=2&q=60`；服务端按 `Accept` 协商 AVIF/WebP。
- 前端仍用 `srcset/sizes` 让浏览器**自主挑选**最优资源。

### 1.5 占位与骨架

- LQIP/BlurHash/SQIP：用极小图或矢量占位避免空白闪烁。
- 不要用 JS 动态量尺寸后再渲染——那是 CLS 制造机。

### 1.6 构建与门禁（管线）

- 构建期用 **Sharp / imagetools** 批量产出多尺寸/多格式；产出 manifest 给模板消费。
- CI 规则：
  - 单图体积 ≤ **400KB**（海报/壁纸可白名单）。
  - **缺少尺寸/比例**的 `<img>/<video>` **直接拒绝合并**。

---

## 2) 字体策略（既要颜值也要速度）🔤

### 2.1 选择与回退

正文优先系统字体栈：

~~~css
font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial, "Noto Sans",
  "Apple Color Emoji", "Segoe UI Emoji";
~~~

标题/品牌需要定制字体：只保留必要字重或使用**可变字体**（Variable Font）。

### 2.2 自托管与子集化

- 产出 **WOFF2**（必要才回退 WOFF），不要让 OTF/TTF 直接上生产。
- **子集化**字形，或用 `unicode-range` 分区（latin / latin-ext / zh-CN）。

### 2.3 FOIT/FOUT 与度量对齐

~~~css
@font-face{
  font-family: Inter;
  src: url(/fonts/inter-var.woff2) format('woff2');
  font-display: swap;        /* 首屏用系统字，下载完再替换 */
  font-weight: 100 900;
  font-style: normal;
  size-adjust: 98%;          /* 对齐回退字体度量，降低 CLS */
  ascent-override: 90%;
  descent-override: 22%;
  line-gap-override: 0%;
}
~~~

- `font-display: swap` 避免 FOIT；配合 `size-adjust`/metrics override 把回流减到最小。

### 2.4 谨慎预加载

~~~html
<link rel="preload" as="font" href="/fonts/inter-var.woff2" type="font/woff2" crossorigin>
~~~

- **仅**对首屏立刻用、且**文字是首屏主角**的字体预加载；滥用会抢带宽，拖慢 LCP。
- 第三方（Google Fonts）建议**下载自托管**，缓存与可用性更可控。

---

## 3) 预取 / 预加载 / 连接预热（优先级就是体验）⛽

### 3.1 术语速查

| Hint | 含义 | 用途 | 示例 |
|---|---|---|---|
| `preconnect` | 预建 TCP/TLS | 跨域关键源 | `<link rel="preconnect" href="https://cdn.example.com" crossorigin>` |
| `dns-prefetch` | 仅做 DNS | 补充级 | `<link rel="dns-prefetch" href="//cdn.example.com">` |
| `preload` | **当前页**必需资源提前拉 | LCP 图/关键 CSS/JS/字体 | `<link rel="preload" as="image" href="/img/hero.avif">` |
| `modulepreload` | 预拉 ESM 模块依赖 | 分包模块 | `<link rel="modulepreload" href="/assets/chunk-abc.js">` |
| `prefetch` | **下一页**可能用资源 | 空闲低优先级 | `<link rel="prefetch" as="script" href="/assets/search.js">` |
| Speculation Rules | 未来页面**预渲染/预取** | 高概率跳转 | 见下文 |

### 3.2 正确使用 `preload`

- **只**对首屏必用资源使用，否则会**抢带宽**。
- 与实际使用**一一对应**（避免 DevTools “预加载未使用”警告）。

~~~html
<link rel="preload" as="image"
      href="/img/hero-1200.avif"
      imagesrcset="/img/hero-800.avif 800w, /img/hero-1200.avif 1200w, /img/hero-1600.avif 1600w"
      imagesizes="(max-width: 800px) 100vw, 1200px">
~~~

### 3.3 Speculation Rules（Prerender/Prefetch）

~~~html
<script type="speculationrules">
{
  "prerender": [{"source": "list", "where": {"href_matches": "*/product/*"}}],
  "prefetch":  [{"source": "document"}]
}
</script>
~~~

- 列表→详情等**高概率**导航可启用；**涉及鉴权/副作用**的页面谨慎，必要时做**无状态资源预取**而非 prerender。

### 3.4 Early Hints（HTTP 103）

- 源站/边缘先发 103 告知可预加载的关键资源（LCP 图/CSS/字体），抢在 HTML 生成前启动下载。  
- 若平台不支持，至少把 `<link rel="preload">` 放在 `<head>` **最前**。

---

## 4) 缓存与协商（让好内容多活一会）🧊

### 4.1 不可变指纹资源

~~~text
Cache-Control: public, max-age=31536000, immutable
ETag: "sha256:..."
~~~

### 4.2 可更新资源（CMS 图、裁剪）

~~~text
Cache-Control: public, max-age=60, stale-while-revalidate=600
ETag: "p_123@rev_42"
~~~

- 浏览器带 `If-None-Match` 命中 304 省流量；CDN 用 `s-maxage` 控制共享缓存时长。
- 变体（尺寸/格式）**各自指纹**，精准失效。

---

## 5) 与框架平台的正确姿势 🧰

- **Next.js**：`next/image`（自动格式/尺寸/占位、`priority`=LCP）、`next/font`（子集与自动 preload）、`Head` 管理 `preconnect/preload`。
- **Nuxt**：`@nuxt/image`、`@nuxt/fonts`、`routeRules` ISR；内容站优先静态化。
- **Astro**：`astro:assets` 产图，`.astro` 输出静态结构；交互岛 `client:visible/idle`。
- **SvelteKit**：`vite-imagetools`/生态组件；首屏资源在布局预声明。

---

## 6) 监控与验收（没有数据就没有优化）📈

- **RUM**：上报 `lcp.resource.name / size / ttfb / transferSize`、`cls` 来源元素、预取/预加载命中率。
- **实验室**：Lighthouse/WebPageTest；DevTools `Performance` 看 LCP 时刻与阻塞脚本；`Coverage` 找未用字形/脚本。
- **阈值**：LCP 图 > **300KB** 或下载 > **1.0s** 告警；“预加载未使用” > **5%** 需要清理。

---

## 7) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 图片无尺寸/比例 | CLS 飙升 | `width/height` 或 `aspect-ratio` |
| 滥用 `preload` | 关键资源变慢 | 仅首屏必用资源用 `preload` |
| 用 GIF 播长动画 | 带宽/CPU 爆炸 | `<video>`（WebM/MP4）+ `poster` |
| 字体托管第三方 | 首屏慢/不可控 | **自托管** + WOFF2 + 子集化 |
| 权重/字体过多 | 体积爆炸 | 可变字体或仅 2–3 个权重 |
| 预取一堆没人点 | 浪费流量 | 基于命中率动态名单/阈值治理 |
| 构建不产变体 | 移动端流量高 | 构建期多尺寸/多格式 + `srcset/sizes` |

---

## 8) 提交前检查清单 ✅

- [ ] LCP 图已 `fetchpriority="high"`；必要时 `<link rel="preload" as="image">`；并提供 `srcset/sizes` 与现代格式。  
- [ ] 所有媒体**有固定尺寸或比例**；顶部不插入新块。  
- [ ] 字体自托管 WOFF2，**子集化**；`font-display: swap` 与 `size-adjust` 完整配置。  
- [ ] 合理使用 `preconnect/dns-prefetch`、`preload`、`prefetch`、`modulepreload`，无“预加载未使用”警告。  
- [ ] 静态指纹资源 `immutable`；动态资源 `s-maxage + stale-while-revalidate`；ETag 协商有效。  
- [ ] RUM/看板已显示 LCP 资源体积、预取命中率与失败比；超阈值会告警。  

---

## 9) 练习 🏋️

1. 把首页英雄图改为 **AVIF + `fetchpriority`**，记录 LCP 改善。  
2. 为产品详情页实现 `<picture>` + `srcset/sizes`，对比移动端流量下降与 INP 是否受影响。  
3. 给首屏字体做 **子集化 + `size-adjust`**，观察 CLS 变化与传输字节差异。  
4. 使用 **Speculation Rules** 对“列表→详情”做预渲染，并监控“预渲染命中率”与失败回退。  

---

**小结**：把**优先级（preload/fetchpriority）**、**格式（AVIF/WOFF2）**、**布局（尺寸/比例）**、**缓存（immutable/SWR/ETag）**四根支柱立稳，你的页面会**更快、更稳、更省**。🔥
