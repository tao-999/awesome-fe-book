# 13.2 Astro / 岛屿架构 🏝️⚡

本章目标：搞懂 **岛屿架构（Islands Architecture）** 与 **Astro** 的“**零 JS by default** + **按需水合**”哲学，给出**最小可跑模板**、**性能心法**与**实战清单**。一句话：**静态为主，交互为辅**；让页面是 **MPA** 的稳与快，交互像 **SPA** 一样丝滑。

---

## 0) TL;DR

- **岛屿 = 交互组件，只在需要的地方上 JS**；页面其余部分是纯 HTML（几乎零 JS）。  
- **Astro**：天然岛屿框架，内置 **部分水合（Partial Hydration）** 指令：`client:load/idle/visible/media/only`。  
- 用 Astro 写**内容型站点/文档/营销页/电商详情**，在按钮、购物车、搜索框这类“岛屿”上接 **React/Vue/Svelte** 等组件。  
- 性能赢点：更少 JS → **更好的 INP/TBT/LCP**，SEO 直接吃满；需要 SSR 时再上 **适配器（Node/Edge/Vercel/CF）**。

---

## 1) 岛屿架构一张图

```
(HTML=静态海面) ──────────────────────────────────────────
  ▣ 搜索框(React, client:idle)
            ▣ 购物车按钮(Vue, client:visible)
                          ▣ 评分控件(Svelte, client:load)
─────────────────────────────────────────────────────────
   ↑ 大部分内容是 SSR/SSG 的纯标记；只有岛屿上 JS 才下载/执行
```

**关键点**
- **解耦**：岛屿内部由各自框架管理状态；页面层是框架无关的静态 HTML。  
- **渐进增强**：没 JS 也能看内容；有 JS 才有交互。  
- **按触发加载**：`visible/idle/media` 控制水合时机，省电省流量。

---

## 2) Astro 基本盘（目录与路由）

```
my-site/
  src/
    pages/
      index.astro           # / 首页
      blog/[slug].astro     # 动态路由
      api/hello.ts          # Endpoint（GET/POST 返回 Response）
    components/
      Counter.jsx           # 岛屿组件（React/Vue/Svelte 任你选）
    layouts/
      BaseLayout.astro
  astro.config.mjs
  package.json
```

- **文件路由**：`src/pages/**` 即路由；`[param]` 动态；`.astro/.md/.mdx/.ts` 均可。  
- **Endpoint**：`.ts/.js` 导出 `GET/POST` 返回 `Response`。  
- **中间件**：`src/middleware.(ts|js)` 可拦截请求（重写、鉴权、i18n）。

---

## 3) 最小可跑示例（Astro + React 岛）

**安装**
```bash
pnpm create astro@latest
# 选 "Empty" 或 "Blog", 语言 TS, 包管理器 pnpm
pnpm astro add react mdx sitemap
pnpm dev
```

**`src/layouts/BaseLayout.astro`**
```astro
---
const { title = 'My Astro Site' } = Astro.props;
---
<html lang="zh-CN">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <title>{title}</title>
  </head>
  <body>
    <header><a href="/">🏝️ Astro</a></header>
    <main><slot /></main>
    <footer>© {new Date().getFullYear()}</footer>
  </body>
</html>
```

**`src/components/Counter.jsx`**
```jsx
import { useState } from 'react';
export default function Counter() {
  const [n, set] = useState(0);
  return <button onClick={() => set(n + 1)}>点击 +1（{n}）</button>;
}
```

**`src/pages/index.astro`**
```astro
---
import BaseLayout from '../layouts/BaseLayout.astro';
import Counter from '../components/Counter.jsx';
const posts = [{ slug: 'hello-astro', title: 'Hello Astro' }];
---
<BaseLayout title="Astro 岛屿示例">
  <h1>静态是基座，交互是岛屿</h1>
  <ul>
    {posts.map(p => <li><a href={`/blog/${p.slug}`}>{p.title}</a></li>)}
  </ul>

  <!-- 岛屿：仅在空闲时水合，减轻首屏 TBT/INP -->
  <Counter client:idle />
</BaseLayout>
```

**`src/pages/api/hello.ts`**
```ts
export const GET = () =>
  new Response(JSON.stringify({ ok: true }), { headers: { 'content-type': 'application/json' } });
```

跑起来：`pnpm dev` → `http://localhost:4321`。

---

## 4) 部分水合指令速查 🧪

- `client:load`：页面加载后立即水合（最急）。  
- `client:idle`：浏览器空闲时水合（推荐默认）。  
- `client:visible`：组件进入视口才水合（懒加载互动）。  
- `client:media="(min-width:900px)"`：媒体查询条件满足才水合（响应式差异）。  
- `client:only="react|vue|svelte"`：**纯客户端组件**，不参与 SSR（如严重依赖 `window` 的库）。

> 心法：**能晚不早，能局部不全局**。把 JS 寄在用户真正会用到的交互上。

---

## 5) 内容为王：Content Collections + MDX

**类型安全的内容集合**
```
src/content/
  config.ts            # 定义集合与 Zod Schema
  blog/                # Markdown/MDX 文章
    hello-astro.mdx
```

