# 10.1 ESLint / Prettier / 规范化 🧹✨

本章目标：给你的仓库装上**统一的编码规范**与**自动格式化**，开发机/CI/IDE 三端同源；并提供 **Flat Config（ESLint 9+）** 的现代配置、Prettier 分工、提交前钩子、CI 流水线与 Monorepo 实践。

---

## 0) 基本法（别打架定律）

- **职责分离**：  
  - **Prettier** 只管“怎么排版”（空格、分号、引号、换行…）。  
  - **ESLint** 只管“对不对”（bug 风险、API 误用、可维护性）。  
  - 不要在 ESLint 里跑 Prettier 规则，**分开执行**，互不干扰。
- **现代形态**：使用 **Flat Config**（`eslint.config.mjs`），类型感知用 `typescript-eslint` 的 **Type-Checked** 配置。
- **一键落地**：**保存即格式化**、**提交即检查**、**CI 必过**。

---

## 1) 快速上手（TS/React 应用示例）

### 1.1 依赖安装
```bash
pnpm add -D eslint @eslint/js typescript typescript-eslint \
           prettier eslint-config-prettier \
           eslint-plugin-react eslint-plugin-react-hooks eslint-plugin-jsx-a11y \
           eslint-plugin-unused-imports
```

> 说明：  
> - `@eslint/js` 提供官方 JS 规则。  
> - `typescript-eslint` = 解析器 + 规则 + 平台（Flat Config）。  
> - `eslint-config-prettier` 用于**关闭**与 Prettier 冲突的 ESLint 规则（不是把 Prettier 跑进 ESLint）。  
> - React 相关插件按需加；非 React 项目可略过 `react*` 与 `jsx-a11y`。

### 1.2 根配置：`eslint.config.mjs`（Flat Config）
```js
// eslint.config.mjs
import globals from "globals";
import js from "@eslint/js";
import tseslint from "typescript-eslint";
import reactPlugin from "eslint-plugin-react";
import reactHooks from "eslint-plugin-react-hooks";
import jsxA11y from "eslint-plugin-jsx-a11y";
import unusedImports from "eslint-plugin-unused-imports";
import prettierCompat from "eslint-config-prettier"; // 关闭与 Prettier 冲突的规则

/** @type {import('eslint').Linter.FlatConfig[]} */
export default [
  // 全局忽略（替代 .eslintignore）
  {
    ignores: [
      "**/dist/**",
      "**/build/**",
      "**/.turbo/**",
      "**/.next/**",
      "**/coverage/**",
      "**/*.min.js",
      "packages/**/dist/**"
    ]
  },

  // JS 基础（ES2023 + 浏览器/Node 全局）
  {
    languageOptions: {
      ecmaVersion: "latest",
      sourceType: "module",
      globals: {
        ...globals.browser,
        ...globals.node
      }
    }
  },
  js.configs.recommended,

  // TS（类型感知规则）
  ...tseslint.configs.recommendedTypeChecked.map((c) => ({
    ...c,
    files: ["**/*.ts", "**/*.tsx"],
    languageOptions: {
      ...c.languageOptions,
      parserOptions: {
        ...c.languageOptions?.parserOptions,
        // 使用 Project Service，无需把每个包都手工列进 project（TS 5.0+）
        projectService: true,
        tsconfigRootDir: new URL(".", import.meta.url).pathname
      }
    }
  })),

  // React + 可访问性
  {
    files: ["**/*.tsx", "**/*.jsx"],
    plugins: {
      react: reactPlugin,
      "react-hooks": reactHooks,
      "jsx-a11y": jsxA11y,
      "unused-imports": unusedImports
    },
    settings: {
      react: { version: "detect" }
    },
    rules: {
      // React 与 Hooks
      ...reactPlugin.configs.recommended.rules,
      ...reactPlugin.configs["jsx-runtime"].rules,
      ...reactHooks.configs.recommended.rules,

      // a11y 推荐
      ...jsxA11y.configs.recommended.rules,

      // 未使用导入/变量（先删再 lint）
      "no-unused-vars": "off",
      "@typescript-eslint/no-unused-vars": "off",
      "unused-imports/no-unused-imports": "warn",
      "unused-imports/no-unused-vars": [
        "warn",
        { vars: "all", varsIgnorePattern: "^_", args: "after-used", argsIgnorePattern: "^_" }
      ]
    }
  },

  // 通用代码质量补丁
  {
    rules: {
      "eqeqeq": ["error", "smart"],
      "no-console": ["warn", { allow: ["warn", "error"] }],
      "no-constant-condition": ["error", { checkLoops: false }],
      "no-restricted-syntax": [
        "warn",
        { selector: "TSEnumDeclaration", message: "建议改用 as const 对象代替 enum（tree-shaking 更友好）。" }
      ],
      // TS 强化
      "@typescript-eslint/consistent-type-imports": ["warn", { prefer: "type-imports", fixStyle: "inline-type-imports" }],
      "@typescript-eslint/no-floating-promises": "error",
      "@typescript-eslint/no-misused-promises": ["error", { checksVoidReturn: { attributes: false } }],
      "@typescript-eslint/consistent-type-definitions": ["warn", "type"]
    }
  },

  // 关闭样式冲突（交给 Prettier）
  prettierCompat
];
```

