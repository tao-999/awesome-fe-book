# 1.1 HTML 语义化与可维护结构

本章目标：用正确的 HTML 元素表达真实含义，建立清晰稳定的文档结构，使样式与脚本更易维护，并天然具备更好的可访问性（a11y）与 SEO 友好度。

---

## 1. 基本原则

1. **语义优先**：先选对元素，再谈样式与交互；不要用外观驱动标签选择。  
2. **结构先行**：先画出“页眉/主内容/侧栏/页脚”的骨架，再填模块。  
3. **原生优先**：优先使用原生可交互元素（`a`、`button`、`input`、`details/summary`）。  
4. **最小 ARIA**：仅在原生语义不足时添加 ARIA，避免与原生语义冲突。  
5. **可维护选择器**：语义化 class 与 BEM/Utility 命名，避免“div 汤”。

---

## 2. 页面骨架与地标（Landmarks）

以下骨架覆盖语言、编码、视口、标题、描述、规范链接、主题色与地标区域，同时提供“跳转到主内容”。

```html
<!doctype html>
<html lang="zh-Hans">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>页面标题 | 站点名</title>
  <meta name="description" content="一句话描述当前页面内容（80–160 字符）。" />
  <link rel="canonical" href="https://example.com/path" />
  <meta name="theme-color" content="#111827" />
</head>
<body>
  <a class="skip-link" href="#main">跳到主内容</a>

  <header>
    <h1 class="site-title"><a href="/">站点名</a></h1>
    <nav aria-label="主导航">
      <ul>
        <li><a href="/posts">文章</a></li>
        <li><a href="/about">关于</a></li>
      </ul>
    </nav>
  </header>

  <main id="main">
    <!-- 主内容 -->
  </main>

  <aside aria-label="侧栏"><!-- 相关链接 / 广告 --></aside>
  <footer><small>© 2025 站点名</small></footer>
</body>
</html>
```

配套仅对阅读器可见的工具类与“跳转链接”样式：

```css
.visually-hidden {
  position: absolute !important;
  width: 1px; height: 1px; padding: 0; margin: -1px;
  overflow: hidden; clip: rect(0 0 0 0); white-space: nowrap; border: 0;
}
.skip-link { position: absolute; left: -9999px; top: 0; }
.skip-link:focus { left: 0; background: #111; color: #fff; padding: .5rem 1rem; }
```

---

## 3. 标题层级与分节元素

- 页面仅一个 `h1`（通常为页面/文章标题），随后按层级使用 `h2`–`h6`，不跳级。  
- `section` 表示主题分组，**应包含可见标题**；`article` 表示可独立分发的自包含内容（文章、卡片、评论）。  
- `aside` 用于旁支信息；`header`、`footer` 可用于页面级或分节级。

示例：文章页面结构（含作者信息与时间）。

```html
<main id="main">
  <article>
    <header>
      <h1>高可维护前端架构的 7 条路径</h1>
      <p>
        <span class="author">作者：张三</span> ·
        <time datetime="2025-09-15">2025-09-15</time>
      </p>
    </header>

    <section>
      <h2>为什么要关注可维护性</h2>
      <p>……</p>
    </section>

    <section>
      <h2>实践清单</h2>
      <h3>语义与结构</h3>
      <ul>
        <li>明确的 landmarks：header/main/aside/footer</li>
        <li>一页仅一个 h1</li>
      </ul>
    </section>

    <footer>
      <p>分类：架构 · 标签：前端、可维护性</p>
    </footer>
  </article>
</main>
```

---

## 4. 列表、段落、引用与代码

- 有序内容用 `ol`，无序内容用 `ul`；术语/定义用 `dl/dt/dd`。  
- 引用其他来源用 `blockquote`，并标注来源。  
- 行内代码用 `code`，多行用 `pre > code`。  
- 图片与说明用 `figure/figcaption`。

```html
<section>
  <h2>术语表（示例）</h2>
  <dl>
    <dt>Landmark</dt>
    <dd>帮助辅助技术定位页面主要区域的元素。</dd>
  </dl>

  <figure>
    <img src="/assets/landmarks.png" alt="页面主要地标区域示意图" />
    <figcaption>地标区域示意图</figcaption>
  </figure>

  <pre><code>npm run build</code></pre>
</section>
```

