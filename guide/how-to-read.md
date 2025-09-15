# 如何阅读 & 约定

> 🧭 这页定义全书的阅读姿势、术语、代码与排版约定。按它走，整本书的示例和清单都能无脑复用。

---

## 1) 推荐阅读路线 🎯

- **首刷（3–5 天）**：Part I–III + Part V 的核心章节，先把“语义化 / CSS 架构 / TS / 工程化底座 / 性能与安全”过一遍。  
- **二刷（按需）**：结合实际项目，选择 Part IV（渲染 & 服务形态）、Part VI（可视化/WASM）、Part VII（跨端）。  
- **专题深潜**：Part VIII–IX（AI & Web3、微前端、实时协作、行业案例）。  
- **随手查**：附录里的检查表与模板，发布前逐条走查即可。

> 章节会标注难度标签：`[必修]`、`[选修]`、`[进阶]`。先必修，后按需。

---

## 2) 术语与风格 📚

- **专有名词**：首次出现写 *英文*（中文注释），例：*Service Worker（离线缓存工作线程）*。后文可只写英文。  
- **大小写**：专有名词与 API 保持官方写法（如 *GitHub*、*TypeScript*、*WebAuthn*）。  
- **标点**：中文正文用全角标点，内嵌代码/命令用半角。英数字与中文之间保留空格：`GraphQL 网关`。  
- **图注**：图片需有描述性 *alt* 与编号，例如：`图 3-2：Edge 渲染请求链`。  
- **示例前置条件**：每段代码要交代运行环境与适用边界（Node 版本、浏览器支持、是否需 https）。

---

## 3) 运行环境与代码风格 🧪

- **语言**：以 **TypeScript** 为主，少量 JavaScript 仅作对比。  
- **运行时**：Node.js ≥ **20**（ESM 优先）。浏览器目标为“现代浏览器近两年版本”。  
- **包管理器**：默认 **pnpm**。如需 npm/yarn，会附对照命令。  
- **模块**：示例默认 **ESM**（`type: "module"`），需 CJS 会明确标注。  
- **格式化**：统一 **Prettier**；Markdown 使用 **Markdownlint**。  
- **安全基线**：示例依赖尽量锁定次级版本（`~`），涉及网络/鉴权/密钥的片段给出最小可运行的 **本地 mock**。

### 建议的最小配置（可直接复用）

`package.json` 片段：
~~~json
{
  "type": "module",
  "engines": { "node": ">=20" },
  "scripts": {
    "dev": "node --watch src/index.ts",
    "build": "tsc -p tsconfig.json",
    "lint:md": "markdownlint \"**/*.md\" -c .markdownlint.json",
    "fmt": "prettier \"**/*.{md,ts,tsx,js,css,json}\" --write"
  },
  "devDependencies": {
    "prettier": "^3.3.0",
    "markdownlint-cli": "^0.41.0",
    "typescript": "^5.6.0"
  }
}
~~~

`tsconfig.json`（基础版）：
~~~json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "strict": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "noUncheckedIndexedAccess": true,
    "types": []
  },
  "exclude": ["dist", "node_modules"]
}
~~~

`.prettierrc`：
~~~json
{ "printWidth": 100, "singleQuote": true, "semi": true }
~~~

`.markdownlint.json`：
~~~json
{
  "default": true,
  "MD013": { "line_length": 100 },
  "MD033": false,
  "MD041": false
}
~~~

---

## 4) 命令行对照表 🧰

| 语义 | pnpm | npm | yarn |
|---|---|---|---|
| 安装依赖 | `pnpm i` | `npm i` | `yarn` |
| 装开发依赖 | `pnpm i -D <pkg>` | `npm i -D <pkg>` | `yarn add -D <pkg>` |
| 运行脚本 | `pnpm dev` | `npm run dev` | `yarn dev` |
| 清缓存 | `pnpm store prune` | `npm cache clean --force` | `yarn cache clean` |

