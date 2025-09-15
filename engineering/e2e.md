# 11.2 端到端（Playwright / Cypress）🛰️🧪

本章目标：把 **E2E 测试**变成“可靠的用户模拟器”，覆盖**关键业务路径**与**跨页交互**，并且**稳定不抖**、**跑得快**、**定位准**。聚焦 **Playwright** 与 **Cypress** 的实战对照：**选型、项目结构、定位器、网络拦截、鉴权、数据准备、并行与缓存、可视化追踪、CI 集成、反模式**。

---

## 0) 心法：E2E 不是“点点点”而是“验价值”
- **只测最重要的用户旅程**：登录、下单、支付、发布、搜索等。  
- **一次只检验一个“承诺”**：页面是否能完成任务（而不是像素级 UI）。  
- **少 Mock、多隔离**：外部三方（支付、短信）可假环境；**你自己家的后端尽量真连**或走独立测试环境。  
- **可观测**：失败能复现、日志/trace 清晰、截图/视频齐全。  

---

## 1) 选型速览 🎯

| 维度 | Playwright | Cypress |
|---|---|---|
| 运行内核 | 自带 Test Runner；Chromium/Firefox/WebKit 原生多浏览器 | 自带 Runner（基于 Chrome 系）+ Electron；多浏览器支持逐渐完善 |
| 速度/并行 | 开箱并行、分片、项目矩阵 | 并行需 CI 切分；官方 Dashboard 提供更强并发（商业） |
| 定位器 | `locator()`、可访问性优先（`getByRole`） | `cy.get()/find()` 为主，也支持 `cy.findByRole`（Testing Library） |
| 网络拦截 | `page.route` / `route.fulfill` 强大灵活 | `cy.intercept` 简洁易用 |
| 追踪 | Trace（视频+快照+网络）一体化 | 视频/截图内置；网络面板友好；第三方可加 |
| 组件测试 | 有（@playwright/experimental-ct-*） | 成熟（Cypress Component Testing） |
| 学习曲线 | 偏工程化 | 偏易用、交互式 Runner 体验好 |

> 现代 Web 应用：**Playwright 默认首选**（跨浏览器、trace 强、CI 友好）。已有 Cypress 基建/付费 Dashboard 的团队继续 Cypress 也很好。

---

## 2) 项目结构建议 🗂️

```
e2e/
  playwright.config.ts | cypress.config.ts
  helpers/             # 封装通用动作：登录、拦截、数据构造
  fixtures/            # 测试数据、鉴权快照
  pages/               # Page Object / Screenplay 模式
  specs/
    auth.spec.ts
    checkout.spec.ts
    search.spec.ts
```

> **Page Object ≠ 大而全**：每页只暴露“**用户能干什么**”，不要把 DOM 细节泄漏给用例。

---

## 3) Playwright 快速上手 ⚡

### 3.1 安装
```bash
pnpm dlx playwright@latest install
pnpm add -D @playwright/test
```

### 3.2 配置（`playwright.config.ts`）
```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e/specs',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  reporter: [['list'], ['html', { open: 'never' }]],
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:5173',
    trace: 'on-first-retry',    // 失败重试才抓 trace，节省资源
    video: 'retain-on-failure',
    screenshot: 'only-on-failure',
    viewport: { width: 1280, height: 800 },
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox',  use: { ...devices['Desktop Firefox'] } },
    // { name: 'webkit', use: { ...devices['Desktop Safari'] } },
    // { name: 'mobile', use: { ...devices['Pixel 7'] } }
  ],
  workers: process.env.CI ? 4 : undefined
});
```

### 3.3 最小用例（`specs/auth.spec.ts`）
```ts
import { test, expect } from '@playwright/test';

test('用户可以登录并看到个人页', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('邮箱').fill('user@example.com');
  await page.getByLabel('密码').fill('hunter2');
  await page.getByRole('button', { name: '登录' }).click();

  await expect(page).toHaveURL(/\/account/);
  await expect(page.getByRole('heading', { name: /欢迎/i })).toBeVisible();
});
```

### 3.4 鉴权加速：存储态复用
```ts
// e2e/helpers/auth.ts
import { test as base } from '@playwright/test';

export const authTest = base.extend<{ authPage: void }>({
  storageState: async ({}, use) => {
    // 准备已登录的 storageState.json（一次登录后保存）
    await use('e2e/fixtures/storageState.json');
  }
});

// specs/need-auth.spec.ts
import { expect } from '@playwright/test';
import { authTest as test } from '../helpers/auth';

test('已登录用户直达订单页', async ({ page }) => {
  await page.goto('/orders');
  await expect(page.getByText('我的订单')).toBeVisible();
});
```

> 生成 `storageState.json`：在单独测试里登录一次并 `await context.storageState({ path: '...' })`。

