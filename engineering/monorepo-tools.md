# 9.1 Turborepo / Nx / Changesets 🧩

本章目标：把 **Monorepo 任务编排（Turborepo / Nx）** 与 **版本发布（Changesets）** 三件套串成一条顺滑产线：  
- 本地与 CI **增量构建 + 分布式缓存**  
- **依赖图**驱动的精准执行（只跑受影响包）  
- **语义化版本**与**自动发版**（含预发布通道）

---

## 0) 心智图（先立规矩）

- **包管理器**：用 `pnpm`（workspaces + 去重）  
- **任务编排**：小而美 → **Turborepo**；平台化、代码生成、深度集成 → **Nx**  
- **版本发版**：用 **Changesets** 管理 **patch/minor/major**，生成 Changelog，自动发 npm  
- **依赖约束**：包间用 `workspace:*` / `workspace:^`，禁止相对路径地狱

---

## 1) Monorepo 基线骨架

```
repo/
  package.json
  pnpm-workspace.yaml
  turbo.json or nx.json
  .changeset/            # Changesets 管理目录
  packages/
    ui/                  # 组件库
    utils/               # 工具库
  apps/
    web/                 # Web 应用
    api/                 # 后端/边缘服务
```

**pnpm-workspace.yaml**
```yaml
packages:
  - "packages/*"
  - "apps/*"
```

**根 package.json（常用脚本）**
```json
{
  "private": true,
  "packageManager": "pnpm@9",
  "scripts": {
    "dev": "turbo run dev --parallel",
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "typecheck": "turbo run typecheck"
  }
}
```

---

## 2) Turborepo：任务编排 + 远程缓存 ⚡

### 2.1 安装与最小配置
```bash
pnpm add -D turbo
```

**turbo.json**
```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],           // 先构建依赖包
      "outputs": ["dist/**"]             // 声明构建产物（用于缓存）
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"]
    },
    "lint": { "outputs": [] },
    "typecheck": { "outputs": [] },
    "dev": {
      "cache": false,                    // dev 不缓存
      "persistent": true                 // 长时间运行任务
    }
  }
}
```

### 2.2 过滤与选择（**超好用**）
```bash
# 只构建 apps/web 及其所有依赖（上游）
turbo run build --filter=apps/web^...

# 只构建依赖于 packages/ui 的下游包
turbo run build --filter=packages/ui...

# 仅构建本分支改动影响到的包
turbo run build --filter=...[origin/main]
```

> 记忆法：`^...` = **往上**（依赖）；`...` = **往下**（依赖它的包）。

### 2.3 远程缓存（可选）
- 开启远程缓存可让 CI 与本地共享结果（**极大**提速）。  
- 典型配置：设置团队/令牌环境变量（如 `TURBO_TEAM`、`TURBO_TOKEN`），CI 注入同样的凭据后，构建命中率飞起。  
- 缓存命中基于：**命令、输入（源文件/锁文件）、环境变量、输出声明**。

---

## 3) Nx：平台化 Monorepo 🔧

当你需要**可视化依赖图、代码生成器、规范化目标（targets）、更细的输入跟踪**，Nx 很香。

### 3.1 初始化与核心文件
```bash
pnpm dlx nx@latest init
```

**nx.json（片段）**
```json
{
  "tasksRunnerOptions": {
    "default": { "runner": "nx/tasks-runners/default", "options": { "cacheableOperations": ["build","test","lint","typecheck"] } }
  },
  "targetDefaults": {
    "build": { "dependsOn": ["^build"], "inputs": ["default", "^default"], "outputs": ["{projectRoot}/dist"] }
  }
}
```

**项目 targets（package.json 或 project.json）**
```json
{
  "name": "@acme/ui",
  "nx": {
    "targets": {
      "build": { "executor": "@nx/js:tsc" },
      "test":  { "executor": "@nx/vite:test" }
    }
  }
}
```

### 3.2 常用命令
```bash
nx graph                       # 打开交互式依赖图
nx affected -t build           # 仅构建受影响项目
nx run-many -t test -p ui,utils --parallel=3
nx g @nx/js:library ui         # 生成器创建库
```

### 3.3 Nx Cloud（可选）
- 一键打开远程缓存与分布式执行：设置 `NX_CLOUD_AUTH_TOKEN`。  
- 与 CI 共享 cache，效果与 Turbo 类似。

> **Turborepo vs Nx**：  
> - 你要**轻量编排 + 过滤 + 缓存** → Turborepo。  
> - 你要**强图谱、生成器、深度插件（React/Node/Vite/Storybook 等官方 executors）** → Nx。

---

## 4) 包之间的依赖与 TS 配置 📐

- `package.json` 里使用：`"dependencies": {"@acme/utils":"workspace:^"}`  
- TypeScript：推荐每包自有 `tsconfig.json`，需要 **project references** 时在根 `tsconfig` 配置 `"references"`，并配合 `tsc -b` 或 bundler 执行器。  
- 统一导出：各包配好 `"exports"` / `"types"`，避免深路径导入。

---

## 5) Changesets：语义化版本与自动发布 🚀

### 5.1 初始化
```bash
pnpm add -D @changesets/cli
pnpm changeset init
```

生成 `.changeset/config.json`（关键字段）
```json
{
  "baseBranch": "main",
  "changelog": "@changesets/changelog-github",
  "commit": false,
  "linked": [],
  "access": "public" // 私有包设为 "restricted"
}
```

