# 1.3 SEO 与爬虫友好实践

本章目标：给站点打好**可索引、可理解、可排名**的基础盘。覆盖：可抓取（crawl）→ 可渲染（render）→ 可索引（index）→ 可评估（rank）的全链路做法；并提供可直接拷贝的模板。

---

## 1) 三个核心观念

- **先内容、后技术**：信息结构与可读性是排名地基；技术优化是加分项。  
- **HTML 可读、URL 可判、状态码可信**：机器人看到的应与用户看到的等价。  
- **以“章节语义 + 结构化数据 + 性能”驱动**：让爬虫更快理解你的页面。

---

## 2) On-Page 基线（标题、描述、语义）

**最小合格头部（可拷贝）**：
```html
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>页面标题 | 站点名</title>
  <meta name="description" content="80–160 字符的准确摘要，包含主体关键词与用户意图。" />
  <link rel="canonical" href="https://example.com/path" />
  <meta name="robots" content="index,follow,max-snippet:-1,max-image-preview:large,max-video-preview:-1" />
</head>
```

实践要点：
- 每页仅一个 **H1**；层级不跳级（见 1.1 章节）。  
- 图片 **alt** 准确，非装饰性图像必须可读；视频有字幕轨。  
- 关键信息在首屏 HTML 中可见（别把主体内容完全塞进延迟加载的 JS）。

---

## 3) URL、状态码与规范化（Canonical）

- **URL 设计**：小写、连字符 `-` 分词、稳定且可读：`/guide/seo-basics/`。  
- **一种内容一个 URL**：HTTP/HTTPS、带不带 `www`、末尾斜杠统一到**首选版本**，其它做 `301/308`。  
- **规范化**：每页自指向 `<link rel="canonical" href="...">`。含参数页面（如 `?utm=...`）同样指回首选。  
- 错误页用 **404/410**；永久移除用 **410**；临时维护用 **503 + Retry-After**。  
- 重复列表页慎用 “view-all canonical”；分页页彼此自 canonical（**不再使用** `rel="prev/next"` 作为信号，但内部链接仍应存在）。

---

## 4) 抓取控制：robots.txt / Robots 指令 / X-Robots-Tag

**robots.txt（站点根目录）**：
```
User-agent: *
Disallow: /admin/
Allow: /assets/
Sitemap: https://example.com/sitemap-index.xml
```

要点：
- `Disallow` 只是**限制抓取**，**不能**保证不被收录。想不收录，用页面 `<meta name="robots" content="noindex">` 或 **HTTP 头 X-Robots-Tag**。  
- 二进制文件（PDF/图像）用 **X-Robots-Tag** 阻止索引（响应头）：  
  ```
  X-Robots-Tag: noindex, noarchive
  ```

---

## 5) Sitemap 策略

- **大小限制**：单个 sitemap ≤ 50,000 URL 或 50MB（未压缩）。多文件时用 **sitemap index**。  
- 强烈建议提供 `lastmod`，CMS 更新时同步刷新。  
- 如果图片多，可在页面 sitemap 中加入 `<image:image>`；新闻站点可单独 news sitemap。

**示例：索引文件**
```xml
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap><loc>https://example.com/sitemaps/pages.xml</loc><lastmod>2025-09-15</lastmod></sitemap>
  <sitemap><loc>https://example.com/sitemaps/posts.xml</loc><lastmod>2025-09-15</lastmod></sitemap>
</sitemapindex>
```

**示例：页面 sitemap 片段**
```xml
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://example.com/guide/seo-basics/</loc>
    <lastmod>2025-09-15</lastmod>
  </url>
</urlset>
```

---

## 6) 结构化数据（JSON-LD 模板）

**Article / BlogPosting（内容页）**：
```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BlogPosting",
  "headline": "SEO 与爬虫友好实践",
  "datePublished": "2025-09-15",
  "dateModified": "2025-09-15",
  "author": { "@type": "Person", "name": "作者名" },
  "publisher": { "@type": "Organization", "name": "站点名" },
  "mainEntityOfPage": { "@type": "WebPage", "@id": "https://example.com/guide/seo/" }
}
</script>
```

**面包屑**：
```html
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@type":"BreadcrumbList",
  "itemListElement":[
    {"@type":"ListItem","position":1,"name":"指南","item":"https://example.com/guide/"},
    {"@type":"ListItem","position":2,"name":"SEO 基础","item":"https://example.com/guide/seo/"}
  ]
}
</script>
```

**站内搜索建议（SiteLinks SearchBox）**：
```html
<script type="application/ld+json">
{
  "@context":"https://schema.org",
  "@type":"WebSite",
  "url":"https://example.com/",
  "potentialAction":{
    "@type":"SearchAction",
    "target":"https://example.com/search?q={query}",
    "query-input":"required name=query"
  }
}
</script>
```

> 只添加与你页面真实呈现匹配的类型；不滥加“富结果”类型。

---

## 7) 国际化与多语言（hreflang）

- 每个变体页互相声明 `hreflang`，并都包含 `x-default`。  
- 语言脚本/地区写全：`zh-Hans-CN`、`en-US`。

```html
<link rel="alternate" href="https://example.com/zh/" hreflang="zh-Hans-CN" />
<link rel="alternate" href="https://example.com/en/" hreflang="en-US" />
<link rel="alternate" href="https://example.com/" hreflang="x-default" />
```