---

## 5. 链接与导航

- 跳转页面用 `a`；触发动作用 `button`。  
- 外链应带 `rel="noopener noreferrer"`；指向当前页的链接应禁用或高亮。  
- 面包屑与分页要使用 `nav` 并提供 `aria-label`。

```html
<nav aria-label="面包屑">
  <ol>
    <li><a href="/">首页</a></li>
    <li><a href="/posts">文章</a></li>
    <li aria-current="page">可维护结构</li>
  </ol>
</nav>

<nav aria-label="分页">
  <a href="?page=1" rel="prev">上一页</a>
  <a href="?page=3" rel="next">下一页</a>
</nav>
```

---

## 6. 表单语义与可访问性

- `label for` 绑定控件，或将控件嵌入 `label`。  
- 逻辑分组用 `fieldset/legend`；辅助说明用 `aria-describedby`。  
- 合理选择 `input type`（`email`、`tel`、`url`、`date`……）。  
- 校验错误状态应标注并关联到输入控件。

```html
<form action="/signup" method="post" novalidate>
  <fieldset>
    <legend>注册</legend>

    <p>
      <label for="email">邮箱</label>
      <input id="email" name="email" type="email" required
             aria-describedby="email-hint email-error" />
      <small id="email-hint">我们不会公开你的邮箱。</small>
      <span id="email-error" class="visually-hidden" aria-live="polite"></span>
    </p>

    <p>
      <label for="pwd">密码</label>
      <input id="pwd" name="pwd" type="password" minlength="8" required />
    </p>

    <button type="submit">创建账户</button>
  </fieldset>
</form>
```

---

## 7. 图像与多媒体

- 内容图像提供准确 `alt`；装饰性图像使用空 `alt=""` 或 CSS 背景。  
- 响应式图像用 `picture/srcset/sizes`；视频提供字幕 `track kind="captions"`。

```html
<picture>
  <source srcset="/img/hero.avif" type="image/avif" />
  <source srcset="/img/hero.webp" type="image/webp" />
  <img src="/img/hero.jpg" alt="团队在白板前讨论系统架构" width="1200" height="600" />
</picture>

<video controls width="640">
  <source src="/media/intro.webm" type="video/webm" />
  <source src="/media/intro.mp4" type="video/mp4" />
  <track kind="captions" src="/media/intro.zh.vtt" srclang="zh" label="中文字幕" default />
  抱歉，浏览器不支持视频播放。
</video>
```

---

## 8. 表格语义

- 使用 `caption` 描述表格目的；表头用 `thead th`，数据区用 `tbody td`。  
- `th` 提供 `scope="col|row"` 或 `headers`/`id` 显式关联。  
- 列分组用 `colgroup/col` 提示样式或含义。

```html
<table>
  <caption>季度性能指标</caption>
  <colgroup>
    <col span="1" />
    <col class="kpi" />
    <col class="kpi" />
  </colgroup>
  <thead>
    <tr>
      <th scope="col">指标</th>
      <th scope="col">Q2</th>
      <th scope="col">Q3</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">LCP</th>
      <td>2.1s</td>
      <td>1.9s</td>
    </tr>
    <tr>
      <th scope="row">INP</th>
      <td>160ms</td>
      <td>120ms</td>
    </tr>
  </tbody>
</table>
```

---

## 9. 组件化与命名策略

- **BEM**：`block__element--modifier`，选择器稳定、可组合。  
- **数据属性**：用 `data-*` 存储轻量语义或状态（便于脚本选择）。  
- **避免过度嵌套**：浅层次结构 + 语义化 class。

```html
<article class="card card--featured" data-id="42">
  <header class="card__header">
    <h3 class="card__title"><a href="/post/42">语义化的价值</a></h3>
  </header>
  <p class="card__excerpt">……</p>
  <footer class="card__footer">
    <button class="btn btn--primary">阅读全文</button>
  </footer>
</article>
```

---

## 10. 最小 ARIA 模式（仅在必要时）

优先使用原生元素。若确需自定义折叠/展开，可使用 `button + aria-expanded + aria-controls`：

