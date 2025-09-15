# 18.1 XSS / CSRF / CSP / SRI 🛡️

目标：把**前端安全四天王**讲清楚并给出能直接抄的配置与代码。口号：**少拼运气，多用机制**。

---

## 0) TL;DR（给忙人）🎯
- **XSS**：所有可进 DOM 的外来内容都要**按上下文转义**或**消毒**；危险 API 封印；能用框架绑定就别自己拼字符串。
- **CSRF**：状态修改接口必须验证**SameSite Cookie + CSRF Token + Origin/Referer** 三件套之一或组合。
- **CSP**：默认全部禁，**白/信任名单 + nonce/hash + strict-dynamic** 放行；禁止 `unsafe-inline`。
- **SRI**：外链脚本/样式加 **integrity**，并搭配 `crossorigin`；从源头控制被篡改风险。

---

## 1) XSS（跨站脚本）💥

### 1.1 形态速记
- **反射型**：URL 参数回显到页面。
- **存储型**：恶意内容落库（评论/昵称），后续被所有人加载。
- **DOM 型**：前端代码把输入带入危险 **sink**（`innerHTML/outerHTML/insertAdjacentHTML`、`document.write`、`eval`、`new Function`、`setTimeout('string')`、`location = ...` 等）。

### 1.2 上下文转义（最稳的通用做法）
- **HTML 文本节点**：`<div>{text}</div>` → 逃逸 `& < > " ' /`。
- **HTML 属性**：`<a title="{text}">` → 同样需要转义；URL 属性（`href/src`）需**白名单协议**（`http/https/mailto/tel`）。
- **URL 上下文**：`<a href="/search?q={encodeURIComponent(q)}">`。
- **JS 字面量上下文**：`<script>var x = JSON.stringify(data)</script>`；或完全避免把数据塞进 `<script>`，改为 `data-*` + JS 读取。
- **CSS**：避免把用户输入写到 style；必要时只允许 **合法类名/预定义枚举**。

### 1.3 框架建议
- **React/Vue/Svelte**：默认绑定是安全的；**禁止** `dangerouslySetInnerHTML` / `v-html` / `{@html}` 直接喂用户输入。实在要用，配 **DOMPurify** 严格白名单。
- **模板引擎（EJS/Handlebars/Nunjucks）**：使用**转义标签**而非原样输出标签（例如 `{{name}}` 而不是 `{{{name}}}`）。

### 1.4 DOMPurify 最小示例（只允许极少标签）
~~~ts
import DOMPurify from 'dompurify';
const clean = DOMPurify.sanitize(userHtml, {
  ALLOWED_TAGS: ['b','i','em','strong','a','p','ul','ol','li','code','pre'],
  ALLOWED_ATTR: ['href','title','target','rel'],
  FORBID_TAGS: ['style','iframe','script'],
  ALLOW_UNKNOWN_PROTOCOLS: false
});
container.innerHTML = clean;
~~~

### 1.5 可信类型（Trusted Types，Chrome）
- CSP 增加：`require-trusted-types-for 'script'; trusted-types app;`
- 代码里注册策略，**禁止**直接把字符串塞进脚本/HTML sink。
~~~ts
// 注册策略，只允许我们用 DOMPurify 产物
// @ts-ignore
window.trustedTypes?.createPolicy('app', {
  createHTML: (s) => DOMPurify.sanitize(s, {RETURN_TRUSTED_TYPE: true})
});
~~~

---

## 2) CSRF（跨站请求伪造）🎯

### 2.1 何时会中招
- 你用 **Cookie** 作为会话（HttpOnly/同域），而接口又允许被第三方页面自动发起（表单/图片/脚本）→ 攻击者借用户浏览器的**隐式身份**完成操作。

