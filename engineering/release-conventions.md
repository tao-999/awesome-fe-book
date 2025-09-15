# 9.2 包版本 / 发布 / 约定式提交 📦🚀

本章目标：把 **语义化版本（SemVer）**、**发布流水线（npm / CI）**、**约定式提交（Conventional Commits）** 三板斧拧成一根绳——从「写代码」到「上线可回滚、有记录、可追溯」。

---

## 0) 心智图（先拍板）

- **版本**：遵循 **SemVer**（`MAJOR.MINOR.PATCH`）。破坏性变更就 **MAJOR**。  
- **发布**：最小化产物（`files` 白名单 / `exports` 映射 / 双格式 ESM+CJS），**打 tag**、**发 npm**、**生成 changelog**。  
- **提交**：用 **约定式提交** 自动推导版本与变更日志；**校验 + 钩子** 保持团队一致性。  
- **通道**：稳定 `latest`；预发布 `next` / `alpha` / `beta`；**dist-tag** 管理多轨。

---

## 1) 语义化版本（SemVer 2.0）与范围 🎯

- **规则**：  
  - `PATCH`（x.y.**z**）：Bug 修复、无 API 变化。  
  - `MINOR`（x.**y**.z）：向后兼容的新功能。  
  - `MAJOR`（**x**.y.z）：破坏性变更（API/默认行为）。  
- **范围运算**（依赖里怎么写）：  
  - `^1.2.3`：锁 **主版本**，允许次/补丁升级（最常用）。  
  - `~1.2.3`：锁到 **次版本**，只允许补丁升级。  
  - `1.2.x`：锁定前两位。  
  - 精准 `1.2.3`：**固定**版本（常见于工具链/可复现实验）。  
- **Monorepo / workspace**：  
  - 包间依赖用 `workspace:^` / `workspace:*`，发布后仍解析为正常范围。  
- **peerDependencies**：对宿主框架（如 React/Vue）用 `peerDependencies` + 对应的 `peerDependenciesMeta.optional`（可选时），避免重复打包与多实例冲突。

---

## 2) `package.json` 发布要点模板 🧩

```json
{
  "name": "@acme/awesome",
  "version": "0.0.0",
  "description": "Awesome lib",
  "license": "MIT",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.esm.js",
      "require": "./dist/index.cjs"
    }
  },
  "main": "./dist/index.cjs",
  "module": "./dist/index.esm.js",
  "types": "./dist/index.d.ts",
  "sideEffects": false,
  "files": ["dist", "README.md", "LICENSE"],
  "engines": { "node": ">=18" },
  "repository": { "type": "git", "url": "git+https://github.com/acme/awesome.git" },
  "bugs": { "url": "https://github.com/acme/awesome/issues" },
  "homepage": "https://github.com/acme/awesome#readme",
  "keywords": ["frontend", "awesome"],
  "peerDependencies": { "react": "^18.2.0" },
  "peerDependenciesMeta": { "react": { "optional": false } },
  "scripts": {
    "build": "tsup",
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "release": "npm run typecheck && npm run build && npm publish --provenance --access public"
  },
  "publishConfig": { "access": "public" }
}
```

> 要点：  
> - **双格式**输出 + **`exports` 映射**（条件导出）；`sideEffects:false` 方便摇树。  
> - 用 **`files` 白名单** 控制发包内容，别靠 `.npmignore` 灰度过滤。  
> - `--provenance` 开启 **供应链追溯**（GitHub OIDC + npm provenance）。

---

## 3) 发布流程（本地与 CI）🛫

### 3.1 本地手动（单包）
```bash
# 登录与 2FA（建议开启）
npm login
npm config set sign-git-tag true

# 版本号（不使用 Changesets / standard-version 时）
npm version patch|minor|major -m "chore(release): %s"

# 发布（公开/组织作用域需要 --access public）
npm publish --provenance --access public

# 推送 tag
git push --follow-tags
```

### 3.2 多通道与 dist-tag
```bash
# 预发布（alpha/beta/rc）
npm version prerelease --preid=alpha
npm publish --tag alpha --provenance

# 稳定发布切到 latest
npm dist-tag add @acme/awesome@1.2.3 latest
npm dist-tag ls @acme/awesome
```

### 3.3 CI（概念）
- 步骤：**checkout → Node + pnpm → 安装 → 测试/构建 → 版本/Changelog → publish**。  
- 注入：`NPM_TOKEN`（发布）、`GITHUB_TOKEN`（创建 Release）。  
- 与 **Changesets** 或 **standard-version** 二选一，不要混用。

---

## 4) 预发布策略（Canary / Next）🧪

- **场景**：想让早鸟用户先吃到变更。  
- **做法**：  
  - 版本：`1.3.0-next.0`、`2.0.0-rc.1` 等（带 **preid**）。  
  - 标签：发布时用 `--tag next`，稳定版仍是 `latest`。  
  - 文档：标注“**实验性** / 可能破坏 API”。

---

## 5) Git 标签与发行说明 🏷️

- `git tag v1.2.3 -m "v1.2.3"` → `git push --tags`。  
- GitHub Releases：从 tag 生成 **Release**，附上自动生成的 **CHANGELOG**（由 Changesets / standard-version 产出）。

---

## 6) 约定式提交（Conventional Commits）📝

**格式**
```
<type>(<scope>): <subject>
<BLANK LINE>
<body>
<BLANK LINE>
BREAKING CHANGE: <描述破坏性变更>
```