```html
<button type="button" aria-expanded="false" aria-controls="sect-1" id="disclosure">
  展开详情
</button>
<section id="sect-1" hidden>
  <h2>详细说明</h2>
  <p>……</p>
</section>

<script>
  const btn = document.getElementById('disclosure');
  const sect = document.getElementById('sect-1');
  btn.addEventListener('click', () => {
    const expanded = btn.getAttribute('aria-expanded') === 'true';
    btn.setAttribute('aria-expanded', String(!expanded));
    sect.hidden = expanded;
  });
</script>
```

如果需求允许，更推荐原生的 `details/summary`：

```html
<details>
  <summary>展开详情</summary>
  <p>……</p>
</details>
```

---

## 11. SEO 与元数据（基础）

- 每页提供唯一的 `title` 与贴合内容的 `meta description`。  
- 使用 `link rel="canonical"` 避免重复内容。  
- 适当加入结构化数据（JSON-LD）。

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BlogPosting",
  "headline": "高可维护前端架构的 7 条路径",
  "datePublished": "2025-09-15",
  "author": { "@type": "Person", "name": "张三" }
}
</script>
```

---

## 12. 可维护结构检查清单

- [ ] 页面包含明确的 landmarks（`header/main/aside/footer`）。  
- [ ] 仅一个 `h1`，标题层级连续且不跳级。  
- [ ] 导航使用 `nav` 并提供 `aria-label`；当前项明确高亮或禁用。  
- [ ] 表单 `label` 完整绑定；错误信息与控件相关联。  
- [ ] 内容图像有正确 `alt`；装饰图像使用空 `alt`。  
- [ ] 表格包含 `caption` 与正确的表头关联。  
- [ ] 交互优先原生元素；自定义部件具备键盘与可读状态。  
- [ ] 结构与样式分离；class 命名一致、可读、可检索。  
- [ ] 元数据完善：`title`、`description`、`canonical`、结构化数据（视内容类型）。

---

## 13. 常见反模式与重构建议

- **div/span 滥用**：用 `div` 模拟 `nav/table/form`。  
  **重构**：换成语义元素，并添加必要标题与 `aria-label`。  
- **链接当按钮** / **按钮当链接**：  
  **重构**：跳转用 `a`，动作用 `button`，并提供 `type="button|submit"`。  
- **标题层级混乱**：多个 `h1` 或跳级。  
  **重构**：统一从 `h2` 起分节，按层级递进。  
- **自定义控件不可达**：没有键盘操作/焦点样式。  
  **重构**：补齐键盘交互、`aria-*` 与可见焦点，或改用原生元素。  
- **表单校验只有颜色提示**：对色弱不友好。  
  **重构**：文本错误 + `aria-describedby` 关联，并用 `aria-live` 宣告。

---

## 14. 验收标准与练习

**验收**：  
- 页面通过标题层级与 landmarks 检查；  
- 表单具备可读错误提示与键盘可达；  
- 图像 `alt` 完整；  
- Lighthouse a11y ≥ 95（或等价工具指标）。

**练习**：  
1) 选一页“div 汤”页面，重写为语义化结构，记录改动前后 DOM 树差异。  
2) 为站点加上 `skip-link` 与 `visually-hidden`，检查键盘导航是否顺畅。  
3) 将一个自定义折叠面板替换为 `details/summary`，或按“最小 ARIA 模式”补全可达性。

---

## 15. 参考模板：最小页面示例

```html
<!doctype html>
<html lang="zh-Hans">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>示例页 | 站点名</title>
  <meta name="description" content="示例页的简短描述。" />
  <link rel="canonical" href="https://example.com/demo" />
</head>
<body>
  <a class="skip-link" href="#main">跳到主内容</a>
  <header>
    <h1 class="site-title"><a href="/">站点名</a></h1>
    <nav aria-label="主导航">
      <a href="/posts" aria-current="page">文章</a>
      <a href="/about">关于</a>
    </nav>
  </header>
  <main id="main">
    <article>
      <header>
        <h2>示例文章标题</h2>
        <p><time datetime="2025-09-15">2025-09-15</time></p>
      </header>
      <p>这里是正文段落。</p>
    </article>
  </main>
  <aside aria-label="侧栏">侧栏内容</aside>
  <footer><small>© 2025 站点名</small></footer>
</body>
</html>
```
