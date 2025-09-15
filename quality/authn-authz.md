# 18.3 鉴权：OAuth2 / OIDC / JWT / Passkeys（WebAuthn） 🔐🪪

目标：给 **Web / 移动 / 桌面** 搭一套“安全、可观测、可演进”的登录与授权体系。口号：**前端不摸长 token，后端守口如瓶，协议各司其职**。

---

## 0) 导航图（TL;DR）🎯
- **浏览器**：用 **OAuth2 授权码 + PKCE**（别再用隐式流）。
- **OIDC**：在 OAuth2 之上加“身份层”（`id_token`），只给客户端看，不拿它打 API。
- **JWT**：短效访问令牌（5–15 分钟），**签名校验 + 密钥轮换 + 受众/颁发方校验**。
- **Passkeys / WebAuthn**：无密码/二要素主力，配合“无用户名登录”。
- **架构推荐**：SPA 采用 **BFF（Backend for Frontend）**；**Refresh Token 仅服务器持有**；浏览器只拿 **HttpOnly SameSite Cookie** 或**内存短效 AT**。

---

## 1) 角色速记 🧠
- **Resource Owner**：用户  
- **Client**：你的 App（SPA/SSR/Native）  
- **Authorization Server / IdP**：身份提供方（Auth0、Keycloak、Cognito、Ory…）  
- **Resource Server**：业务 API  
- **Access Token（AT）**：打 API 的票（短效）  
- **Refresh Token（RT）**：换 AT 的票（尽量不在浏览器落地）  
- **ID Token（OIDC）**：用户是谁（给客户端看的身份声明）

---

## 2) OAuth2 授权码 + PKCE（浏览器首选）🪜

### 2.1 为什么是它
- **PKCE**：`code_verifier` / `code_challenge` 抗拦截 + 抗授权码注入。  
- 天然兼容 **OIDC**（附带 `id_token`）。  
- 支持 **Refresh Token 轮换**、**DPoP**（持有证明）等现代增强。

### 2.2 流程（简化）
1. 客户端生成 `state`（抗 CSRF）与 `code_verifier`，派生 `code_challenge = S256(code_verifier)`。  
2. 跳转 IdP `/authorize?...&response_type=code&code_challenge=...&state=...`。  
3. 登录→重定向回 `redirect_uri?code=...&state=...`。  
4. 客户端（或 BFF）以 `code + code_verifier` 向 `/token` 换 **AT（+RT）**。  
5. 用 AT 调 API；到期用 RT 换新（**轮换 + 重用检测**）。

### 2.3 SPA 三种姿势（由稳到激进）
- **BFF（推荐）**：浏览器 ↔ BFF 用 **HttpOnly SameSite Cookie** 维持会话；**RT 放 BFF**；BFF 向 API 附带短效 AT。  
- **纯 SPA**：Code+PKCE，**AT 只放内存**；**尽量别给 RT**（或极短效 + 轮换 + DPoP）。  
- **SSR（Next/Nuxt）**：服务端完成换码与刷新，前端只接 Cookie。

---

## 3) OIDC：把“你是谁”标准化 📛
- 授权时加 `scope=openid profile email`。  
- **ID Token**（JWT）字段：`iss`、`aud`、`sub`、`exp/iat/nbf`、`nonce`。  
- **只用于 UI**（头像/昵称/UID），**别拿 ID Token 调 API**。  
- 回调时校验 `state` 与 `nonce`，拒绝重放。

---

## 4) JWT 访问令牌：校验与轮换 🔏

### 4.1 校验要点
- 使用 IdP 的 **JWKS（/.well-known/jwks.json）**，按 `kid` 取公钥，**缓存 + 轮换**。  
- 校验：`iss`（颁发方）、`aud`（受众）、`exp/nbf`、签名算法（RS256/ES256/EdDSA）。  
- **短效 AT** 5–15 分钟；**RT 轮换** + **重用检测**（被盗用时吊销会话）。

### 4.2 发送方绑定（可选）
- **DPoP**：AT 与客户端私钥绑定，抗窃取重放。  
- **mTLS**：服务↔服务的强绑定方案。