> 文中若只给出 `pnpm`，你可按上表替换为 `npm/yarn`。

---

## 5) 目录与示例工程结构 🗂️

- 本书示例集中在 `examples/`，每章一个子目录：`examples/<slug>`。  
- 每个示例包含：`README.md`（如何运行）、`package.json`（脚本）、`src/`（最小可运行代码）、可选 `docker-compose.yml`（需要服务时）。

示例结构模板：
~~~
examples/
  csr-cache-hints/
    README.md
    package.json
    tsconfig.json
    src/
      main.ts
~~~

---

## 6) 链接与引用 🔗

- **优先级**：官方文档 > 标准/规范 > 一手权威文章 > 生态博文。  
- **相对链接**：站内文档使用相对路径（方便迁移）。  
- **更新声明**：引用若与主流版本差异较大，正文给出“测试版本与日期”。

---

## 7) 图片、图表与公式 🖼️

- 图片放在当前章节的 `assets/` 目录：`part-x/.../assets/xxx.png`。  
- 需可访问的 *alt* 文本；涉及流程/架构优先使用矢量与文本化图（如 Mermaid 可选用，但不强制）。  
- 公式默认 KaTeX 语法，适度使用。

---

## 8) 兼容性与版本标签 🧭

- 每个涉及运行时/框架的章节会在页首给出 **Badge**：  
  - 例：`Node ≥20 | Vite 5 | React 19 | TypeScript 5.6`  
- 若代码对旧版本有兼容写法，会以 **“兼容路径”** 小节单独说明。

---

## 9) 提交与版本管理 🧾

- **提交规范**：Conventional Commits（`feat: …` / `fix: …` / `docs: …` / `chore: …` / `refactor: …`）。  
- **文档改动**：大章节改动需在章节顶部追加 `Changelog` 块：  
  - `2025-09-15 fix：更新 PWA 缓存章节示例到 Workbox v7`。

---

## 10) 质量保障与发布前检查 ✅

- [ ] 代码可运行（含 `README` 步骤）。  
- [ ] 章节内所有链接可达、图片有 *alt*。  
- [ ] 使用统一术语，避免同概念多译。  
- [ ] 性能/安全片段给出“适用边界 & 风险”说明。  
- [ ] 通过 `pnpm fmt` 与 `pnpm lint:md`。  
- [ ] 若含基准数据，标注测试环境与日期。

---

## 11) 新章节模板 🧱

将以下模板粘到新章节文件开头，按需删改：

~~~markdown
# <章节标题> 〔必修/选修/进阶〕

> 一句话目标：这章让你能够 _______ ，并在 _______ 场景下做出可验证的选型。

## 背景 & 问题
- 这个主题试图解决什么问题？与相邻主题的边界？

## 快速上手（10 分钟）
1. 环境要求：Node ≥20、浏览器版本、工具链……
2. 步骤：克隆示例 → 安装 → 运行 → 看到什么结果
3. 常见错误与排查

## 实战清单（可直接套用）
- [ ] 配置项 A：为什么、默认值、推荐值  
- [ ] 配置项 B：与 A 的取舍  
- [ ] 观测指标：如何埋点与评估

## 对比与选型
- 维度表：功能覆盖 / DX / 性能 / 生态 / 观测 / 成本
- 何时选择 X，何时选择 Y

## 风险与边界
- 安全、性能、兼容性、法规相关注意

## 延伸阅读
- 官方文档 / 标准 / 关键文章列表
~~~

---

## 12) 反馈与勘误 📮

- 建议通过仓库 **Issue/PR** 反馈：描述问题、截图/复现、环境信息。  
- 页面底部的“反馈”入口也可直达；会按“勘误—优化—新增”优先级处理。

---

## 13) 许可与署名 🪪

- 文档内容：建议 **CC BY-NC-SA 4.0**。  
- 代码示例：建议 **MIT**。  
- 商用或培训引用请先取得授权；若引用本书片段，请保留来源链接与作者署名。
