# 8.2 tsup / tsx / 速构脚手架 ⚡

本章目标：用 **tsx** 做“**超快运行 TS/ESM 的开发运行器**”，用 **tsup** 做“**零样板的打包器**”，再拼出几套**可复制的脚手架模版**（库/CLI/服务/Monorepo），让你从“写代码”到“发 npm/部署”一路顺滑。

---

## 0) 这俩是啥 & 何时用

- **tsx**：基于 esbuild 的 **TS/JS/ESM 运行器**。代替 `ts-node`/`node --loader`，支持 `--watch`、ESM、JSX、import 别名等，**开发期/脚本期**超爽。  
- **tsup**：基于 esbuild 的 **零配置打包器**，天然支持 TS/JSX、同时产出 **ESM+CJS**、`dts` 类型打包、代码分割、`minify/sourcemap` 等，**发布期**稳。

> 简单心法：**开发跑 tsx，发布打 tsup**。应用类（非发布 npm）也可直接用 Vite，但“库/CLI/脚本/微服务”场景，tsx+tsup 更干净。

---

## 1) 快速起步 🏁

```bash
# 安装
pnpm add -D tsup tsx typescript
pnpm tsc --init

# 运行 TS 文件（开发）
pnpm tsx src/dev.ts         # 单次
pnpm tsx watch src/server.ts  # 热重载

# 打包
pnpm tsup src/index.ts --format esm,cjs --dts
```

最小 `tsconfig.json`（Node 20+ 推荐）：
```json
{
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2020",
    "jsx": "react-jsx",
    "strict": true,
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] },
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "types": []
  },
  "include": ["src"]
}
```

---

## 2) 库脚手架模版（ESM + CJS + dts）📦

**目录**
```
my-lib/
  src/
    index.ts
  package.json
  tsconfig.json
  tsup.config.ts
  README.md
```

**src/index.ts**
```ts
export function sum(a: number, b: number) {
  return a + b;
}
```

**package.json**
```json
{
  "name": "@scope/my-lib",
  "version": "0.0.0",
  "description": "My tiny lib",
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
  "files": ["dist"],
  "sideEffects": false,
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsup",
    "typecheck": "tsc --noEmit",
    "prepublishOnly": "pnpm run typecheck && pnpm run build"
  },
  "peerDependencies": {},
  "devDependencies": { "tsup": "^8.0.0", "tsx": "^4.0.0", "typescript": "^5.0.0" }
}
```

**tsup.config.ts**
```ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['esm', 'cjs'],
  dts: true,              // 打包类型声明
  sourcemap: true,
  clean: true,            // 清理 dist
  treeshake: true,
  target: 'es2020',
  minify: true,
  splitting: false,       // 库默认不分包，保持产物简单
  external: []            // 也可用 tsup --external:react 等
});
```

> 提示：库场景把第三方依赖（尤其 `react`/`vue`）放 **peerDependencies** 并在 `external` 外部化，保持产物干净。

---

## 3) CLI 脚手架模版（可全局安装）🧰

**目录**
```
my-cli/
  src/cli.ts
  package.json
  tsconfig.json
  tsup.config.ts
```

**src/cli.ts**
```ts
#!/usr/bin/env node
import { readFileSync } from 'node:fs';
import { resolve } from 'node:path';

const pkg = JSON.parse(readFileSync(resolve(__dirname, '../package.json'), 'utf-8'));

const args = process.argv.slice(2);
if (args.includes('--version') || args.includes('-v')) {
  console.log(pkg.version);
  process.exit(0);
}

console.log('Hello from my-cli!');
```

**package.json**
```json
{
  "name": "my-cli",
  "version": "0.0.0",
  "license": "MIT",
  "type": "module",
  "bin": { "my-cli": "dist/cli.cjs" },
  "scripts": {
    "dev": "tsx watch src/cli.ts",
    "build": "tsup src/cli.ts --format cjs,esm --dts=false",
    "prepublishOnly": "pnpm build"
  },
  "dependencies": {},
  "devDependencies": { "tsup": "^8.0.0", "tsx": "^4.0.0", "typescript": "^5.0.0" }
}
```

**tsup.config.ts**
```ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/cli.ts'],
  format: ['cjs', 'esm'],
  target: 'node20',
  clean: true,
  sourcemap: true,
  minify: false,
  banner: { cjs: '#!/usr/bin/env node' } // 给 CJS 产物补 shebang
});
```

> 运行期建议：**开发**直接 `pnpm dev` 用 `tsx` 跑，**发布**后用户执行的是打包好的 `dist/cli.cjs`（冷启动更快、零依赖）。

---

