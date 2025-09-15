# 12.1 GitHub Actions / GitLab CI ⚙️🤖

本章目标：把 **CI/CD** 落在两个常用平台上——**GitHub Actions** 与 **GitLab CI**。覆盖**单仓/Monorepo**、**缓存**、**矩阵**、**按改动触发**、**并行/依赖**、**环境/密钥**、**可观测产物**（测试报告/覆盖率/构建物）、**预览环境**与**发布流水线**（配合 9.x/10.x 章节）。

---

## 0) 心法（先立规矩）

- **快**：缓存依赖 & 构建产物、并行执行、只跑受影响部分（Turbo/Nx）。  
- **稳**：分阶段（lint → test → build → e2e → release），每步产物留痕。  
- **准**：按路径/变更触发；主干与 PR 策略不同；受保护环境要审批。  
- **省**：失败再抓 Trace/视频；只上传必要产物；利用 OIDC 免长期密钥。  

---

## 1) 通用目录（Monorepo 示例）

```
.
├─ apps/
│  ├─ web/        # Vite 前端
│  └─ api/        # Node 服务
├─ packages/
│  ├─ ui/         # 组件库（tsup）
│  └─ utils/
├─ .github/workflows/   # GH Actions
├─ .gitlab-ci.yml       # GitLab CI（二选一）
├─ turbo.json / nx.json
└─ pnpm-lock.yaml
```

---

## 2) GitHub Actions：最小 CI（Lint + Test + Build）

**.github/workflows/ci.yml**
```yaml
name: CI
on:
  push:
    branches: [main]
    paths:
      - "apps/**"
      - "packages/**"
      - "pnpm-lock.yaml"
      - ".github/workflows/**"
  pull_request:
    paths:
      - "apps/**"
      - "packages/**"
      - "pnpm-lock.yaml"

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  node:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      # 若要用 OIDC（发布/云部署），需要 id-token: write
      # id-token: write
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v3
        with: { version: 9 }

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "pnpm"

      - name: Install
        run: pnpm i --frozen-lockfile

      - name: Lint & Typecheck
        run: |
          pnpm turbo run lint typecheck --filter=...[origin/main]

      - name: Unit tests (Vitest)
        run: pnpm turbo run test --filter=...[origin/main] -- --coverage

      - name: Build
        run: pnpm turbo run build

      - name: Upload coverage
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-lcov
          path: "**/coverage/lcov.info"
```

要点：  
- `concurrency` 防止同分支重复跑。  
- 使用 Turbo 的 `--filter=...[origin/main]` 只测受影响包（Nx 用 `nx affected -t` 替换）。  
- 缓存：`setup-node cache: pnpm` + Actions 自带层级缓存。

---

## 3) Playwright/Cypress E2E（按需）

**.github/workflows/e2e.yml**
```yaml
name: E2E
on: [pull_request]
jobs:
  pw:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }

      - run: pnpm i --frozen-lockfile
      - run: pnpm -C apps/web build && pnpm -C apps/web preview & npx wait-on http://localhost:4173
      - run: pnpm dlx playwright install --with-deps
      - run: pnpm -C apps/web e2e # 例如：playwright test --reporter=list,html

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-artifacts
          path: |
            playwright-report/**
            **/*.zip
```

建议：`trace: on-first-retry`（见 11.2）以减少开销。

---

## 4) 发布流水线（Changesets + npm provenance）

**.github/workflows/release.yml**
```yaml
name: Release
on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write
  id-token: write   # OIDC for npm --provenance

jobs:
  release:
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
      - name: Create Version PR or Publish
        uses: changesets/action@v1
        with:
          version: pnpm changeset version
          publish: pnpm turbo run build && pnpm changeset publish
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

要点：  
- 打开 `id-token: write`，配合 `npm publish --provenance`（9.2 已讲）。  
- 通过 Changesets 自动开 Version PR 或直接发布。

---

## 5) 复用工作流（workflow_call）

**.github/workflows/_node-reusable.yml**
```yaml
name: reusable-node
on: { workflow_call: {} }

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm i --frozen-lockfile
      - run: pnpm turbo run lint test build --filter=...[origin/main]
```

调用方：
```yaml
name: CI
on: [push, pull_request]
jobs:
  use:
    uses: ./.github/workflows/_node-reusable.yml
```

---

## 6) 矩阵与分片（Node/OS/浏览器）

```yaml
strategy:
  fail-fast: false
  matrix:
    node: [18, 20]
    os: [ubuntu-latest, windows-latest]
runs-on: ${{ matrix.os }}
steps:
  - uses: actions/setup-node@v4
    with: { node-version: ${{ matrix.node }}, cache: pnpm }