### 3.5 网络拦截（仅对三方或不稳定依赖）
```ts
test('搜索接口返回空结果时提示', async ({ page }) => {
  await page.route('**/api/search**', async route => {
    await route.fulfill({ json: { items: [] } });
  });
  await page.goto('/search?q=anything');
  await expect(page.getByText('没有找到结果')).toBeVisible();
});
```

---

## 4) Cypress 快速上手 🌲

### 4.1 安装
```bash
pnpm add -D cypress @testing-library/cypress
```

### 4.2 配置（`cypress.config.ts`）
```ts
import { defineConfig } from 'cypress';

export default defineConfig({
  e2e: {
    baseUrl: process.env.BASE_URL || 'http://localhost:5173',
    specPattern: 'e2e/specs/**/*.cy.{ts,js}',
    supportFile: 'e2e/support/e2e.ts',
    video: true,
    screenshotsFolder: 'e2e/artifacts/screens',
    videosFolder: 'e2e/artifacts/videos'
  }
});
```

`e2e/support/e2e.ts`：
```ts
import '@testing-library/cypress/add-commands'; // cy.findByRole 等
```

### 4.3 最小用例（`specs/auth.cy.ts`）
```ts
describe('登录', () => {
  it('成功登录后进入个人页', () => {
    cy.visit('/login');
    cy.findByLabelText('邮箱').type('user@example.com');
    cy.findByLabelText('密码').type('hunter2');
    cy.findByRole('button', { name: '登录' }).click();

    cy.url().should('match', /\/account/);
    cy.findByRole('heading', { name: /欢迎/i }).should('be.visible');
  });
});
```

### 4.4 `cy.intercept` 网络拦截
```ts
it('空结果提示', () => {
  cy.intercept('GET', '**/api/search*', { items: [] }).as('search');
  cy.visit('/search?q=abc');
  cy.wait('@search');
  cy.contains('没有找到结果').should('be.visible');
});
```

### 4.5 会话缓存（减少重复登录）
```ts
beforeEach(() => {
  cy.session('user', () => {
    cy.visit('/login');
    cy.findByLabelText('邮箱').type('user@example.com');
    cy.findByLabelText('密码').type('hunter2');
    cy.findByRole('button', { name: '登录' }).click();
    cy.url().should('include', '/account');
  });
});
```

---

## 5) 定位器与可访问性 ♿

**优先级**：`getByRole`/`getByLabelText` > 文本 > `data-testid` > CSS/XPath。  
- Playwright：`page.getByRole('button', { name: '提交' })`、`locator('[data-testid="foo"]')`。  
- Cypress：`cy.findByRole('button', { name: '提交' })`（Testing Library 命令）。

> 给复杂控件补 **`aria-label`/`aria-labelledby`** 或 `data-testid`。**不要**依赖 `.class` 与 `nth-child`。

---

## 6) 测试数据与环境准备 🧱

- **测试环境**：独立后端、固定种子数据、可“重置”端点（`POST /__reset`）。  
- **数据构造**：用后端 **工厂接口 / seed 脚本** 创建用户/订单；不要让 E2E 通过 UI 一步步点出数据。  
- **幂等**：创建前先检查/清理；避免跨用例污染。  
- **时钟控制**：使用 Playwright `page.clock`（实验）或业务层提供“固定时间”开关；Cypress 有 `cy.clock()`。

---

## 7) 并行、分片与缓存 🏎️

**Playwright**
```bash
# 4 个 worker 并行、根据文件自动拆分
pnpm playwright test --workers=4

# 在 CI 上按文件名哈希切片（GitHub Actions 示例变量）
pnpm playwright test --shard=${{ matrix.shard }}/${{ strategy.total }}
```

**Cypress**
- CI 横向并行：把 spec 文件按 glob 分配到不同 Job。  
- 使用 **Dashboard 并行**（商业）可自动负载均衡。  

**缓存**：缓存浏览器二进制、依赖、以及 **trace/video** 产物（仅失败保留）。

---

## 8) 可视化证据：Trace / Screenshot / Video 🎥

**Playwright**
- `trace: 'on-first-retry'`：失败才抓，包含 **DOM 快照 + 网络 + 控制台 + 溯源**。  
- 本地查看：`npx playwright show-trace trace.zip`。  
- `expect(locator).toHaveScreenshot()` 可做轻量“视觉回归”（记得稳定化：禁动效、固定字体）。

**Cypress**
- 自带截图/视频；视觉回归可用 `cypress-image-snapshot`（注意像素差阈值与稳定化）。

---

## 9) 网络策略：何时 Mock，何时真跑 🌐

- **你方后端**：优先**真连测试环境**（发现契约/部署/跨域/缓存等真实问题）。  
- **三方服务**：使用 **沙盒环境/测试账号**；必要时本地 `route/intercept`。  
- **不可控不稳定因素**（广告、实验流量）→ 本地拦截或在测试环境关闭；**在代码中保留“测试开关”**。