## 4) 服务/Worker 脚手架（Node/Edge）🛰️

**目录**
```
my-service/
  src/server.ts
  package.json
  tsconfig.json
  tsup.config.ts
```

**src/server.ts**
```ts
import http from 'node:http';

const server = http.createServer((req, res) => {
  res.setHeader('content-type', 'application/json; charset=utf-8');
  res.end(JSON.stringify({ ok: true, path: req.url }));
});

const port = Number(process.env.PORT ?? 3000);
server.listen(port, () => console.log('listening on http://localhost:' + port));
```

**package.json**
```json
{
  "name": "my-service",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "tsx watch --env-file .env src/server.ts",
    "build": "tsup src/server.ts --format esm --minify --sourcemap",
    "start": "node dist/server.esm.js"
  },
  "devDependencies": { "tsup": "^8.0.0", "tsx": "^4.0.0", "typescript": "^5.0.0" }
}
```

**tsup.config.ts**
```ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/server.ts'],
  format: ['esm'],
  target: 'node20',
  sourcemap: true,
  clean: true,
  minify: true,
  env: { NODE_ENV: 'production' }
});
```

> Edge/Workers（如 Cloudflare）则把 `target` 改成 `es2022`，并避免 Node 内置模块。

---

## 5) Monorepo 速构（pnpm + Turborepo）🧩

**目录**
```
my-repo/
  package.json
  pnpm-workspace.yaml
  turbo.json
  packages/
    ui/        # 组件库（tsup）
    utils/     # 工具库（tsup）
  apps/
    web/       # 应用（可 Vite）
    api/       # 服务（tsx dev + tsup build）
```

**pnpm-workspace.yaml**
```yaml
packages:
  - "packages/*"
  - "apps/*"
```

**root package.json（关键脚本）**
```json
{
  "private": true,
  "packageManager": "pnpm@9",
  "scripts": {
    "build": "turbo build",
    "dev": "turbo dev",
    "lint": "turbo lint",
    "typecheck": "turbo typecheck"
  },
  "devDependencies": { "turbo": "^2.0.0" }
}
```

**turbo.json（任务缓存 + 依赖推导）**
```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "dependsOn": [] },
    "lint": { "outputs": [] },
    "typecheck": { "outputs": [] }
  }
}
```

> **库包** `packages/ui|utils` 用 tsup，**应用包** `apps/web` 用 Vite；包间通过 `exports` + `types` 输出，应用按包名直接引用。

---

## 6) tsx 的高频玩法 🧪

- **热重载开发**：`tsx watch src/server.ts`（改文件自动重启）。  
- **预加载/别名**：`tsx --watch -r dotenv/config src/foo.ts`；配合 `tsconfig-paths/register` 处理别名（或直接在 tsx 中 `--tsconfig-paths`）。  
- **测试脚本**：小工具无需上 Jest/Vitest，也可直接 `tsx` 跑断言脚本。  
- **ESM/CJS 混用**：优先全局 **ESM**；CJS 需要时由 tsup 产出 `cjs` 分发给老环境。

---

## 7) tsup 常用选项小抄 📝

```ts
defineConfig({
  entry: ['src/index.ts'],     // 多入口：['src/a.ts','src/b.ts'] 或对象映射
  format: ['esm', 'cjs', 'iife'],
  dts: true,                   // 也可传 { resolve: true, entry: 'src/index.ts' }
  sourcemap: true,             // 'inline' 也可
  minify: true,                // terser? no，esbuild 内置
  target: 'es2020',
  treeshake: true,
  splitting: true,             // 仅 ESM 生效
  clean: true,
  outDir: 'dist',
  skipNodeModulesBundle: true, // 默认不打包 node_modules（推荐）
  env: { NODE_ENV: 'production' },
  define: { __BUILD_TIME__: JSON.stringify(Date.now()) },
  esbuildOptions(options) {
    options.keepNames = true;  // 保留函数名便于调试
  }
});
```

> **类型声明注意**：`dts: true` 基于 rollup-plugin-dts，复杂 path 别名偶有解析问题；需更稳可改用 `tsc --emitDeclarationOnly` 独立生成。

---

## 8) 常见坑与修复 🧯

1. **ESM/CJS 地狱**  
   - 包含 CJS 依赖时，在 ESM 中 `import pkg from 'cjs-only'` 可能报错；使用动态导入或让 tsup 产出 **双格式**，消费端按需加载。  
   - `package.json` 有 `"type": "module"` 时，CJS 路径处理用 `fileURLToPath(import.meta.url)` 获取 `__dirname` 等价物。