### 4.3 撤销与内省
- `/oauth/revoke` 撤销 RT/AT；  
- `/oauth/introspect` 适合**不透明令牌**或需要实时吊销判断的 API。

---

## 5) 授权环节安全三锁 🔐
- **`state`**：请求与回调比对（放 Cookie/SessionStorage）。  
- **`nonce`**：OIDC ID Token 的防重放参数。  
- **精确 `redirect_uri` 白名单**：拒绝通配；登出回跳同理。  
- **PKCE**：浏览器客户端必配。

---

## 6) Passkeys / WebAuthn：无密码登录主力 🗝️

### 6.1 能力
- 公私钥替代密码，不可被钓鱼；可作为 **2FA** 或 **纯无密码**。  
- **平台凭据**（设备内置，可云同步）与 **跨平台 FIDO Key**。

### 6.2 注册流程（`navigator.credentials.create`）
1. 服务器生成 `challenge`，下发 `rp.id / user.id / alg 列表`。  
2. 前端 `create()` → 系统生物识别/Pin → 返回 `attestationObject + clientDataJSON`。  
3. 服务端验证：`challenge`、`origin`、`rpIdHash`、签名算法、唯一 `credentialId`；存 `publicKey/signCount`。  
4. 通常 `attestation: "none"`（除非合规要求）。

### 6.3 登录流程（`navigator.credentials.get`）
1. 服务器生成新 `challenge`，选择候选凭据（支持**无用户名** discoverable）。  
2. 前端 `get()` → 返回 `authenticatorData + clientDataJSON + signature (+ userHandle)`。  
3. 服务端用已存公钥验签；校 `origin/rpId/challenge`；处理 `signCount`（iOS/云同步可能 0）。

### 6.4 参数建议
- `userVerification: "required"`；`residentKey: "preferred/required"` 以启用“无用户名登录”。  
- 过渡路径：先做 **密码+Passkey**（2FA）→ 成熟后转 **纯 Passkey**；保留 **恢复通道**（Magic Link / TOTP 备份）。

---

## 7) 架构蓝图（可落地）🏗️

### 7.1 SPA + BFF（最佳实践）
- 浏览器与 BFF：**HttpOnly SameSite Cookie** 会话票据（短效，可滚动刷新）。  
- BFF 与 IdP：处理 Code+PKCE / 刷新，**RT 存服务器**（加密 at rest）。  
- BFF 与 API：按需获取/缓存**短效 AT**；401 透明刷新；失败降级到重新授权。  
- 浏览器**不持久化 Token**（最多内存态标识），XSS 面暴降。

### 7.2 SSR（Next/Nuxt）
- 中间件完成授权交换；受保护路由检查会话；下游 API 由服务器持 Token。  
- 跨子域时同站策略（`SameSite=Lax` + CSRF Token）或 BFF 代理。

### 7.3 Native / 桌面
- 使用系统浏览器（ASWebAuthenticationSession / Custom Tabs）+ URL Scheme 回调；**Code+PKCE**；不配置 client secret。

---

## 8) 代码片段（可抄可改）🧩

### 8.1 校验 JWT（Node / `jose`）
```ts
import * as jose from 'jose';
const JWKS = jose.createRemoteJWKSet(new URL('https://idp.example.com/.well-known/jwks.json'));

export async function verifyAccessToken(token: string) {
  const { payload } = await jose.jwtVerify(token, JWKS, {
    issuer: 'https://idp.example.com/',
    audience: 'api://your-audience'
  });
  // 业务内授权（scope/role/tenant）
  if (!payload.scope?.includes('read:articles')) throw new Error('insufficient_scope');
  return payload;
}
```

