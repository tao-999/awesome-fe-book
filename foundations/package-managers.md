# 3.3 包管理器：pnpm / npm / yarn 对比

本章目标：掌握三大包管（**pnpm / npm / Yarn（v1 与 Berry v2+）**）的**工作原理、指令对照、工作区（monorepo）、锁定与复现、CI 缓存、Peer 依赖、覆盖与补丁**，给出**选型建议 + 落地清单**。所有片段可直接拷贝。

---

## 0) 一句话结论

- **个人/团队日常开发、Monorepo 首选：`pnpm`**（硬链接 + 严谨依赖解析 + 一致跨平台）。  
- **简单单包/传统生态：`npm`**（“系统自带”、`npm ci` 稳定）。  
- **需要 PnP/Zero-Install/插件体系：`Yarn Berry`**（`nodeLinker: pnp`、.yarn/cache、插件多）。  
- **遗留项目：`Yarn v1`**（兼容性强，但已进入维护模式，优先迁出）。

---

## 1) 关键差异总览

| 维度 | pnpm | npm | Yarn v1 | Yarn Berry (v2+) |
|---|---|---|---|---|
| 锁文件 | `pnpm-lock.yaml` | `package-lock.json` | `yarn.lock` (v1) | `yarn.lock`（格式与 v1 不兼容） |
| node_modules 布局 | **内容寻址+硬链接**，严格树（`.pnpm` → symlink） | 扁平提升（hoist） | 扁平提升 | **可选 PnP（无 `node_modules`）** 或 node-modules |
| 工作区（Monorepo） | `pnpm-workspace.yaml` 一等公民 | `workspaces`（npm 7+） | `workspaces`（基础） | `workspaces` + 强力 `foreach` / constraints |
| 离线/零安装 | 有全局 store，可离线；非零安装 | 有本地缓存；非零安装 | 有缓存；非零安装 | **Zero-Install（提交 `.yarn/cache` + `.pnp.*`）** |
| 覆盖/修复 | `overrides`、`packageExtensions`、`patch` | `overrides`（npm v8+） | `resolutions` | `resolutions`、`packageExtensions`、`patch:` 协议 |
| Peer 依赖 | **更严格**（不满足常阻断；可调） | 自动安装（npm 7+），较宽松 | 警告为主 | **解析完善**，报错/修复能力强 |
| CI 冻结安装 | `pnpm i --frozen-lockfile` / `--prefer-offline` / `pnpm fetch` | `npm ci` | `yarn --frozen-lockfile` | `yarn install --immutable`（可加 `--immutable-cache`） |
| 速度/磁盘 | **快 + 省磁盘**（共享全局 store） | 一般 | 较快 | PnP 快、磁盘全进 `.yarn/cache` |

---

## 2) 基础安装与 Corepack 固定版本

**推荐开启 Corepack（Node ≥ 16.13；Node ≥ 18 体验最佳）**，确保每台机器使用同版本包管。

```bash
# 只需一次（全局）
corepack enable

# 在项目 package.json 固定包管版本
# 任选其一（示例版本请按团队规范替换）
# "packageManager": "pnpm@9.5.0"
# "packageManager": "npm@10.8.2"
# "packageManager": "yarn@4.2.2"
```

> 提交到仓库后，任何人进入项目目录执行 `pnpm|npm|yarn install` 都会自动使用锁定版本。

---

## 3) 指令对照表（常用）

| 目标 | pnpm | npm | Yarn v1 | Yarn Berry |
|---|---|---|---|---|
| 安装依赖 | `pnpm install` | `npm install` | `yarn` | `yarn install` |
| 冻结安装（CI） | `pnpm i --frozen-lockfile` | `npm ci` | `yarn --frozen-lockfile` | `yarn install --immutable` |
| 添加依赖 | `pnpm add react` | `npm i react` | `yarn add react` | `yarn add react` |
| 添加开发依赖 | `pnpm add -D vite` | `npm i -D vite` | `yarn add -D vite` | `yarn add -D vite` |
| 升级依赖 | `pnpm up react@latest` | `npm up react@latest` | `yarn upgrade react@latest` | `yarn up react@latest` |
| 移除依赖 | `pnpm remove lodash` | `npm rm lodash` | `yarn remove lodash` | `yarn remove lodash` |
| 运行脚本 | `pnpm run build` 或 `pnpm build` | `npm run build` | `yarn build` | `yarn build` |
| 跨包运行（工作区） | `pnpm -r run build` / `pnpm -r test` / `--filter` | `npm -w pkg run build` | `yarn workspaces run build` | `yarn workspaces foreach -pt run build` |
| 临时执行（不安装到 deps） | `pnpm dlx create-vite` | `npx create-vite` | `yarn dlx create-vite` | `yarn dlx create-vite` |
| 审计 | `pnpm audit` | `npm audit` | `yarn audit` | `yarn npm audit` |