```

E2E 分片（Playwright）：
```yaml
- run: pnpm playwright test --shard=${{ strategy.job-index + 1 }}/4
```

---

## 7) 可观测产物（报告/构建物/诊断）

- **测试报告**：`lcov`/`junit.xml` 入库（Codecov、Allure、Artifacts）。  
- **构建物**：上传 `dist/**` 或 Docker 镜像（缓存分层）。  
- **诊断**：Fail 才上传 Trace/视频/日志；长留 7~14 天即可。  

`actions/upload-artifact@v4` / `download-artifact@v4` 组合跨 Job 传递产物。

---

## 8) 环境与审批（GitHub Environments）

```yaml
environment:
  name: production
  url: https://app.example.com
```

- 在仓库 **Settings → Environments** 设置 `production` 需审批与密钥范围。  
- 不同环境的变量/密钥隔离（`secrets.production.*`）。

---

## 9) 只跑受影响改动（Turbo / Nx）

**GitHub**：在 CI 步骤使用  
```bash
pnpm turbo run test --filter=...[origin/main]
# 或 Nx:
pnpm nx affected -t test --base=origin/main --head=HEAD
```

**按路径触发**（已在 `on.paths` 中示例）。组合两者：**触发少、执行更少**。

---

## 10) 安全与密钥

- 优先 **OIDC** 到云/注册表，**少放长期密钥**（AWS/GCP/Azure 均支持）。  
- Secrets 最小化：分环境、定期轮换，禁止回显到日志。  
- 锁主干：保护分支 + 必需状态检查（CI、SAST、审查）。  

---

## 11) GitLab CI：同等能力的 YAML

**.gitlab-ci.yml（Monorepo 示例）**
```yaml
stages: [lint, test, build, e2e, release]

default:
  image: node:20
  before_script:
    - corepack enable
    - corepack prepare pnpm@9 --activate
    - pnpm i --frozen-lockfile
  cache:
    key:
      files:
        - pnpm-lock.yaml
    paths:
      - .pnpm-store

lint:
  stage: lint
  script:
    - pnpm turbo run lint typecheck --filter=...[origin/main]
  rules:
    - changes:
        - apps/**/*
        - packages/**/*
        - pnpm-lock.yaml

test:
  stage: test
  needs: ["lint"]
  script:
    - pnpm turbo run test --filter=...[origin/main] -- --coverage
  artifacts:
    when: always
    paths:
      - "**/coverage/lcov.info"

build:
  stage: build
  needs: ["test"]
  script:
    - pnpm turbo run build
  artifacts:
    paths:
      - apps/web/dist
      - apps/api/dist

e2e:
  stage: e2e
  image: mcr.microsoft.com/playwright:v1.48.0-jammy
  needs: ["build"]
  script:
    - pnpm -C apps/web preview & npx wait-on http://localhost:4173
    - pnpm -C apps/web e2e
  artifacts:
    when: on_failure
    paths:
      - playwright-report/

release:
  stage: release
  needs: ["build"]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - pnpm changeset version
    - pnpm turbo run build
    - npm publish --provenance --access public
  # 若使用 OIDC 到 npm/云提供商，配置 CI_JOB_JWT 或专用 OIDC 设置
```

要点（GitLab）：  
- `stages` + `needs` = 拆层并行；`rules:changes` = 路径感知。  
- 缓存依赖（基于 `pnpm-lock.yaml`），`artifacts` 传递构建与报告。  
- E2E 用官方 Playwright 镜像更省心。  
- **Review App**（预览环境）：
```yaml
review:
  stage: build
  rules:
    - if: '$CI_MERGE_REQUEST_ID'
  script:
    - ./scripts/deploy_preview.sh "$CI_ENVIRONMENT_SLUG"
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_COMMIT_REF_SLUG.preview.example.com
    on_stop: stop_review

stop_review:
  stage: build
  script: ./scripts/teardown_preview.sh "$CI_ENVIRONMENT_SLUG"
  when: manual
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
```

---

## 12) 性能小抄 🏎️

- **依赖缓存**：pnpm + 锁文件作为 cache key；Docker 用多阶段 + 层缓存。  
- **按需追踪**：测试失败才上传大产物（Trace/视频）。  
- **矩阵降维**：PR 只跑 `ubuntu + Node 20`，主干/发布再跑全矩阵。  
- **并行切片**：长测（E2E）按文件/哈希分片。  

---

## 13) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 每次全量构建 | CI 排队慢 | 结合 `paths`/`affected` 只跑受影响模块 |
| 大产物全量上传 | 存储爆炸 | 失败才留证据，构建物短期保留 |
| 把密钥写死变量 | 泄露风险 | OIDC + Environments；Secrets 只在需用 Job 暴露 |
| 步骤耦合 | 一处失败全重跑 | 分 Job + `needs`，善用 artifacts |
| PR 与主干同策略 | 浪费 | PR 跑快速集；合并后跑完整回归 |
| 缓存没声明输出 | 命中率低 | 对构建工具声明 `outputs`（Turbo/Nx）映射缓存 |

---

## 14) 提交前检查清单 ✅

- [ ] CI 对 **lint / typecheck / unit / build / e2e** 分层且可并行。  
- [ ] 依赖与构建有缓存；E2E 失败留 Trace。  
- [ ] 仅对改动路径触发；Monorepo 仅跑受影响包。  
- [ ] 环境/密钥最小权限；生产需审批。  
- [ ] 发布流水线支持 **Changesets + provenance**；产物与版本可追溯。  

---

## 15) 练习 🏋️

1. 为当前仓库加一套 **CI（GH 或 GL）**：PR 跑 Lint+Test+Build，主干再加 E2E。  
2. 把 **Turbo/Nx** 的受影响过滤接入，测量 CI 时间下降幅度。  
3. 将 **发布工作流** 改成 OIDC 免密发布，并启用 `--provenance`，验证 SBOM/追溯链路生效。  

---

**小结**：CI/CD 的精髓是**快、稳、准、可追溯**。用好缓存与过滤，用工整的分层与产物传递，用 OIDC 把密钥风险降到冰点，发布就会像自动扶梯——踏上去，稳稳到达。🛗✨