2. **别名在运行器/打包器不一致**  
   - tsx 支持 tsconfig 路径需 `--tsconfig-paths` 或安装 `tsconfig-paths/register`。  
   - tsup 别名可用 `alias`（`tsup` 9+ 内置 via esbuild `alias`）或在源码侧避免深路径穿透。

3. **CLI shebang 丢失**  
   - 用 `banner.cjs: '#!/usr/bin/env node'`；或构建后在 CI 里 `chmod +x dist/cli.cjs`。

4. **类型打包失败/循环引用**  
   - 拆分类型入口，或改 `tsconfig` 的 `baseUrl/paths`，必要时用 `tsc --emitDeclarationOnly`。

5. **Node 版本差异**  
   - 指定 `target: 'node18'|'node20'`；跨版本特性（如 `fetch`）请在运行时 `polyfill` 或降级实现。

---

## 9) “一键新建项目”脚手架示例（degitting 模版）🪄

用一个小型 CLI 把 Git 模版拉到本地并做最小替换。

**src/cli.ts**
```ts
#!/usr/bin/env node
import { execSync } from 'node:child_process';
import { existsSync, writeFileSync } from 'node:fs';
import { join } from 'node:path';
import { fileURLToPath } from 'node:url';
import prompts from 'prompts';

const { name, template } = await prompts([
  { type: 'text', name: 'name', message: '项目名', initial: 'my-app' },
  { type: 'select', name: 'template', message: '模版', choices: [
      { title: 'library (tsup)', value: 'user/repo#lib' },
      { title: 'cli (tsup+tsx)', value: 'user/repo#cli' },
      { title: 'service (tsx)',  value: 'user/repo#service' }
    ] }
]);

if (existsSync(name)) throw new Error(`目录 ${name} 已存在`);
execSync(`pnpm dlx giget ${template} ${name}`, { stdio: 'inherit' });

const pkgPath = join(process.cwd(), name, 'package.json');
const pkg = JSON.parse(await (await import('node:fs/promises')).readFile(pkgPath, 'utf-8'));
pkg.name = name;
writeFileSync(pkgPath, JSON.stringify(pkg, null, 2));

console.log(`\n🎉 Created ${name}`);
console.log(`\n  cd ${name}\n  pnpm i\n  pnpm dev\n`);
```

**package.json（脚手架自身）**
```json
{
  "name": "create-awesome",
  "version": "0.0.0",
  "type": "module",
  "bin": { "create-awesome": "dist/cli.cjs" },
  "dependencies": { "prompts": "^2.4.2" },
  "devDependencies": { "tsup": "^8.0.0", "tsx": "^4.0.0", "typescript": "^5.0.0" },
  "scripts": { "build": "tsup src/cli.ts --format cjs --minify" }
}
```

> 用户使用：`npm create awesome@latest`（等价于 `npx create-awesome`），即可交互拉取模版。

---

## 10) CI / 发布流水线（GitHub Actions 片段）🤖

**.github/workflows/release.yml**
```yaml
name: release
on:
  push:
    branches: [main]
jobs:
  build-and-publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - name: Use Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: https://registry.npmjs.org
      - run: pnpm i --frozen-lockfile
      - run: pnpm run typecheck
      - run: pnpm run build
      - run: npm publish --provenance --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## 11) 提交前检查清单 ✅

- [ ] **开发用 tsx，发布用 tsup**，两条链路独立。  
- [ ] 包导出 **ESM + CJS + dts**，`exports`/`types`/`files` 配置齐全。  
- [ ] 依赖外部化合理：`peerDependencies` + `external`；`sideEffects:false` 保障 tree-shaking。  
- [ ] `tsconfig` 采用 `NodeNext` 或与运行环境匹配的模块解析。  
- [ ] CLI 产物含 shebang，可执行；服务产物目标环境明确（Node/Edge）。  
- [ ] CI 中分离 **类型检查** 与 **构建**，产物含 `sourcemap`（上传到错误上报平台）。  

---

## 12) 练习 🏋️

1. 把一个小工具库用 **tsup** 重构为双格式产出（ESM+CJS），并补上 `exports` 与 `types`。  
2. 写一个 **CLI**，`tsx watch` 开发，`tsup` 构建发布；支持 `--version`/`--help`。  
3. 用 **Turborepo** 搭一个 Monorepo：`packages/utils`（tsup）、`apps/api`（tsx dev + tsup build）、`apps/web`（Vite），验证缓存与依赖推导是否生效。  

---

**小结**：**tsx = 极速运行**，**tsup = 极速打包**。两者组合，能覆盖库、CLI、服务、脚手架等 80% 的工程场景，**快、稳、干净**。🚀