---

## 4) Monorepo / Workspaces

### 4.1 pnpm（推荐）
`pnpm-workspace.yaml`：
```yaml
packages:
  - "apps/*"
  - "packages/*"
```

典型脚本：
```bash
pnpm -r run build                   # 递归所有包
pnpm -r test --filter ./packages/ui # 只测某包
pnpm -r exec tsc -b                 # 在每个子包执行命令
```

### 4.2 npm
`package.json`：
```json
{ "workspaces": ["apps/*", "packages/*"] }
```
```bash
npm run build --workspaces
npm run build -w @acme/ui
```

### 4.3 Yarn Berry
`.yarnrc.yml`（可选 PnP）：
```yml
nodeLinker: pnp   # 或 node-modules
```
```bash
yarn workspaces foreach -pt run build
```

---

## 5) 解析与 node_modules 布局

- **npm / Yarn v1**：扁平 hoist，子依赖常被提升到顶层，**幽灵依赖**易发生（包 A 间接用到包 B，但 B 未在 A 的 `package.json` 声明也能跑起来）。  
- **pnpm**：使用**内容寻址全局存储**（`~/.pnpm-store`），项目里的 `node_modules` 通过硬链接与符号链接指向 `.pnpm` 下的真实版本路径，**不提供幽灵依赖**，问题更早暴露。  
  - 兼容旧包：可按需设置 `public-hoist-pattern` 或 `shamefully-hoist` 做有限提升。  
- **Yarn Berry**：推荐 **PnP**（无 `node_modules`，依赖从 `.pnp.cjs`/`.pnp.data` 映射加载），解析速度快且更严格；也可切回 `node-modules`。

---

## 6) Peer Dependencies 行为

- **pnpm**：对未满足的 `peerDependencies` 倾向报错/阻断（可通过 `pnpm.peerDependencyRules.*`、`public-hoist-pattern` 等缓解）。  
- **npm(7+)**：会尝试自动安装 peer，失败时再报冲突。  
- **Yarn Berry**：解析与报错信息更友好，支持 `packageExtensions` 给第三方包补足缺失的 peer。

> 团队建议：**显式声明你直接消费的 peer**（在你自己的 `package.json` 里写全），减少隐式依赖。

---

## 7) 覆盖 / 补丁 / 修包

### 7.1 版本覆盖（顶层强制替换）

**pnpm / npm（`overrides`）**
```json
{
  "overrides": {
    "minimist": "^1.2.8",
    "foo > bar": "2.0.0"   // 只覆盖 foo 依赖链下的 bar
  }
}
```

**Yarn（`resolutions`）**
```json
{
  "resolutions": {
    "minimist": "^1.2.8"
  }
}
```

### 7.2 给第三方包补依赖或修 `peer`（packageExtensions）

**pnpm**
```json
{
  "pnpm": {
    "packageExtensions": {
      "some-legacy-lib@*": {
        "peerDependencies": { "react": ">=18" }
      }
    }
  }
}
```

**Yarn Berry**
```yml
# .yarnrc.yml
packageExtensions:
  "some-legacy-lib@*":
    peerDependencies:
      react: ">=18"
```

### 7.3 制作补丁（不等上游）

**pnpm**
```bash
pnpm patch some-lib@1.2.3
# 编辑后保存
pnpm patch-commit some-lib@1.2.3
```

**Yarn Berry**
```bash
yarn patch some-lib@1.2.3
yarn patch-commit some-lib@1.2.3
# package.json 可使用 "patch:some-lib@npm:1.2.3#<hash>"
```

---

## 8) 复现性与 CI

- **npm**：`npm ci` 完全基于 `package-lock.json`，最快最稳。  
- **pnpm**：  
  - `pnpm fetch`：基于 `pnpm-lock.yaml` 预取到 store（CI 可缓存 store 目录），随后 `pnpm i --offline` 几乎瞬间。  
  - `pnpm i --frozen-lockfile`：锁文件不可变。  
- **Yarn Berry**：`yarn install --immutable` + 缓存 `.yarn/cache`；如走 Zero-Install，可直接 `git clone && yarn run build`。