> 如果你不是 React 项目，删掉 React/JSX 相关块即可；Vue/Svelte 在 §4 给出配方。

### 1.3 Prettier：`.prettierrc.json`
```json
{
  "printWidth": 100,
  "singleQuote": true,
  "trailingComma": "all",
  "semi": true,
  "tabWidth": 2,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

`.prettierignore`
```
dist
build
coverage
*.min.js
```

### 1.4 编辑器：`.editorconfig`
```
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true

[*.md]
trim_trailing_whitespace = false
```

### 1.5 脚本与钩子（提交即规范）
`package.json`
```json
{
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "fmt": "prettier . --write",
    "fmt:check": "prettier . --check"
  },
  "devDependencies": {
    "husky": "^9.0.0",
    "lint-staged": "^15.0.0"
  }
}
```

初始化钩子：
```bash
pnpm dlx husky init
# 添加 pre-commit 阶段
printf '%s\n' 'pnpm lint-staged' > .husky/pre-commit
```

`package.json` 中加入：
```json
{
  "lint-staged": {
    "*.{ts,tsx,js,jsx,json}": ["prettier --write", "eslint --fix"],
    "*.{css,scss,md,yml,yaml}": ["prettier --write"]
  }
}
```

---

## 2) CI 流水线（GitHub Actions 片段）

`.github/workflows/lint.yml`
```yaml
name: Lint
on: [push, pull_request]
jobs:
  lint:
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
      - run: pnpm fmt:check
      - run: pnpm lint -- --max-warnings=0
```

> Monorepo（Turborepo/Nx）建议把 `lint`/`fmt:check` 作为 **独立管道**，并结合 `--filter`/`affected` 只跑受影响的包。

---

## 3) 性能与稳定性小抄 🏎️

- **类型感知**：`recommendedTypeChecked` 自带强规则；若初始成本太高，可先用 `...tseslint.configs.recommended`，逐步升级。  
- **缓存**：`eslint --cache` 在传统 CLI 生效；多数 CI 通过包管理器缓存 + 任务缓存（Turbo/Nx）即可。  
- **规则强度策略**：把“团队底线”定为 `error`，争议项先 `warn`；季度回顾统一提升。  
- **按域分层**：UI 包、Node 服务、脚手架各自有覆写（files: globs），避免一刀切。

---

## 4) 其他技术栈配方 🍱

### 4.1 Vue 3（`eslint-plugin-vue`）
```bash
pnpm add -D eslint-plugin-vue
```
`eslint.config.mjs` 中追加：
```js
import vue from "eslint-plugin-vue";

export default [
  // ...前面的配置
  {
    files: ["**/*.vue"],
    languageOptions: { parserOptions: { ecmaVersion: "latest", sourceType: "module" } },
    ...vue.configs["flat/recommended"],
    rules: {
      // 根据团队口味微调
      "vue/html-self-closing": ["warn", { html: { void: "always", normal: "never", component: "always" } }]
    }
  },
  // 让 TS/JS 规则也覆盖 Vue SFC 的 <script>
];
```

### 4.2 Svelte（`eslint-plugin-svelte`）
```bash
pnpm add -D eslint-plugin-svelte svelte-eslint-parser
```
```js
import svelte from "eslint-plugin-svelte";

