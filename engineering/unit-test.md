# 11.1 单测（Vitest / Jest）🧪

本章目标：把**单元测试**从“仪式感”变成“生产力”。围绕 **Vitest**（Vite 家族，快、原生 ESM）与 **Jest**（生态庞大、在老项目/React Native 仍常见）给出：**选型心法、配置模板、测试写法、Mock 策略、覆盖率、CI 集成、反模式**。

---

## 0) 先立规矩：测试金字塔里的“单测”🧱
- **单测（unit）**：函数/组件的最小可测试单元，**快**、**确定性**、**无网络/无外部 IO**。
- **集成（integration）**：模块协作、数据库/网络**用仿真层**（如 MSW/测试库 in-memory DB）。
- **端到端（e2e）**：黑盒点击。**别拿单测做 e2e**。

> 心法：**快到随手就跑**，**小到定位就明**，**稳到 CI 不抖**。

---

## 1) 选型：Vitest vs Jest 🎯
- **Vitest**：原生 ESM、与 Vite 共用插件解析，**启动快、热重载爽**；前端/Node 工具都适合。
- **Jest**：生态老牌，插件多；在 **React Native、旧仓**、或深度依赖 Jest API 的项目里仍适用。

简表：
- **速度**：Vitest 🏎️ > Jest（需 Babel/ts-jest/transform）。
- **ESM/TS**：Vitest 原生；Jest 需配置转换。
- **同构/Vite 项目**：Vitest 更顺手。
- **RN/老工程**：多半继续 Jest 更省心。

---

## 2) Vitest 最小可用配置 ⚡

### 2.1 安装
```bash
pnpm add -D vitest @vitest/coverage-v8
# 浏览器模拟环境（前端）
pnpm add -D jsdom
```

### 2.2 `vitest.config.ts`
```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'jsdom',          // 纯 Node 库换成 'node'
    globals: true,                  // it/describe/expect 无需显式导入
    include: ['**/*.{test,spec}.{ts,tsx,js}'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'lcov'],
      all: true,
      thresholds: { lines: 80, functions: 80, branches: 70, statements: 80 }
    },
    setupFiles: ['./test/setup.ts'] // 全局钩子/扩展
  }
});
```

### 2.3 例子：纯函数与 React 组件
```ts
// src/sum.ts
export const sum = (a: number, b: number) => a + b;
```

```ts
// src/sum.test.ts
import { sum } from './sum';

describe('sum', () => {
  it('adds numbers', () => {
    expect(sum(1, 2)).toBe(3);
  });

  it.each([
    [0, 0, 0],
    [-1, 1, 0],
    [1.2, 1.3, 2.5],
  ])('sum(%p,%p) = %p', (a, b, r) => {
    expect(sum(a as number, b as number)).toBeCloseTo(r as number);
  });
});
```

```tsx
// src/Hello.tsx
export function Hello({ name }: { name: string }) {
  return <h1>Hello, {name}</h1>;
}
```

```tsx
// src/Hello.test.tsx
import { render, screen } from '@testing-library/react';
import '@testing-library/jest-dom';
import { Hello } from './Hello';

it('renders greeting', () => {
  render(<Hello name="Ada" />);
  expect(screen.getByRole('heading', { name: /ada/i })).toBeInTheDocument();
});
```

`test/setup.ts`（全局扩展）：
```ts
import '@testing-library/jest-dom';
```

---

## 3) Jest 最小可用配置（旧仓/React Native）🧰

### 3.1 安装
```bash
pnpm add -D jest ts-jest @types/jest
# 浏览器环境
pnpm add -D jest-environment-jsdom @testing-library/jest-dom
```

### 3.2 `jest.config.ts`
```ts
import type { Config } from 'jest';

const config: Config = {
  testEnvironment: 'jsdom',
  transform: { '^.+\\.(t|j)sx?$': ['ts-jest', { tsconfig: 'tsconfig.json' }] },
  setupFilesAfterEnv: ['<rootDir>/test/setup.ts'],
  collectCoverageFrom: ['src/**/*.{ts,tsx,js,jsx}'],
  coverageReporters: ['text', 'html', 'lcov'],
  coverageThreshold: { global: { lines: 80, functions: 80, branches: 70, statements: 80 } }
};
export default config;
```

---

## 4) Mock 策略：**只 Mock 边界** 🧩

### 4.1 模块 Mock（Vitest）
```ts
// api.ts
export async function getUser(id: string) {
  const r = await fetch(`/api/users/${id}`);
  return r.json();
}
```

```ts
// api.test.ts
import { vi } from 'vitest';
import * as api from './api';

it('getUser parses response', async () => {
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
    json: async () => ({ id: 'u1', name: 'Ada' })
  }));
  expect(await api.getUser('u1')).toEqual({ id: 'u1', name: 'Ada' });
});
```

### 4.2 网络 Mock：**MSW**（推荐）
```bash
pnpm add -D msw
```
```ts
// test/setup.ts
import { afterAll, afterEach, beforeAll } from 'vitest';
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

const server = setupServer(
  http.get('/api/users/:id', ({ params }) => {
    return HttpResponse.json({ id: params.id, name: 'Ada' });
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```
> 优点：用**真实 fetch** + Service Worker/Node 拦截，贴近生产，又不触网。

