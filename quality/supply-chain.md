# 18.2 供应链安全 / 依赖审计 🔒📦

目标：把 **NPM/PNPM/Yarn** 的供应链风险控住，从“装包”到“上线”全链路可追可控。口号：**锁、审、签、隔、最小权限**。

---

## 0) TL;DR（放在 README 顶部的硬规矩）🎯
- **锁定 + 冻结**：提交 lockfile；CI 用 `npm ci` / `pnpm i --frozen-lockfile` / `yarn --immutable`。
- **最小化安装面**：CI 默认 `ignore-scripts`；仅对白名单包开放 `postinstall`。
- **私有代理仓**：所有安装都走 **企业私有仓/代理**（Jfrog/Nexus/Verdaccio）；外源包先“隔离 → 审计 → 准入”。
- **自动审计与修复**：`npm|pnpm audit` + **OSV/Snyk** 扫描；Renovate/Dependabot 自动发 PR。
- **签名与溯源**：启用 **npm provenance**；制品/镜像用 **Sigstore/cosign**；生成 **SBOM（CycloneDX/SPDX）**。
- **防依赖混淆**：内部 `@scope` 强制映射到私有 registry；禁止 `github:`/`http:` 安装源。
- **账号安全**：npm 组织强制 **2FA**；CI 使用 **最小权限 Token**；仓库启用 **分支保护 + 签名提交**。

---

## 1) 风险图谱（知道敌人是谁）🧠
- **恶意/投毒包**：typosquatting、`event-stream`、Protestware。
- **依赖混淆（Dependency Confusion）**：内部包名被公网同名劫持。
- **安装脚本滥用**：`pre/postinstall` 执行任意代码，窃取环境变量/令牌。
- **越权升级**：lockfile 漂移，间接依赖被拉升到有洞版本。
- **不可追溯**：谁引入了什么？哪个版本？上线镜像里到底放了啥？

---

## 2) 锁定与安装硬化（第一道栅栏）🧱
- **提交并守护 lockfile**：`package-lock.json` / `pnpm-lock.yaml` / `yarn.lock`。
- **CI 冻结安装**
    - npm：`npm ci --ignore-scripts`
    - pnpm：`pnpm i --frozen-lockfile --ignore-scripts`
    - yarn(berry)：`yarn --immutable`（可配 `enableScripts: false`）
- **Corepack 固定工具链**：项目根写 `packageManager: "pnpm@9.x"`；CI 先 `corepack enable`。
- **脚本白名单**：仅允许确需的 `postinstall`（如 `esbuild`/`sharp` 下载二进制）。其余默认禁用。

GitHub Actions 片段：
    
    - run: corepack enable
    - run: pnpm i --frozen-lockfile --ignore-scripts
    - run: pnpm run build

---

## 3) 私有代理仓与命名防线（把水闸装好）🚧
- 强制 `@company` scope 走内网仓库：

    ; .npmrc / .pnpmrc / .yarnrc.yml（等价配置）
    @company:registry=https://npm.company.internal
    registry=https://npm.company.internal
    audit=true

- 代理仓策略：**白名单引入**、缓存镜像、准备“准入门槛”（无高危脚本/许可证合规/信誉分达标）。
- 禁止直接安装源：`github:` / `git+ssh:` / `http(s)://*.tar.gz`（必须先入私库再安装）。

---

## 4) 审计与自动升级（持续体检）🩺
- **本地/CI 快扫**：`npm audit --audit-level=moderate` / `pnpm audit`；再用 **OSV-Scanner** 交叉验证。
- **平台级 SCA**：Snyk/Dependabot/Renovate 持续监控 → 自动 PR（小步快跑，固定合并窗口）。
- **修复优先级**
    1) 直接升级（兼容范围内）
    2) `overrides`/`resolutions` 钉子依赖
    3) `patch-package` 临时补丁
    4) 必要时 **Fork**（紧急且上游未合）

`overrides` 示例（package.json）：
    
    {
      "overrides": {
        "minimist": "^1.2.8",
        "postcss>nanoid": "^3.3.7"
      }
    }

---

## 5) 安装脚本与环境隔离（高危点减法）🧪
- CI 默认 `ignore-scripts`；确需脚本的包使用**白名单 rebuild**：

    pnpm rebuild esbuild sharp

- 隔离敏感变量：安装/构建阶段不暴露云密钥；优先 **OIDC** / 短期最小权限 Token。
- Yarn Berry：`enableScripts: false`、`pnpMode: strict` 拦截幽灵依赖与脚本执行。

---

## 6) SBOM / 签名 / 溯源（证据链）🧾✍️
- 生成 **SBOM**
    - CycloneDX：`npx @cyclonedx/cyclonedx-npm --omit dev -o sbom.json`
    - Syft：`syft packages dir:. -o cyclonedx-json`