> hreflang 与 canonical **同时存在**：每个变体自 canonical，互相通过 hreflang 关联。

---

## 8) 渲染策略（CSR/SSR/SSG 与无限滚动）

- **优先 SSR/SSG 或 ISR**：主内容在 HTML 首次响应里就出现；JS 仅用于增强与交互。  
- **避免“纯 CSR 首屏空壳”**：搜索引擎虽能执行 JS，但渲染有延迟与失败风险，且对多数非 Google 的爬虫并不可靠。  
- **无限滚动**：必须有可到达的分页 URL（`/page/2`），并在页面中放出分页链接；滚动只是增强层。  
- **社交抓取**（OG/Twitter bot）不执行或很少执行 JS，预览信息应在 **HTML 里**。  
- 不再推荐“动态渲染（给爬虫返回快照）”的旧黑科技；用**同构/预渲染**的现代方案更稳。

---

## 9) 社交卡片（提升分享点击率）

```html
<meta property="og:type" content="article" />
<meta property="og:title" content="SEO 与爬虫友好实践" />
<meta property="og:description" content="可抓取、可索引、可评估的落地做法与模板。" />
<meta property="og:url" content="https://example.com/guide/seo/" />
<meta property="og:image" content="https://example.com/og/seo.png" />
<meta name="twitter:card" content="summary_large_image" />
```

> 预览图建议 ≥ 1200×630，大小适中，主题清晰；与页面标题一致。

---

## 10) 性能与 Core Web Vitals

目标阈值（页面级）：**LCP < 2.5s、CLS < 0.1、INP < 200ms**。  
落地做法：
- 关键图片懒加载 + LCP 资源优先（`<link rel="preload">`）+ 正确尺寸。  
- **Critical CSS** 内联首屏样式，延后非关键 CSS/JS；字体 `font-display: swap`。  
- 使用 CDN、压缩（Brotli/gzip）、缓存（`Cache-Control`/`ETag`）。  
- 图片：AVIF/WebP 优先，`srcset/sizes` 响应式。  
- 监控：真实用户数据（RUM）上报 + 实验对照（见 Part V）。

---

## 11) 监控与运维

- 搜索引擎站长平台（Search Console/Bing Webmaster）提交 sitemap、监控抓取/索引、手动操作。  
- 服务器日志分析（按 UA、状态码、响应时间）看爬虫抓取效率与异常。  
- 重要模板的 **可抓取测试**：禁用 JS 的快照是否仍能看到主体内容与链接。

---

## 12) 反模式与修复

| 反模式 | 风险 | 修复 |
|---|---|---|
| 纯 CSR 首屏空壳 | 抓取失败/延迟索引 | 迁移 SSR/SSG/ISR；至少提供静态占位与首屏内容 |
| `noindex` 写在 robots.txt | 无效 | 用 `<meta name="robots" content="noindex">` 或 X-Robots-Tag |
| 重复内容无规范化 | 权重稀释 | 自 canonical；统一 301 到首选 URL |
| 仅无限滚动无分页链接 | 深层内容不可达 | 提供 `/page/n` 与内部链接 |
| 滥加结构化数据 | 违规或降权 | 仅标注真实呈现的信息 |
| 移除焦点/慢首屏 | 影响可用与 Vitals | 保留 `:focus-visible`；优化 LCP 与脚本分发 |

---

## 13) 发版前检查清单

- [ ] 每页：唯一 `title` + 合理 `description` + 自 canonical。  
- [ ] 主体内容在无 JS 时仍可见（至少可爬）。  
- [ ] robots.txt 存在且不误伤；敏感资源用 X-Robots-Tag 控制索引。  
- [ ] sitemap/index 最新，`lastmod` 正确。  
- [ ] 核心模板含必要 JSON-LD（Article/Breadcrumb/Organization 或 WebSite）。  
- [ ] OG/Twitter 卡片可预览。  
- [ ] Core Web Vitals 达标。  
- [ ] 重要路径返回正确状态码（200/301/404/410/503）。  

---

## 14) 快速模板集

**A. 文章页头部（精简版）**
```html
<title>标题 | 站点名</title>
<meta name="description" content="80–160 字符摘要。" />
<link rel="canonical" href="https://example.com/guide/seo/" />
<meta name="robots" content="index,follow,max-image-preview:large" />
<meta property="og:type" content="article" />
<meta property="og:title" content="标题" />
<meta property="og:description" content="摘要。" />
<meta property="og:url" content="https://example.com/guide/seo/" />
<meta property="og:image" content="https://example.com/og/seo.png" />
<meta name="twitter:card" content="summary_large_image" />
```

**B. robots 响应头（Nginx 示例）**
```
add_header X-Robots-Tag "noindex" always;
```

**C. News/Blog 列表分页链接（可见、可点击）**
```html
<nav aria-label="分页">
  <a href="/blog/page/1" rel="prev">上一页</a>
  <a href="/blog/page/3" rel="next">下一页</a>
</nav>
```

---

## 15) 小结

- 让**可抓取**（robots/sitemap/URL）与**可理解**（语义/结构化数据）并进，再用**性能**与**社交卡片**提升展示效果。  
- 以“模板 + 清单”持续巡检，比零散技巧更稳。把本章模板集成到你的框架布局与 CI 检查里，即可获得长期红利。