**常用 type**：
- `feat` 新功能 → **MINOR**  
- `fix` 修复 → **PATCH**  
- `perf` 性能优化（通常 PATCH）  
- `refactor` 重构（非特性/修复）  
- `docs` 文档、`style` 样式、`test` 测试、`build` 构建链路、`ci` 持续集成、`chore` 杂务、`revert` 回滚  
- **破坏性变更**：`feat!: ...` 或在 body 写 `BREAKING CHANGE:` 段落 → **MAJOR**

**示例**
```
feat(card): 新增 hover 阴影与紧凑尺寸

在 tokens 中引入 spacing.sm，默认尺寸保持兼容。
```

```
fix(api): 避免分页参数为 0 时抛错

BREAKING CHANGE: 移除了 page=0 的历史特殊语义，请从 1 开始计数。
```

---

## 7) 提交校验与交互面板（commitlint + commitizen + husky）🧰

### 7.1 安装与配置
```bash
pnpm add -D @commitlint/{cli,config-conventional} commitizen cz-git husky lint-staged
```

**commitlint.config.cjs**
```js
module.exports = { extends: ['@commitlint/config-conventional'] };
```

**package.json 片段**
```json
{
  "scripts": {
    "prepare": "husky install",
    "commit": "cz"
  },
  "config": {
    "commitizen": { "path": "cz-git" }
  },
  "lint-staged": {
    "*.{ts,tsx,js,jsx,vue,md,css}": ["prettier --write"]
  }
}
```

### 7.2 Husky 钩子
```bash
# 安装钩子
pnpm prepare
npx husky add .husky/commit-msg "pnpm commitlint --edit $1"
npx husky add .husky/pre-commit "pnpm lint-staged"
```

> 结果：每次提交 **必须**符合规范；改动文件自动格式化。团队从此“写成一行”。

---

## 8) 自动化发版（不与 Changesets 混用时）🤖

选择 **standard-version**（根据约定式提交生成版本与 changelog）：

```bash
pnpm add -D standard-version
```

**package.json**
```json
{
  "scripts": {
    "release": "standard-version",
    "release:minor": "standard-version --release-as minor",
    "release:major": "standard-version --release-as major",
    "publish:npm": "npm publish --provenance --access public"
  }
}
```

流程：
```bash
pnpm release         # 读提交记录 → 更新版本 + CHANGELOG + tag
git push --follow-tags
pnpm publish:npm
```

> **注意**：你在 9.1 已用 **Changesets**，推荐继续用它管理版本与发版；**不要**与 standard-version 同时驱动，避免冲突。

---

## 9) 依赖与版本策略（实战）🧠

- **框架类**（React/Vue）→ `peerDependencies` + `^` 范围；应用侧实际安装。  
- **工具链**（eslint/tsup 等）→ 开发依赖固定或 `~`，确保构建稳定。  
- **多包对齐**：需要统一版本号的包，用 Changesets `linked` 或手动同步。  
- **最小支持环境**：在 `engines.node` 指明；必要时在 CI 做多版本矩阵测试。  
- **导出策略**：优先 `exports`；提供类型与双格式；CJS 消费者需要时保留 `main`。  
- **破坏性变更**：写清迁移指南（`MIGRATION.md` / Release notes）。

---

## 10) 安全与合规 🛡️

- **2FA**：npm 账号开启双因子（发布更安全）。  
- **provenance**：`npm publish --provenance` 生成 **供应链证明**。  
- **发包内容**：只白名单 `dist` 与必需文件；检查 `.env`/测试夹不被发布。  
- **许可证**：选择兼容的开源协议（MIT/Apache-2.0 等），放 `LICENSE`。  
- **依赖审计**：在 CI 跑 `pnpm audit`（或 SCA 工具），高危尽快修复。  
- **私有包**：`publishConfig.access: "restricted"`；组织作用域 `@org/*`。

---

## 11) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 不写 `exports`，只留 `main` | 子路径地狱/兼容性差 | 用 `exports` + 条件导出（types/import/require） |
| 发一堆源码/测试 | 包大且泄露内部实现 | 用 `files` 白名单，只发 `dist` |
| 把 React/Vue 打进库里 | 多实例冲突、包大 | 用 `peerDependencies` |
| 破坏性变更悄悄发 minor | 用户炸锅 | 走 **MAJOR** + `BREAKING CHANGE` + 迁移指南 |
| `latest` 发实验版 | 回滚困难 | 用 `--tag next`/`alpha`，稳定版用 `latest` |
| standard-version 与 Changesets 混用 | 版本冲突 | **二选一**，团队统一工具 |

---

## 12) 提交前检查清单 ✅

- [ ] 版本变更符合 **SemVer**；破坏性标注清晰。  
- [ ] `package.json`：`exports` / `types` / `files` / `sideEffects:false` / `engines` 到位。  
- [ ] 依赖分层：框架在 `peerDependencies`；工具在 `devDependencies`。  
- [ ] 提交信息符合 **约定式提交**；钩子（commitlint/husky）正常生效。  
- [ ] 产物可用（本地 `npm pack` 验证 tarball 内容）。  
- [ ] 发布通道与 **dist-tag** 正确（`latest` vs `next`）。  
- [ ] CI 注入 `NPM_TOKEN`，使用 `--provenance`；tag 与 Release 已同步。

---

## 13) 练习 🏋️

1. 给你的小库补上 `exports` 映射与 `files` 白名单，跑一次 `npm pack` 查看包内文件。  
2. 配置 commitlint + husky + cz-git，写 3 条规范提交，并用 standard-version/Changesets 生成 changelog。  
3. 发布一版 `1.1.0-next.0` 到 `next` 通道；收集反馈后发布稳定版到 `latest`，对比消费者安装行为。

---

**小结**：**版本可读、提交流水线化、发布可追溯**，包才算“可维护的产品”。让工具替你记账，脑子留给设计与代码。🧠✨