- **验证与门禁**
    - 用 **OSV/Snyk** 扫 SBOM；CI 不达标直接失败。
    - 制品签名：容器/构建产物用 **cosign sign-attest**；部署前 **cosign verify**。
    - npm 发布启用 **provenance**：`npm publish --provenance`（配合 GitHub Actions 生成 SLSA 证明）。
- 开源健康度：**OpenSSF Scorecard**、**socket.dev** 打分低的包需替换或加沙箱。

---

## 7) 依赖混淆与镜像污染（专项防御）🧯
- 内部包统一 `@company/*` 命名；**不要**将内部无 scope 包发到公网。
- 代理仓固定上游（只信任 npm 官方），校验 tarball `sha512`；关闭“任意 registry 跳转”。
- 生产镜像做二次校验：`npm ci --prefer-online --ignore-scripts` 并验证 **lockfile integrity**。

---

## 8) 发布与账号安全（人是最大变量）🔐
- npm 组织强制 **2FA（auth + publish）**；CI 用 **automation token**，权限最小化。
- 开启 **分支保护 / 必须签名提交（GPG/SSH）**；发布走 **Release PR + 审核**。
- 包 `publishConfig` 示例：

    {
      "access": "public",
      "provenance": true,
      "registry": "https://registry.npmjs.org/"
    }

- `engines` 与 `engine-strict=true` 统一 Node/包管器版本，避免“版本差异导致依赖树不同”。

---

## 9) 许可证合规（别踩雷）⚖️
- 生成 License 报表（`license-checker` / `pnpm licenses list` / `yarn licenses`）。
- 设 **拒绝名单**（如 AGPL-3.0）与 **替换策略**；无法替换时做 **例外评审**。
- 发布可分发制品时保留三方许可证归属文件（`THIRD-PARTY-NOTICES`）。

---

## 10) 反模式与纠偏 🧨
| 反模式 | 症状 | 纠偏 |
|---|---|---|
| CI 直接 `npm i` | 锁漂/不可复现 | `npm ci` / `--frozen-lockfile` |
| 任意 `postinstall` | 窃取环境变量 | CI `ignore-scripts` + 白名单 `rebuild` |
| 从 Git/URL 安装 | 不可追溯/篡改 | 先入私库、固定版本、审计后准入 |
| 不提交 lockfile | 随机依赖树 | 锁定 & 审查 PR 中的 lock diff |
| 只看 `npm audit` | 漏报/过报 | 叠加 **OSV/Snyk** 与 Scorecard |
| 生产镜像含 devDeps | 面积/风险扩大 | `NODE_ENV=production` 或 `pnpm prune --prod` |
| 账号无 2FA | 供应链单点失守 | 强制 2FA + automation token |
| 一键 `audit fix --force` | 破坏兼容 | 评估升级 + e2e 验证 + 渐进 rollout |

---

## 11) 验收清单 ✅
- [ ] CI 使用**冻结安装**，并默认 `ignore-scripts`。  
- [ ] 所有安装走**私有代理仓**；内部 `@scope` 已强制映射。  
- [ ] 自动化 **SCA**（OSV/Snyk）与 **Renovate/Dependabot** 开启；PR 包含 lockfile diff。  
- [ ] 关键制品有 **SBOM** 与 **签名/证明**；部署前验证通过。  
- [ ] 账号 **2FA**、发布 **provenance**、最小权限 Token 就位。  
- [ ] 许可证报表无红线；例外项留档与审计。  

---

## 12) 上手脚本与配置片段（可抄）🛠️
**package.json**

    {
      "packageManager": "pnpm@9.6.0",
      "overrides": { "minimist": "^1.2.8" },
      "engines": { "node": ">=18.18" }
    }

**.npmrc / .pnpmrc**

    @company:registry=https://npm.company.internal
    registry=https://npm.company.internal
    audit=true
    engine-strict=true
    fund=false

**Renovate（最小策略）**

    {
      "extends": ["config:recommended"],
      "rangeStrategy": "bump",
      "schedule": ["after 8pm on sunday"],
      "packageRules": [
        { "matchDepTypes": ["devDependencies"], "automerge": true }
      ]
    }

**生成 SBOM（CycloneDX）**

    npx @cyclonedx/cyclonedx-npm --omit dev -o sbom.json

---

## 13) 练习 🏋️
1. 在仓库接入 **Renovate**，限定每周一次窗口，观察 PR 噪音与通过率。  
2. 把安装流程改成 **冻结 + 忽略脚本**，统计构建时长与安全事件下降幅度。  
3. 为生产镜像生成 **SBOM** 并用 **OSV** 扫描；把结果作为发布门禁。  
4. 选一个“脚本重”依赖，改为 **白名单 rebuild** 流程，验证可行性。  

---

**小结**：供应链安全不是一次性扫描，而是**流程化的守门**：**锁定（可复现） → 审计（持续） → 代理隔离（准入） → 签名溯源（可证） → 最小权限（可控）**。把这五步固化进 CI/CD，你的包生态就稳了。🛡️
