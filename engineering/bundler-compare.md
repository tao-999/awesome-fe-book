# 8.1 Vite / Rollup / Webpack / esbuild / SWC 一览 ⚙️

本章是“大前端构建工具矩阵”的地图与指南针：谁负责**开发体验**（HMR、冷/热启动）、谁擅长**生产打包**（Tree-shaking、分包、产物体积）、谁又是**极速转译器**（JS/TS/JSX 的编译器/压缩器）。最后给出**选型决策树**、**性能小抄**与**可拷贝配置**。

---

## 0) 速通心智图

- **Vite**：开发期用 **原生 ESM + dev server**，生产期用 **Rollup** 打包；HMR 爽、生态全。  
- **Rollup**：**库/组件库首选** 的打包器（ESM 友好、产物干净、外部化友好）。  
- **Webpack**：**全能工业机**；复杂遗留、强自定义、**Module Federation** 场景仍有地位。  
- **esbuild**：Go 写的**超快打包/转译器**；也能独立打包，常做 Vite/Webpack 的**底层加速器**。  
- **SWC**：Rust 写的**转译/压缩器**；在 Next.js/Rspack 等里做 **Babel/Terser 的现代替代**。

> 一句话：**Vite 负责“爽”，Rollup 负责“净”，Webpack 负责“全”，esbuild/SWC 负责“快”。**

---

## 1) 核心维度对比（矩阵）

| 维度 | Vite | Rollup | Webpack | esbuild | SWC |
|---|---|---|---|---|---|
| 定位 | Dev Server +（产出用）Rollup | 打包器（库友好） | 通用打包器（插件最全） | 超快打包/转译 | 转译/压缩（编译器） |
| 冷启动/HMR | 🚀 极快 | — | 较慢（看项目体量） | 🚀🚀 | — |
| 产出体积 | 优（靠 Rollup） | 优 | 可优 | 良（功能较少） | 依赖上层 |
| 代码分割 | ✅ | ✅ | ✅ | ✅（有限制） | — |
| Tree-shaking | ✅ | ✅ | ✅ | ✅ | — |
| TS/JSX | ✅（esbuild 转译） | 需插件 | ✅ | ✅ | ✅ |
| CSS/预处理 | ✅（PostCSS/Sass 等） | 需插件 | ✅ | 有限 | — |
| SSR/框架集成 | ✅（SvelteKit/SSR 模式等） | — | ✅（Next/自建） | 限 | — |
| 多包/Monorepo | ✅（很适合） | ✅（库） | ✅ | ✅ | ✅ |
| 代表生态 | React/Vue/Svelte/TS | 库、工具包 | MF、复杂产线 | CLI/工具链 | Next.js/Rspack 等 |

> 注：SWC 不单独打包产物，它是“编译引擎”。真正打包由上层（Vite/Rollup/Webpack/Rspack 等）决定。

---

## 2) Vite：开发期“原生 ESM”，生产期交给 Rollup

### 2.1 安装与脚本
```bash
pnpm create vite@latest myapp --template react-ts
cd myapp
pnpm i
pnpm dev
pnpm build   # 走 Rollup
pnpm preview
```

### 2.2 基础配置（`vite.config.ts`）
```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  server: { port: 5173 },
  build: {
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom']
        }
      }
    }
  }
});
```

### 2.3 库模式（产出 ESM+CJS）
```ts
// vite.config.ts（库模式）
import { defineConfig } from 'vite';
import dts from 'vite-plugin-dts';

export default defineConfig({
  build: {
    lib: {
      entry: 'src/index.ts',
      name: 'AwesomeLib',
      formats: ['es', 'cjs'],
      fileName: (fmt) => `index.${fmt}.js`
    },
    rollupOptions: {
      external: ['react'], // peer 依赖外部化
    }
  },
  plugins: [dts()]
});
```

> **何时选 Vite**：应用/站点开发、需要顶级开发体验（HMR 秒回）、与现代框架配套（Vue/Svelte/React）。

---

## 3) Rollup：库作者的挚友

### 3.1 最小库配置（`rollup.config.mjs`）
```js
import dts from 'rollup-plugin-dts';
import typescript from '@rollup/plugin-typescript';
import { terser } from 'rollup-plugin-terser';

export default [
  // 代码产物
  {
    input: 'src/index.ts',
    external: ['react'],
    output: [
      { file: 'dist/index.esm.js', format: 'esm', sourcemap: true },
      { file: 'dist/index.cjs', format: 'cjs', sourcemap: true, exports: 'named' }
    ],
    plugins: [
      typescript({ tsconfig: './tsconfig.json' }),
      terser()
    ]
  },
  // 类型声明
  {
    input: 'src/index.ts',
    output: { file: 'dist/index.d.ts', format: 'es' },
    plugins: [dts()]
  }
];
```