### 2.2 防护三件套（至少选一，推荐组合）
1) **SameSite Cookie**  
   - `Set-Cookie: session=...; HttpOnly; Secure; SameSite=Lax`（默认 GET 导航不带；表单 POST 导航在部分浏览器带或不带，建议仍做 Token）。  
   - 需要跨站嵌入/跳回登录的场景，改 `SameSite=None; Secure`，**更要有 CSRF Token**。
2) **CSRF Token**（表单或自定义请求头携带）  
   - 服务端签发 **双提交** Token（Cookie + 表单字段一致）或**服务端存储** Token。  
   - 非简单请求用 `X-CSRF-Token` 头，跨域时需通过 CORS 明确允许。
3) **Origin/Referer 校验**  
   - 检查写接口的 `Origin/Referer` 是否在白名单来源（注意隐私插件可能去 Referer，Origin 更稳）。

> 另外：**所有状态修改接口**只接受 `POST/PUT/PATCH/DELETE`，拒绝 GET。

### 2.3 Express 伪代码
~~~ts
// 生成 Token（服务端保存到服务端会话 / 或加密签名放 Cookie）
app.get('/form', (req, res) => {
  const token = issueCsrfToken(req);
  res.cookie('csrf', token, { httpOnly: false, sameSite: 'Lax' });
  res.render('form', { token });
});

function verifyCsrf(req, res, next) {
  const tokenFromCookie = req.cookies['csrf'];
  const tokenFromBody   = req.body['_csrf'] || req.get('x-csrf-token');
  const origin = req.get('origin');
  if (!isTrustedOrigin(origin)) return res.sendStatus(403);
  if (!safeEqual(tokenFromCookie, tokenFromBody)) return res.sendStatus(403);
  next();
}
app.post('/api/transfer', verifyCsrf, handler);
~~~

---

## 3) CSP（内容安全策略）🧱

### 3.1 理念
**默认全部拒绝**，只对明确的源放行；内联脚本必须使用 **nonce/hash**，再配合 `strict-dynamic` 让受信任脚本动态加载的子脚本继承信任。

### 3.2 推荐基线（HTTP 响应头）
~~~text
Content-Security-Policy:
  default-src 'none';
  script-src 'self' 'nonce-r4nd0m' 'strict-dynamic' https:;
  style-src  'self' 'nonce-r4nd0m' https: 'unsafe-inline';
  img-src    'self' data: https:;
  font-src   'self' https: data:;
  connect-src 'self' https:;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
  object-src 'none';
  require-trusted-types-for 'script';
  upgrade-insecure-requests;
~~~
- **`nonce-...`** 每次请求随机生成，内联 `<script nonce="...">` 才会执行。  
- **`strict-dynamic`**：受信任脚本动态插入的脚本也被信任（减少维护长白名单）。  
- 样式若无法去内联，可暂时 `unsafe-inline`，更优解是 CSS 文件化或使用 `nonce`。

### 3.3 页面内配合
~~~html
<script nonce="r4nd0m">window.__BOOTSTRAP__ = {/*...*/}</script>
<script nonce="r4nd0m" src="/assets/app.js" defer></script>
~~~

### 3.4 报警与灰度
- 增加 `Content-Security-Policy-Report-Only` 观察违规再切换强制。
- `report-to`/`report-uri` 收集违规事件，做告警看板。

---

## 4) SRI（子资源完整性）🔐
为外链脚本/样式增加哈希，防 CDN/传输篡改。

~~~html
<link rel="stylesheet"
      href="https://cdn.example.com/x.css"
      integrity="sha384-abc..."
      crossorigin="anonymous">
<script src="https://cdn.example.com/x.js"
        integrity="sha384-def..."
        crossorigin="anonymous" defer></script>
~~~
- 推荐 **SHA-384/512**；内容更新需同步更新 hash。  
- 与 CSP 搭配时，`script-src` 允许对应源或 `https:`；SRI 解决**完整性**，CSP 控制**来源**。

---

## 5) 额外安全响应头（顺手就上）
~~~text
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=(), interest-cohort=()
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Resource-Policy: same-site
Cross-Origin-Embedder-Policy: require-corp
~~~