**GitHub Actions 缓存示例（pnpm）**：
```yaml
- name: Get pnpm store
  run: echo "PNPM_STORE_PATH=$(pnpm store path --silent)" >> $GITHUB_ENV
- uses: actions/cache@v4
  with:
    path: ${{ env.PNPM_STORE_PATH }}
    key: ${{ runner.os }}-pnpm-${{ hashFiles('pnpm-lock.yaml') }}
    restore-keys: ${{ runner.os }}-pnpm-
- run: pnpm fetch
- run: pnpm install --offline --frozen-lockfile
```

---

## 9) 安全与审计

- `npm audit` / `pnpm audit` / `yarn npm audit`：基于 npm advisory 数据。  
- 大型仓库建议引入**组织级策略**：锁定 registry、强制 `overrides`、禁止可疑协议（`git+ssh` 对生产锁定等）。

---

## 10) 零安装（Yarn Berry 特色）

- 提交 `.yarn/cache` 与 `.pnp.*` 后，新人**无需 install**。  
- 注意：仓库体积变大、与某些工具（老式捆绑器/脚本）不兼容时需切回 `node-modules`。  
- 配置切换：
```yml
# .yarnrc.yml
nodeLinker: node-modules   # 从 PnP 切回传统模式
```

---

## 11) 迁移指北

### npm → pnpm
```bash
pnpm import                # 从 package-lock.json 生成 pnpm-lock.yaml
pnpm i
```
- 若第三方包依赖幽灵依赖，临时设置：
```json
{
  "pnpm": {
    "publicHoistPattern": ["*"]  // 或 "shamefullyHoist": true
  }
}
```
逐步回收至严格模式。

### Yarn v1 → pnpm
- 直接 `pnpm import`（支持从 `yarn.lock` 生成新锁）。  
- 删除 `.yarn`、`yarn.lock`，提交 `pnpm-lock.yaml`。

### Yarn v1 → Yarn Berry
```bash
yarn set version stable
# 生成 .yarnrc.yml、.yarn/*
```
按需设 `nodeLinker` 与 `immutable`。

---

## 12) 实用配置清单（可拷贝）

**.npmrc（npm）**
```
engine-strict=true
audit=true
fund=false
```

**.yarnrc.yml（Yarn Berry）**
```yml
nodeLinker: pnp         # 或 node-modules
enableInlineHunks: true
yarnPath: .yarn/releases/yarn-4.x.cjs
```

**.pnpmrc（pnpm）**
```
strict-peer-dependencies=true
prefer-offline=true
auto-install-peers=false
```

**pnpm-workspace.yaml**
```yaml
packages:
  - "apps/*"
  - "packages/*"
```

---

## 13) CLI 小抄（工作区常用）

```bash
# pnpm：按图过滤
pnpm -r run build --filter @acme/ui
pnpm -r exec tsc -b
pnpm why react                 # 依赖来源分析
pnpm store prune               # 清理全局 store

# npm：工作区
npm run test --workspaces
npm run build -w @acme/ui

# Yarn Berry：
yarn workspaces foreach -pt run build
yarn why react
yarn cache clean
```

---

## 14) 选型建议（场景化）

- **Monorepo + 追求速度/体积** → **pnpm**。  
- **公司统一镜像 + 零痛点上手** → **npm**（配合 `npm ci` ）。  
- **强约束依赖解析 / Zero-Install / 插件生态** → **Yarn Berry**。  
- **历史项目** → 保留 `Yarn v1` 但计划迁移（优先到 pnpm 或 Berry）。

---

## 15) 提交前检查清单 ✅

- [ ] `packageManager` 字段锁定包管与版本（Corepack）。  
- [ ] 仅保留一种锁（`pnpm-lock.yaml` / `package-lock.json` / `yarn.lock`）。  
- [ ] CI 使用冻结安装（`npm ci` / `pnpm --frozen-lockfile` / `yarn --immutable`）。  
- [ ] 若从 npm/Yarn 迁移到 pnpm，先开启 `publicHoistPattern` 过渡再收紧。  
- [ ] 对直接消费的 `peerDependencies` **显式声明**。  
- [ ] 使用 `overrides/resolutions/packageExtensions/patch` 治理供应链。  

---

## 16) 练习 🏋️

1. 把现有仓库切换到 **pnpm + workspace**；在 CI 接入 `pnpm fetch + --offline`。  
2. 用 `overrides`/`resolutions` 修一次真实的漏洞告警并写复盘。  
3. 对比 `npm ci`、`pnpm --frozen-lockfile`、`yarn --immutable` 的冷启动时间与缓存命中率，形成团队基线报表。

---