### 3.2 要点
- **external/peerDepsExternal**：外部化 React/Vue 等，保持库体积干净。  
- **preserveModules**：需要按模块产出时开启（更利于 tree-shaking）。  
- **多入口**：使用 `input: { core:'...', utils:'...' }` 或插件管理入口。

> **何时选 Rollup**：**组件库/工具库**、需严控产物结构与外部化策略的场景。

---

## 4) Webpack：全能工业机 & Module Federation

### 4.1 最小应用配置（`webpack.config.js`）
```js
const path = require('path');

module.exports = {
  mode: process.env.NODE_ENV ?? 'development',
  entry: './src/main.tsx',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'app.[contenthash].js',
    clean: true,
    publicPath: '/'
  },
  devtool: 'source-map',
  devServer: {
    port: 8080,
    hot: true,
    historyApiFallback: true
  },
  resolve: { extensions: ['.ts', '.tsx', '.js'] },
  module: {
    rules: [
      { test: /\.[tj]sx?$/, loader: 'babel-loader', exclude: /node_modules/ },
      { test: /\.css$/, use: ['style-loader', 'css-loader', 'postcss-loader'] },
      { test: /\.(png|svg|jpg|gif)$/, type: 'asset' }
    ]
  }
};
```

### 4.2 何时仍选 Webpack
- 需要 **Module Federation**（微前端/跨仓共享运行时代码）。  
- 复杂遗留工程、插件/Loader 生态最全、与某些企业脚手架强绑定。  
- 对打包细节、产物兼容性、老旧浏览器支持有**硬性要求**。

> 现代项目若无特殊需求，**优先 Vite**；Webpack 适合“你知道你要它”。

---

## 5) esbuild：极致速度的打包/转译器

### 5.1 CLI 速用
```bash
# 单页应用（快速出活）
esbuild src/main.tsx --bundle --outdir=dist --sourcemap --minify --format=esm

# 多入口/代码分割
esbuild src/app.ts src/admin.ts --bundle --outdir=dist --splitting --format=esm

# 开发服务器（简单）
esbuild src/main.tsx --bundle --servedir=dist --watch
```

### 5.2 要点
- **超快**（Go 实现）；支持 TS/JSX、分割、别名、Define 注入。  
- CSS 支持有限（可配合 PostCSS/UnoCSS 等或交给上层）。  
- 常做 **Vite/Bundler 的底层**（转译、压缩插件）。

---

## 6) SWC：Rust 转译/压缩引擎（Babel/Terser 替代）

### 6.1 `.swcrc` 基本配置
```json
{
  "jsc": {
    "parser": { "syntax": "typescript", "tsx": true, "decorators": false },
    "target": "es2020",
    "transform": { "react": { "runtime": "automatic", "useBuiltins": true } },
    "loose": false,
    "externalHelpers": true
  },
  "module": { "type": "es6" },
  "minify": true,
  "sourceMaps": true
}
```

### 6.2 在 Node/Jest 中使用
```bash
# 运行时注册（替代 ts-node/babel-register）
node -r @swc/register src/server.ts

# Jest 转换器
pnpm add -D @swc/jest
```

> SWC 不是打包器；通常在 **Next.js / Rspack / Vite 插件** 中作为编译与压缩后端。

---

## 7) 选型决策树（实用）

1. **做应用（React/Vue/Svelte）** → 默认 **Vite**。  
2. **做库/组件库** → **Rollup（或 Vite 库模式）**；external 友好、产物干净。  
3. **要 Module Federation/复杂企业链路** → **Webpack**（或 Rspack 生态）。  
4. **只要最快的构建/工具脚手架** → 直接上 **esbuild**（或把它作为 Vite/Webpack 的加速器）。  
5. **想替换 Babel/Terser** → **SWC**（在你的 bundler 里挂接）。

---

## 8) 性能小抄（通用）