### 8.2 生成 `state/nonce` & 发起授权（浏览器）
```ts
const state = crypto.randomUUID().replace(/-/g,'');
const nonce = crypto.randomUUID().replace(/-/g,'');
sessionStorage.setItem('oidc.state', state);
sessionStorage.setItem('oidc.nonce', nonce);

// 生成 PKCE
function b64url(buf: ArrayBuffer) {
  return btoa(String.fromCharCode(...new Uint8Array(buf))).replace(/=/g,'').replace(/\+/g,'-').replace(/\//g,'_');
}
const verifierBytes = crypto.getRandomValues(new Uint8Array(32));
const verifier = b64url(verifierBytes.buffer);
const challenge = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(verifier)).then(b64url);

location.assign(
  `https://idp.example.com/authorize?response_type=code&client_id=...&redirect_uri=${encodeURIComponent(location.origin+'/callback')}` +
  `&scope=openid%20profile%20email&state=${state}&code_challenge=${challenge}&code_challenge_method=S256&nonce=${nonce}`
);
```

### 8.3 WebAuthn 注册（前端调用参数）
```ts
const cred = await navigator.credentials.create({
  publicKey: {
    rp: { id: 'example.com', name: 'Awesome App' },
    user: { id: new Uint8Array(userIdBytes), name: email, displayName: nickname },
    challenge: new Uint8Array(challengeBytes),
    pubKeyCredParams: [{ alg: -7, type: 'public-key' }, { alg: -257, type: 'public-key' }], // ES256/RS256
    authenticatorSelection: { residentKey: 'preferred', userVerification: 'required' },
    attestation: 'none',
    timeout: 60000
  }
});
```

---

## 9) 常见坑与纠偏 🧨
| 反模式 | 后果 | 纠正 |
|---|---|---|
| 浏览器把 AT/RT 放 `localStorage` | XSS 一穿就透 | **BFF 会话 Cookie** 或 AT 仅内存 |
| 用 `id_token` 调 API | 身份/授权混乱 | API 只认 **Access Token** |
| 不校 `state/nonce/iss/aud` | 重放/冒用 | 全量校验 + 精确 `redirect_uri` 白名单 |
| AT 设置 1–12 小时 | 被盗长期可用 | **AT 5–15 分钟** + **RT 轮换** |
| 多应用同域共享 Cookie | 会话污染 | 独立子域/Path + 独立 Cookie 前缀 |
| Passkey 不留恢复通道 | 用户换机锁死 | 启用多设备同步 + 备用邮箱/TOTP |
| 不缓存 JWKS / 不看 `kid` | 密钥轮换时全挂 | **按 `kid` 缓存**，失败再回源刷新 |
| 仍用隐式流 | 令牌暴露在 URL | 迁移到 **授权码 + PKCE** |

---

## 10) 提交前检查清单 ✅
- [ ] 采用 **授权码 + PKCE**（隐式流已禁）。  
- [ ] OIDC：`state/nonce` 生成与回调校验齐全；`id_token` 只用于 UI 身份。  
- [ ] AT 短效（≤15m）；**RT 仅服务器持有**并启用**轮换 + 重用检测**。  
- [ ] API 网关/服务完成 **JWT 校验（iss/aud/exp/签名）**，**JWKS 缓存 + 轮换**。  
- [ ] 浏览器仅用 **HttpOnly SameSite Cookie** 维持会话（或内存 AT）；不落地到 `localStorage`。  
- [ ] Passkeys 已支持：注册/登录严格校验，开启**无用户名登录**与**恢复通道**。  
- [ ] 登出：后端清会话 +（可选）OIDC RP-Initiated Logout。  
- [ ] 观测：登录成功率、错误分类、刷新失败率、JWKS 拉取失败告警。

---

## 11) 练习 🏋️
1. 本地跑一个 **Keycloak**，打通 **Code+PKCE+OIDC**；BFF 存会话，前端仅 `id_token` 用于展示头像/昵称。  
2. 在 API 网关加 **JWT 验证**与 **`scope`/`role` 授权**；把 AT 过期从 60m 改到 10m，观察失败率与体验。  
3. 在“账号安全”页加 **启用 Passkey**：首次作为 2FA，第二次试无用户名登录。  
4. 实装 **RT 轮换**：模拟重用旧 RT，确认被标记并吊销全会话。  

---

**小结**：以 **OAuth2（PKCE）** 为地基，**OIDC** 提供身份，**JWT** 负责通行证，**Passkeys** 负责无密码与强验证；再辅以 **BFF 会话** 与**完善校验/轮换/观测**，你的登录与授权就能做到**更顺滑、更难骗、更易管**。🚀