**`src/content/config.ts`**
```ts
import { defineCollection, z } from 'astro:content';

const blog = defineCollection({
  type: 'content',
  schema: z.object({
    title: z.string(),
    date: z.string().transform(v => new Date(v)),
    tags: z.array(z.string()).default([])
  })
});

export const collections = { blog };
```

页面读取：
```astro
---
import { getCollection } from 'astro:content';
const posts = await getCollection('blog');
---
<ul>
  {posts.map(p => <li><a href={`/blog/${p.slug}`}>{p.data.title}</a></li>)}
</ul>
```

> 好处：**构建期校验 Frontmatter**，拿到是**已校验的数据**，避免线上踩雷。

---

## 6) 资源与优化（Images / Fonts / 转场）

- 图片：用 `astro:assets`（或 `@astrojs/image`）生成不同尺寸与格式，`<img loading="lazy">` 默认友好。  
- 字体：优先 **本地子集化**（`fonttools`/`subfont`/或 Vite 插件），减少 CLS。  
- 视图转场：接入 **View Transitions** 插件（官方）可在 MPA 间实现 SPA 级页面切换动画。  
- 预取：导航链接加 `rel="prefetch"` 或使用集成的链接预取（Intersection + hover）。

---

## 7) SSR / SSG / 混合部署

- 默认 **SSG**（静态导出）：最省钱最快。  
- 需要实时数据时启用 **SSR 模式**，选适配器：
  - `@astrojs/node`（Node）、`@astrojs/vercel`、`@astrojs/netlify`、`@astrojs/cloudflare` …  
- **混合**：大多数页面静态化，少量路由走 SSR；端点用于 BFF 小能力。

**`astro.config.mjs` 示例**
```js
import { defineConfig } from 'astro/config';
import react from '@astrojs/react';
import mdx from '@astrojs/mdx';

export default defineConfig({
  integrations: [react(), mdx()],
  output: 'hybrid',              // 'static' | 'server' | 'hybrid'
  server: { host: true }
});
```

---

## 8) 多框架共存？可以，但要克制

Astro 可同时挂 **React/Vue/Svelte** 岛屿；但：
- 每种 UI 框架都要带上自己的运行时 → **JS 体积叠加**。  
- 统一选型一个主 UI 框架，其他只在**不可替代**时再引入。  
- 岛屿内尽量**独立**，避免跨岛通信（跨岛 = 跨框架 = 复杂度）。

---

## 9) 性能与可维护性心法 ⚙️

- **优先用 `.astro` 组件写静态结构**；只有交互才上 React/Vue/Svelte。  
- **把状态封装在岛里**，页面负责数据喂养与 SEO。  
- **水合策略分层**：首屏关键交互 `idle`，非首屏 `visible`，重组件 `media`。  
- **小岛原则**：一个岛只做一件事（按钮/卡片/搜索框），小而可替换。  
- 结合 **SW/缓存**：内容走 SSG + `stale-while-revalidate`，交互数据用 SWR 策略。

---

## 10) 与其他“岛屿派”一瞥

- **Qwik**：细粒度恢复（Resumability），比“水合”更激进（执行更少）。  
- **Fresh (Deno)**：服务器首屏 + 岛屿，跑在 Deno/Edge。  
- **Marko**：指令化的分块水合，电商站点常用。  
- Astro 的定位更像 **“岛屿集成器 + 构建器”**，偏内容/营销/文档站，也能做中小电商与门户。

---

## 11) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 页面全是岛 | JS 反而比 SPA 还大 | 用 `.astro` 输出静态结构；岛只包交互点 |
| `client:load` 滥用 | 首屏 TBT/INP 爆炸 | 改 `idle/visible/media`，分级水合 |
| 多框架堆满 | 运行时重复、体积激增 | 统一框架；必要时再引入小岛 |
| 岛间共享全局状态 | 难以调试、耦合上天 | 岛内自治；通信靠事件/端点，不共享可变单例 |
| 忽视无 JS 场景 | 可访问性/SEO 降级 | 保证无 JS 也有可用体验；表单有后备提交 |
| SSG 全靠构建期取数 | 内容陈旧 | 混合：SSG + Endpoint + ISR/再验证 |

---

## 12) 提交前检查清单 ✅

- [ ] 非交互区域使用 `.astro`，减少 JS 传输。  
- [ ] 岛屿水合策略合理（`idle/visible/media/only`），无不必要 `client:load`。  
- [ ] 图片通过 `astro:assets` 或等价方案做尺寸/格式优化。  
- [ ] 内容集合定义了 **Zod Schema**，Frontmatter 通过校验。  
- [ ] SSR/SSG/混合模式选择清晰，适配器配置正确。  
- [ ] 没 JS 也可完成主要任务（降级策略 OK）。  

---

## 13) 练习 🏋️

1. 用 Astro 搭一个**博客首页**：文章列表（SSG）+ 右侧“订阅”小组件（`client:visible`）。  
2. 把“加入购物车”做成独立 **React 岛**，水合策略设 `client:idle`；测首屏 INP 相较 SPA 的提升。  
3. 将首屏图片换成 `astro:assets` 管线，输出 `avif/webp` 多格式与 `srcset`，对比 LCP 变化。  

---

**小结**：Astro 与岛屿架构的威力在于一句朴素真理——**用户浏览网页首先是“看内容”**。把 JS 留给真正的交互，把静态做到极致，你的站点会又快又稳，还优雅。🏝️🚀