### 5.2 开发者提 PR 时写变更
```bash
pnpm changeset
# 交互选择：要 bump 的包 + bump 级别（patch/minor/major）
# 生成一个 .changeset/xxxx.md
```

`.changeset/near-cats-sing.md` 示例：
```md
---
"@acme/ui": minor
"@acme/utils": patch
---

UI 新增 <Tabs/>；utils 修复 parseDate 边界问题。
```

### 5.3 版本与发布脚本
**根 package.json**
```json
{
  "scripts": {
    "changeset": "changeset",
    "version-packages": "changeset version",
    "release": "turbo run build --filter=...[HEAD] && changeset publish"
  }
}
```

- `changeset version`：读取 `.changeset/*.md`，**改写各包版本**与依赖范围，并生成 **CHANGELOG**。  
- `changeset publish`：按更新过的包逐一 `npm publish`（默认从 `./dist` 或包自身目录发布）。

### 5.4 预发布通道（beta/next）
```bash
pnpm changeset pre enter next   # 进入预发布模式，版本形如 1.2.0-next.0
# 开发、写 changeset、发几版...
pnpm changeset pre exit         # 退出预发布模式，恢复正常语义化版本
```

### 5.5 GitHub Actions 自动发版（推荐）
`.github/workflows/release.yml`
```yaml
name: Release
on:
  push:
    branches: [main]
jobs:
  version_or_publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: https://registry.npmjs.org
          cache: pnpm
      - run: pnpm i --frozen-lockfile
      - name: Create Release PR or Publish
        uses: changesets/action@v1
        with:
          version: pnpm version-packages
          publish: pnpm release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

> 工作流逻辑：  
> - 有未发布的 changeset → 自动开一个 **Version PR**（改版本号 + 生成 Changelog）。  
> - 合并到 main → 触发 **publish**，构建并 `npm publish`。

---

## 6) CI 组合拳（增量 + 缓存）🤖

以 Turborepo 为例（Nx 类似）：

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm i --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm test
      - run: pnpm build
        env:
          # 如启用 Turbo 远程缓存，这里注入令牌即可共享缓存
          TURBO_TEAM: ${{ secrets.TURBO_TEAM }}
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
```

> 加速点：  
> - **远程缓存** 命中后，`build/test` 在 CI 直接“快进”。  
> - 拆分多个 Job 并发执行（lint/typecheck/test），最终汇总结果。

---

## 7) 约定与策略（实践共识）📏

- **脚本名统一**：每包都实现 `dev/build/test/lint/typecheck`，顶层编排才不混乱。  
- **输出目录一致**：库统一 `dist/`，应用可以自定义。  
- **依赖走 workspace**：`"@acme/utils":"workspace:^"`；不要 `file:../utils` 或相对导入。  
- **Changelog 可读**：在 changeset 摘要里写“**面向用户**”的变化，不是内部实现细节。  
- **版本策略**：库倾向 **SemVer**；若多个包要**同版本**，在 `.changeset/config.json` 里用 `"linked": [["@acme/ui","@acme/utils"]]`。  
- **构建与类型检查解耦**：打包交给 bundler/executor，类型检查用 `tsc --noEmit` 独立跑。

---

## 8) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 包间相对路径导入 | 版本/缓存错乱、IDE 路径地狱 | 用 `workspace:*`，并发布后用包名导入 |
| 每个包脚本名各写各的 | 顶层命令无法编排 | 统一 `dev/build/test/lint/typecheck` |
| 不声明 `outputs` | 缓存命中率低 | 在 Turbo/Nx 中为每个任务声明产物 |
| 全量构建 | CI 慢到怀疑人生 | 用 **affected**/过滤 + 远程缓存 |
| 直接 `npm version` | 版本与变更不同步 | 统一使用 **Changesets** |
| 将服务器状态/环境嵌到库构建 | 产物不可复用 | 库只做纯函数/纯 UI，环境注入在应用层 |

---

## 9) 提交前检查清单 ✅

- [ ] `pnpm-workspace.yaml` 覆盖所有包；包间依赖使用 `workspace:*`。  
- [ ] Turborepo/Nx 的 **pipeline/targets** 完整，`build` 依赖 `^build`。  
- [ ] 声明 **outputs**（构建/测试产物），命中缓存验证通过。  
- [ ] CI 采用 **增量执行** +（可选）**远程缓存**。  
- [ ] Changesets 已初始化；有变更就写 `.changeset/*.md`；发布走自动流水线。  
- [ ] Changelog 语言面向用户；语义化版本符合预期（含预发布流程）。  

---

## 10) 练习 🏋️

1. 在现有 Monorepo 加入 **Turborepo**，为 `build/test/lint/typecheck` 声明 outputs，测一次本地与 CI 缓存命中率。  
2. 把项目迁到 **Nx**：生成 `ui` 库与 `web` 应用，用 `nx affected -t build` 验证只构建被改动影响的项目。  
3. 初始化 **Changesets**：为 `@acme/ui` 新增组件并写一个 `minor` changeset，合并后自动发版到 npm；再尝试一次 `pre enter next` 的预发布流程。  

---

**小结**：**Turborepo/Nx** 负责“**快**与**准**”，**Changesets** 负责“**稳**与**清楚**”。三者合力，把 Monorepo 从“工具堆”升格为“生产线”：谁改动、谁构建；谁变化、谁发版。🏗️🚀