---

## 6) Token 存哪？（常见误区）🧨
- **敏感会话**：优先 **HttpOnly Cookie**（配 SameSite & CSRF 方案）。  
- **不要把长期 Token 放 localStorage**（易被 XSS 盗取）；若必须放，**短期 + 绑定客户端 + 轮换**。  
- 前端存储只放**低敏**数据；高敏操作通过服务端会话校验。

---

## 7) 端到端示例：Nginx / Node / 前端

### 7.1 Nginx 头部模板
~~~nginx
add_header Content-Security-Policy "default-src 'none'; script-src 'self' 'nonce-$request_id' 'strict-dynamic' https:; style-src 'self' 'nonce-$request_id' https: 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' https: data:; connect-src 'self' https:; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'; require-trusted-types-for 'script'; upgrade-insecure-requests" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
~~~

### 7.2 Express 设置 Cookie & Samesite
~~~ts
res.cookie('session', token, {
  httpOnly: true,
  secure: true,
  sameSite: 'Lax', // 或 None + CSRF Token
  path: '/',
  maxAge: 7*24*3600*1000
});
~~~

### 7.3 前端表单（带 CSRF）
~~~html
<form method="POST" action="/api/transfer">
  <input type="hidden" name="_csrf" value="{{csrfToken}}">
  <!-- 其他字段 -->
</form>
~~~

---

## 8) 反模式与纠偏 🧨
| 反模式 | 危害 | 纠偏 |
|---|---|---|
| 把用户输入塞进 `innerHTML` | DOM XSS | 用文本节点绑定；要 HTML 就 DOMPurify + Trusted Types |
| 在 `<script>` 里直接拼 JSON | XSS | 用 `JSON.stringify` 或 `data-*` + JS 读取 |
| 仅靠 SameSite 防 CSRF | 兼容差/被绕过 | 叠加 **CSRF Token** 与 **Origin 校验** |
| `unsafe-inline` 一开到底 | CSP 形同虚设 | 用 **nonce/hash**；逐步清理内联脚本 |
| 外链脚本不加 SRI | 篡改风险 | `integrity` + `crossorigin="anonymous"` |
| 把长期 Token 放 localStorage | 被 XSS 窃取 | 用 **HttpOnly Cookie** 或短期可轮换 Token |
| GET 修改状态 | 易被 CSRF/缓存污染 | 改用 POST/PUT/PATCH/DELETE，禁缓存 |

---

## 9) 验收清单 ✅
- [ ] 所有用户输入按**上下文转义**；富文本经过 **DOMPurify**；危险 sink 受控。  
- [ ] 写接口具备 **SameSite/CSRF/Origin** 之一（推荐组合），且仅接受非 GET。  
- [ ] 已启用 **CSP（Report-Only → Enforce）**，使用 **nonce/hash + strict-dynamic**；违规有告警。  
- [ ] 外链脚本/样式开启 **SRI**；内部资源走自家域名/CDN。  
- [ ] 关键响应头（`nosniff/Referrer-Policy/Permissions-Policy` 等）就位。  
- [ ] E2E 覆盖：XSS 注入用例、跨域页面表单攻击、CSP 阻断回归。  

---

## 10) 练习 🏋️
1. 把项目切换到 **CSP Report-Only**，收 48 小时违规报表，逐项消除后切到 Enforce。  
2. 将所有外链依赖补全 **SRI**，写一个 CI 脚本在构建后校验 hash。  
3. 为“评论/富文本”接入 **DOMPurify + Trusted Types**，并封装安全渲染组件。  
4. 写一条 **CSRF 攻击用例**（第三方域表单 POST），验证你的防线生效。  

---

**小结**：XSS 靠**输入处理 + API 约束**，CSRF 靠**会话策略 + Token/Origin**，CSP/SRI 是**兜底护栏**。把机制开到位，风险自然降到“脚本小子都嫌无聊”的水平。🔒