---

## 10) 登录策略对比 🔐

- **Storage State（Playwright）**：一次登录 → 保存 `storageState.json` → 多用例复用，**最快**。  
- **`cy.session`（Cypress）**：缓存登录流程或直接写入 Cookie/LocalStorage。  
- **后端“测试登录”接口**：仅测试环境开放，返回已签发的会话/Token。  
- 避免频繁走 UI 登录，除非**专测登录流程**。

---

## 11) 组件测试 vs E2E 🧩

- **组件测试**：更小粒度、速度快、定位清晰；Playwright/ Cypress 都支持。  
- **E2E**：**跨页路径**与**系统集成**；两者**互补**。  
- 指南：复杂交互（日期选择、富文本）用 **组件测试**练兵，流程串路用 **E2E**兜底。

---

## 12) 稳定性：拒绝“睡觉流” 😴

- **不要 `wait(1000)`**；改用**状态断言**：  
  - Playwright：`await expect(locator).toBeVisible()` / `toHaveURL` / `toHaveText`。  
  - Cypress：断言自动重试：`cy.contains('已保存').should('be.visible')`。  
- **网络就绪**：等待 **请求完成** 或 **UI 可交互**，不要盯 `DOMContentLoaded`。  
- **禁动效**：测试 CSS 加 `prefers-reduced-motion: reduce`；或开关全局动画时间为 0。  
- **隔离**：每个用例独立 `page/context`/`cy.visit`；避免复用导致串味。  

---

## 13) CI 集成（GitHub Actions 片段）🤖

**Playwright**
```yaml
name: e2e
on: [push, pull_request]
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
      - run: pnpm dlx playwright install --with-deps
      - run: pnpm run build && pnpm run preview & npx wait-on http://localhost:4173
      - run: pnpm playwright test --reporter=list,html
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-trace
          path: |
            playwright-report/
            **/*.zip
```

**Cypress**
```yaml
name: e2e
on: [push, pull_request]
jobs:
  cy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm i --frozen-lockfile
      - run: pnpm run build && pnpm run preview & npx wait-on http://localhost:4173
      - run: pnpm dlx cypress run --browser chrome --config-file e2e/cypress.config.ts
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: cypress-artifacts
          path: e2e/artifacts/**
```

---

## 14) 性能与成本 💸
- **并行优先**：文件级并行；禁用无用浏览器；移动端仅抽样。  
- **选择性 Trace/视频**：失败才留证；减少 IO。  
- **按变更触发**：只有 `apps/web/**` 改动才跑 E2E（结合 Turbo/Nx 的 affected）。  
- **预览环境**：对每个 PR 部署 Preview，再跑 E2E，**实现环境即测试**。

---

## 15) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| `sleep()` 到处都是 | 慢且抖 | 用**可见性/网络完成**断言；Playwright `expect`/Cypress 重试 |
| 事事 Mock | “E2E”变“UI 测试” | 自家后端尽量真连；三方用沙盒/拦截 |
| CSS 选择器定位 | 轻微 UI 改动即全红 | 用 `getByRole`/`getByLabelText`/`data-testid` |
| 大而杂用例 | 定位困难 | 拆成清晰步骤；Page Object 暴露意图 |
| 每次都走 UI 登录 | 慢 | `storageState`/`cy.session`/测试登录端点 |
| 无证据 | CI 失败难复现 | 开 trace/视频/网络捕获（失败时保留） |

---

## 16) 提交前检查清单 ✅
- [ ] 覆盖“黄金路径”（登录、下单/发布、支付/提交表单、搜索/筛选）。  
- [ ] 定位器可访问性优先；无脆弱 CSS 选择器。  
- [ ] 无 `sleep`；全部使用状态/网络断言。  
- [ ] 登录使用缓存/快照；跨用例不共享易变状态。  
- [ ] 失败保留 trace/视频/截图；产物归档到 CI。  
- [ ] E2E 跑在 **预览环境** 或独立测试服务上；数据幂等可清理。

---

## 17) 练习 🏋️
1. 用 Playwright 为“搜索 → 结果 → 加入购物车 → 结算”写一条端到端用例，开启 `on-first-retry` trace 并在 CI 归档产物。  
2. 在 Cypress 中为“评论组件”做**组件测试**（输入、提交、错误提示），并用 `cy.intercept` 模拟失败重试。  
3. 为登录场景做“**两种策略**”对照：UI 登录与 `storageState`/`cy.session` 加速，对比时长与稳定性。  

---

**小结**：E2E 的价值在“**用户价值能否被兑现**”。Playwright/Cypress 都能胜任——选一个、立规范、建数据、控波动、留证据，你的回归就会**快而稳**，发布不再心跳加速。🚀