- **拆分依赖**：大依赖独立成 `vendor` chunk；路由级 `dynamic import()`。  
- **图片/字体独立管**：用专门流水线（Image/CDN/字体子集化），不要让 JS 打包管一切。  
- **SourceMap 策略**：开发 `cheap-module-source-map/inline`，生产只为错误上报生成 `hidden` map。  
- **缓存**：CI 用 **pnpm + 缓存**，构建器开启 **持久缓存**（Webpack `cache: filesystem`）。  
- **TS 加速**：类型检查与打包**解耦**（`tsc --noEmit`/`vue-tsc` 单独跑），转译交给 esbuild/SWC。  
- **Monorepo**：用 **Turborepo/Nx** 做任务缓存；库产物预构建，应用按需引用。  
- **CSS**：仅在必要处启用 `@apply`/嵌套；Tailwind 建议 **JIT** 与 `content` 精确匹配以减少产物。

---

## 9) 反模式与纠偏

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 把服务器渲染产物也交给单一打包器做所有事 | 构建慢、耦合重 | 前后端产线分离；SSR 层独立 |
| 盲目开启所有插件 | 冷启动慢、HMR 卡 | 按需启用；仅保留“真的用到”的插件 |
| 重复打包同一依赖 | 产物膨胀 | `externals`/`optimizeDeps`/`manualChunks` 规划 |
| 生产还保留庞大 source map | 体积大、泄露源码 | 用 hidden/上传到错误上报平台，产物不包含 |
| 把类型检查放在打包链路 | dev 卡顿 | 类型检查独立并行（tsc/vue-tsc/eslint） |

---

## 10) 可拷贝模板 🧰

### 10.1 Vite（React）增强模板
```ts
// vite.config.ts
import { defineConfig, splitVendorChunkPlugin } from 'vite';
import react from '@vitejs/plugin-react';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [
    react({ jsxRuntime: 'automatic' }),
    tsconfigPaths(),
    splitVendorChunkPlugin()
  ],
  build: {
    target: 'es2020',
    sourcemap: process.env.SOURCEMAP === '1',
    rollupOptions: {
      output: {
        chunkFileNames: 'chunks/[name]-[hash].js',
        assetFileNames: 'assets/[name]-[hash][extname]'
      }
    }
  },
  server: { port: 5173, open: true }
});
```

### 10.2 Rollup（库）外部化 + 类型
```js
// rollup.config.mjs
import dts from 'rollup-plugin-dts';
import esbuild from 'rollup-plugin-esbuild';
import { dependencies, peerDependencies } from './package.json' assert { type: 'json' };
const externals = [...Object.keys(dependencies ?? {}), ...Object.keys(peerDependencies ?? {})];

export default [
  {
    input: 'src/index.ts',
    external: externals,
    plugins: [esbuild({ target: 'es2020' })],
    output: [
      { file: 'dist/index.esm.js', format: 'esm' },
      { file: 'dist/index.cjs', format: 'cjs', exports: 'named' }
    ]
  },
  { input: 'src/index.ts', output: { file: 'dist/index.d.ts', format: 'es' }, plugins: [dts()] }
];
```

### 10.3 Webpack（MF 片段）
```js
// webpack.modulefederation.js
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'app1',
      filename: 'remoteEntry.js',
      exposes: { './Button': './src/Button' },
      remotes: { app2: 'app2@https://host/remoteEntry.js' },
      shared: { react: { singleton: true }, 'react-dom': { singleton: true } }
    })
  ]
};
```

### 10.4 esbuild（工具脚手架）
```json
// package.json scripts
{
  "scripts": {
    "dev": "esbuild src/cli.ts --bundle --platform=node --outfile=dist/cli.cjs --watch",
    "build": "esbuild src/cli.ts --bundle --platform=node --outfile=dist/cli.cjs --minify --sourcemap"
  }
}
```

### 10.5 SWC（Node 服务转译）
```js
// node -r @swc/register src/server.ts
// .swcrc 同上
```

---

## 11) 练习 🏋️

1. 把现有 React 应用从 Webpack 迁到 **Vite**：保留别名、PostCSS、环境变量；用 `manualChunks` 分包。  
2. 为组件库写 **Rollup** 配置：外部化 peer deps，启用 `rollup-plugin-dts`，确保 ESM+CJS 输出。  
3. 写一个 **esbuild** 脚手架：接受 `--watch`、`--minify` 开关，支持多入口与 `--splitting`。  
4. 用 **SWC** 替换 Babel 在 Jest 中的转译，跑一次基准对比构建与测试用时。

---

## 12) TL;DR

- **应用**：Vite；**库**：Rollup；**工业/联邦**：Webpack；**极致速度/底层加速**：esbuild/SWC。  
- 让 **打包器** 做它擅长的事，把**类型检查/图片/字体/SSR** 各自分流；少即是多，快才是王道。🏎️