### 4.3 计时器与日期
```ts
import { vi } from 'vitest';

vi.useFakeTimers();
const fn = vi.fn();
setTimeout(fn, 1000);
vi.advanceTimersByTime(1000);
expect(fn).toHaveBeenCalledTimes(1);

vi.setSystemTime(new Date('2025-01-01'));
```

---

## 5) 测试设计：让用例“说人话” 🧠
- **命名**：行为 + 条件 + 期望（中文也行，别省略主语）。
- **Arrange-Act-Assert（AAA）**：先准备，再动作，再断言。
- **反例覆盖**：空值/边界/异常路径；**每个 bug 都配一个回归用例**。
- **数据构造器**：避免“复制粘贴 JSON 地狱”，用 builder/factory 生成测试数据。
- **属性测试（Property-based）**：对纯函数引入 fast-check：
  ```bash
  pnpm add -D fast-check
  ```
  ```ts
  import { test, assert } from 'vitest';
  import fc from 'fast-check';
  test('sum is commutative', () => {
    fc.assert(fc.property(fc.integer(), fc.integer(), (a, b) => a + b === b + a));
  });
  ```

---

## 6) 前端组件测试招式（React/Vue）🎨

### 6.1 React Testing Library（RTL）
- **多用角色/文本查找**：`getByRole`, `getByLabelText`，模拟真实用户而非类名。
- **避免快照滥用**：UI 细节易变；**行为断言**更稳。
- **用户交互**：
  ```ts
  import user from '@testing-library/user-event';
  await user.click(screen.getByRole('button', { name: /save/i }));
  ```

### 6.2 Vue Test Utils
```bash
pnpm add -D @vue/test-utils @testing-library/vue
```
```ts
import { render, screen } from '@testing-library/vue';
import Counter from './Counter.vue';

it('increments', async () => {
  render(Counter);
  await screen.findByText('0');
  await screen.getByRole('button', { name: /plus/i }).click();
  expect(screen.getByText('1')).toBeInTheDocument();
});
```

---

## 7) 覆盖率与门禁 📈
- **阈值**：给全局与关键包设 **合理**门槛（如 L80 F80 B70），**不要以量代质**。
- **可视化**：`coverage/lcov-report/index.html` 看红绿，针对热点补测。
- **CI 必跑**：在 PR 阶段阻塞低于阈值的改动；与变更行/文件相关的**增量覆盖**更有意义。

---

## 8) 性能与稳定性小抄 🏎️
- **分层测试**：纯函数最多，组件行为适量，集成少量，e2e 更少。
- **隔离外部依赖**：网络/时钟/随机数都可控（MSW、fakeTimers、seed）。
- **并发与泄漏**：避免跨用例共享可变单例；必要时 `vi.restoreAllMocks()`。
- **Watch 模式**：Vitest `pnpm vitest -- --watch`，改动即跑，提升反馈速度。
- **Monorepo**：子包自带配置；顶层用 Turbo/Nx 只跑受影响的测试。

---

## 9) CI 集成（GitHub Actions 片段）🤖
```yaml
name: test
on: [push, pull_request]
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm i --frozen-lockfile
      - run: pnpm vitest run --coverage
```

Monorepo（Turborepo）：
```yaml
- run: pnpm turbo run test --filter=...[origin/main]
```

---

## 10) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 为覆盖率而覆盖率 | brittle 用例一堆、维护困难 | 针对**行为与风险点**写测试，设**合理阈值** |
| 滥用快照 | UI 微调全仓爆红 | 只对**结构稳定**对象/序列化结果做小快照 |
| 到处全局 Mock | 难以推理、串味 | Mock 在**用例就近**定义，`afterEach` 清理 |
| 单测里访问真网络/DB | 慢、脆、不可重复 | 用 **MSW / in-memory**，网络 IO 全拦截 |
| 将「业务逻辑」塞进组件 | 组件测试困难 | 逻辑抽出为纯函数 + hooks/composables 可单测 |
| 不测异常路径 | BUG 都在边角料 | 每个函数都要有**失败路径**用例与断言 |

---

## 11) 提交前检查清单 ✅
- [ ] 本地 `pnpm test` 通过、无资源泄漏日志。
- [ ] 覆盖率达阈值；新增逻辑有对应用例。
- [ ] 网络/时间/随机数均可控；无真实外部依赖。
- [ ] 组件测试以**可访问性选择器**定位，不绑样式细节。
- [ ] Mock/Spy 在 `afterEach` 统一还原（`vi.restoreAllMocks()` / `jest.restoreAllMocks()`）。

---

## 12) 练习 🏋️
1. 用 **Vitest + MSW** 为一个“搜索输入框”写：输入去抖 300ms、请求取消、空状态与错误提示的完整用例。  
2. 把一个“胖组件”的业务逻辑抽到纯函数/自定义 Hook，补上**属性测试**与**边界用例**。  
3. 在 CI 上开启 **增量覆盖**统计：只对本 PR 改动的行/文件设门槛，防止“历史债务”阻塞迭代。

---

**小结**：单测的价值在于**快速、稳定、可定位**。用 Vitest/Jest 建立“**快反馈** + **可控边界** + **行为断言**”的测试习惯，你的代码会变得更可重构、更敢改。🚀