export default [
  // ...
  {
    files: ["**/*.svelte"],
    ...svelte.configs["flat/recommended"],
    rules: { "svelte/no-at-html-tags": "warn" }
  }
];
```

> 提示：不同插件的 **Flat Config 名称** 可能略有差异（`flat/recommended` / `recommended`），以插件文档为准。

---

## 5) VS Code 建议设置（保存即整洁）🧰

`.vscode/settings.json`
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"  // 需要时手动触发或设为 "always"
  },
  "eslint.experimental.useFlatConfig": true,
  "eslint.validate": ["javascript", "javascriptreact", "typescript", "typescriptreact", "vue", "svelte", "json"],
  "prettier.printWidth": 100
}
```

> 避免“重复修复”抖动：**Prettier 负责格式化**，ESLint 只做代码修复（imports、规则）。如果保存时太慢，把 `fixAll.eslint` 改为手动命令。

---

## 6) Monorepo 模式（根 + 子包）

- **根**放公共规则与忽略；子包按域覆写：  
  - `packages/ui/eslint.config.mjs`：UI 特有规则（如 `react/jsx-no-literals`）。  
  - `apps/api/eslint.config.mjs`：Node/服务端规则（如 `no-console` 放宽、增加安全插件）。  
- Turborepo：`turbo.json` 中 `lint` 任务 `outputs: []`，配 `--filter=...[origin/main]` 只跑受影响包。  
- Nx：使用 `targetDefaults.lint` 与 `nx affected -t lint`。

---

## 7) 扩展：文档与样式也要“干净”

**Markdown & YAML**
```bash
pnpm add -D markdownlint-cli yaml-lint
```
`package.json`
```json
{
  "scripts": {
    "lint:md": "markdownlint '**/*.md' -i node_modules -i dist",
    "lint:yaml": "yamllint ."
  }
}
```

**CSS / Tailwind**
```bash
pnpm add -D stylelint stylelint-config-standard stylelint-config-prettier
```
`stylelint.config.cjs`
```js
module.exports = {
  extends: ["stylelint-config-standard", "stylelint-config-prettier"]
};
```

---

## 8) 常见坑与修复 🧯

| 症状 | 可能原因 | 处理 |
|---|---|---|
| ESLint 巨慢/报 “Parsing error: Cannot read file tsconfig” | 类型感知找不到项目 | 在 Flat Config 的 TS 段设置 `projectService: true` 或为子包单独放 `tsconfig.json` |
| Prettier 与 ESLint 抢格式 | 把 Prettier 装进 ESLint | **不要**用 `eslint-plugin-prettier`；使用 `eslint-config-prettier` 关闭冲突规则，Prettier 单独跑 |
| React 17+ 无 `import React` 报错 | 未启用 JSX Runtime 配置 | 使用 `reactPlugin.configs["jsx-runtime"]` 或在规则里关掉 `react/react-in-jsx-scope` |
| 未使用导入老是警告 | TS/ESLint 重叠 | 关闭 `@typescript-eslint/no-unused-vars`，改用 `eslint-plugin-unused-imports` 一把梭 |
| Monorepo 子包 TS 解析错乱 | 根/子包 tsconfig 冲突 | 子包各自 `tsconfig.json`；根 TS 只做基础共享，不要跨包 `references` 失配 |

---

## 9) 提交前检查清单 ✅

- [ ] 本地 `pnpm fmt:check && pnpm lint -- --max-warnings=0` 全绿。  
- [ ] VS Code 保存即格式化，ESLint 仅修正逻辑问题。  
- [ ] CI 有 **Prettier 检查** 与 **ESLint 检查** 两步。  
- [ ] Monorepo：子包有针对性覆写；根忽略正确。  
- [ ] 新文件类型（Vue/Svelte/Node 脚本）已有对应插件与规则覆盖。  

---

## 10) 备选方案（了解即可）

- **Biome**：一体化 Lint + Format + Import 排序（Rust 实现，超快）；迁移需评估规则覆盖与团队一致性。  
- **Rome（旧）**：已停更，转向 Biome，不再建议新项目使用。

---

**小结**：让 **Prettier 负责“颜值”**，**ESLint 负责“智商”**。Flat Config + 类型感知 + 提交钩子 + CI 双保险，把“规范”变成**自动化的默认值**，你的代码库会越来越丝滑。🧴🚀
