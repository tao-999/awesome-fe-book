# 学习路径与全景图

> 本页提供面向不同目标的学习路线（阶段→里程碑→验证），以及与全书章节的映射。你可以按“地基→工程化→生产化→专项”的顺序，或直接按目标路线走完毕设项目。

---

## 0. 使用说明

- **时间粒度**：以 1–2 周为一个 Sprint，结束时必须产出可运行的 Demo 或文档。
- **验证优先**：每个 Sprint 以“验收清单”为准（性能、可用性、安全、可观测至少命中 2 项）。
- **章节映射**：每个任务都挂到本书对应章节，便于回查与扩展阅读。

---

## 1. 能力矩阵（自评表）

| 维度 | L1 入门 | L2 熟悉 | L3 独立交付 | L4 方案设计 | L5 体系化 |
|---|---|---|---|---|---|
| 语义化 & a11y | 知道语义标签 | 能做基本可访问表单 | 能做 WCAG 核查 | 能制定团队规范 | 能做无障碍审计与改造 |
| CSS 架构 | 会用 Flex/Grid | 能组件化样式 | 掌握 BEM/Tokens | 统一主题与暗色 | 设计跨平台 Design System |
| JS→TS | ESNext 语法 | 基本 TS 类型 | 严格模式与泛型 | 设计类型安全 API | 跨仓库类型治理 |
| 状态与数据 | 会用 SWR/Redux | 能分层与缓存 | 抽象数据边界 | 组合复杂数据流 | 统一跨应用数据契约 |
| 构建与发布 | Vite 基本使用 | 诊断与优化 | CI/CD 单仓 | Monorepo 流水线 | 版本/回滚/灰度体系 |
| 渲染形态 | 知道 CSR/SSR | 会用 Next/Nuxt | 选型并迁移 | BFF 与 ISR | Edge/多区域策略 |
| 性能与安全 | 懂 Web Vitals | 基本优化实践 | 有监控与报警 | 指标驱动优化 | 体系化治理 |
| 可观测性 | 记录错误 | 采集指标 | Trace 关联 | 自定义视图 | 观测平台与规范 |
| 专项（可视化/多媒体/WASM/跨端/AI/Web3） | 能跑 demo | 会改示例 | 场景落地 | 方案对比 | 体系与标准 |

> 建议把目标定在 L3 起步，关键栈（工程化/渲染/性能/可观测）冲到 L4。

---

## 2. 总体路线（阶段→里程碑）

### 阶段 A · 地基（2–3 周）
**目标**：以 TS + 现代 CSS + 基础可访问性打底。
- 里程碑 M1：完成一套 `Design Tokens` + 样式架构（BEM 或 Utility-first）。
- 里程碑 M2：将现有 JS 过渡到 TS，开启严格模式，消除 `any`。

**章节映射**：
- [foundations/semantics.md](../foundations/semantics.md) · [foundations/a11y.md](../foundations/a11y.md) · [foundations/css.md](../foundations/css.md) · [foundations/css-architecture.md](../foundations/css-architecture.md) · [foundations/js-ts.md](../foundations/js-ts.md) · [foundations/ts-advanced.md](../foundations/ts-advanced.md)

---

### 阶段 B · 工程化（2–3 周）
**目标**：搭建构建、质量、测试与 CI。
- 里程碑 M3：`Vite + ESLint + Prettier + Vitest + Playwright` 最小可运行基线。
- 里程碑 M4：GitHub Actions 完成构建与预览部署；变更即预览。

**章节映射**：
- [engineering/build-tools.md](../engineering/build-tools.md) · [engineering/quality.md](../engineering/quality.md) · [engineering/testing.md](../engineering/testing.md) · [engineering/ci-cd.md](../engineering/ci-cd.md)

---

### 阶段 C · 运行形态（2–3 周）
**目标**：掌握 CSR/SSR/SSG/ISR/Edge 的取舍与迁移。
- 里程碑 M5：用 Next/Nuxt/SvelteKit 实现 **同一业务** 的两种渲染模式 A/B。
- 里程碑 M6：引入 BFF（tRPC/GraphQL/REST 其一）与缓存策略。

**章节映射**：
- [runtime/rendering-modes.md](../runtime/rendering-modes.md) · [runtime/meta-frameworks.md](../runtime/meta-frameworks.md) · [runtime/bff.md](../runtime/bff.md) · [runtime/data-caching.md](../runtime/data-caching.md)

---

### 阶段 D · 生产化（持续）
**目标**：把“能跑”变成“可运维、可回滚、可观测、可演进”。
- 里程碑 M7：Web Vitals 达标（LCP/CLS/INP），有日志、指标与 Trace。
- 里程碑 M8：灰度发布与 Feature Flags，P0 故障 30 分钟内可回滚。

**章节映射**：
- [quality/perf.md](../quality/perf.md) · [quality/observability.md](../quality/observability.md) · [engineering/release-ops.md](../engineering/release-ops.md) · [quality/web-sec.md](../quality/web-sec.md)

---

## 3. 目标路线（任选其一或组合）

### 路线 R1 · 就业导向（8 周）
**目标**：1 份可展示的“企业级前端基线 + 典型业务 Demo”。

| 周数 | 主题 | 产出 |
|---|---|---|
| 1 | 地基：语义化/TS/CSS 架构 | Token + TS 严格模式改造清单 |
| 2 | 组件化与设计系统 | 10+ 基础组件 + Storybook 文档 |
| 3 | 数据与状态 | SWR/React Query + 分页/缓存策略 |
| 4 | 测试与质量 | 单测 70%/E2E 关键路径 100% |
| 5 | 渲染形态 | CSR/SSR/SSG 对比 Demo |
| 6 | BFF 与数据层 | tRPC/GraphQL 任一 + 接口契约 |
| 7 | 性能与安全 | Web Vitals 达标 + CSP/SRI |
| 8 | 可观测与发布 | Actions + 预览环境 + 回滚脚本 |

**章节导航**：优先 Part I–V、X；可选 Part II（按框架栈）。

---

### 路线 R2 · 全栈创业（12 周）
**目标**：一个可 Demo 的“全栈最小可行产品”（MVP）。
- 前端：Next/Nuxt/SvelteKit + Design System
- BFF：Cloudflare Workers/Vercel Functions
- 数据：KV/SQLite/PlanetScale/Turso 任一
- 支付：Stripe/PayPal 沙箱
- 观测：OpenTelemetry + Sentry

**章节导航**：Part II–V、IV、IX、X；选读 Part VII（桌面/小程序）以扩展触点。

---

### 路线 R3 · 可视化/多媒体/WASM（8 周）
**目标**：数据仪表盘或 3D/音视频处理的可运行产品雏形。
- D3/AntV 图层与交互
- Three.js + WebGPU（可退 WebGL）
- WebCodecs/WebRTC 基础链路
- Rust → WASM 的计算热区替换

**章节导航**：Part VI 全读；配合 Part V（性能）与 Part III（工程）。

---

### 路线 R4 · 前端 × AI & Web3（8 周）
**目标**：文件侧 RAG 前端或钱包/合约交互的安全 UX。
- 流式接口与前端流控
- 本地嵌入/切片与 Workers 协同
- 钱包连接（EIP-1193/6963）、签名与风险提示
- 合约交互与异常兜底

**章节导航**：Part VIII；配合 Part IV（BFF/缓存）与 Part V（安全/观测）。

---

## 4. 里程碑项目（可挑 2 个作为毕设）

1. **多形态渲染 Blog**：同一仓库实现 CSR/SSR/SSG，切换策略对比首屏指标。  
   - 参考：[runtime/rendering-modes.md](../runtime/rendering-modes.md) · [quality/web-vitals.md](../quality/web-vitals.md)

2. **IM/聊天室最小实现**：WebSocket + 鉴权 + 消息持久化 + E2E 小集。  
   - 参考：[arch/case-im.md](../arch/case-im.md) · [arch/realtime-crdt.md](../arch/realtime-crdt.md)

3. **协作文档/白板**：Yjs/CRDT，同步冲突解决，离线与恢复。  
   - 参考：[arch/case-collab.md](../arch/case-collab.md)

4. **电商最小闭环**：列表→购物车→下单→支付沙箱→订单页，埋点与风控提示。  
   - 参考：[arch/case-ecommerce.md](../arch/case-ecommerce.md)

5. **WebGPU 可视化/渲染**：百万点渲染或体积数据可视化，降级策略。  
   - 参考：[viz/three-webgpu.md](../viz/three-webgpu.md)

---

## 5. 验收清单（每个 Sprint 至少满足）

- [ ] **功能**：功能闭环可演示（录屏/线上预览链接/说明文档）。  
- [ ] **质量**：单测 ≥ 60%，关键路径 E2E 通过；Lint/Format 无阻塞。  
- [ ] **性能**：Web Vitals 至少 2 项达标（记录测试环境与日期）。  
- [ ] **安全**：基本 CSP/SRI/依赖审计通过（高危 CVE 不得上线）。  
- [ ] **观测**：错误日志 + 性能指标上报（有可视化面板/截图）。  
- [ ] **文档**：README 含运行步骤、配置、边界与常见问题。

---

## 6. 学习节奏建议（每周 6–10 小时）

| 时间块 | 内容 |
|---|---|
| 2h | 章节精读 + 做笔记（概念与对比） |
| 2h | 跟打示例/复刻最小 Demo |
| 1h | 读官方文档/标准，补齐细节 |
| 1h | 写总结或复盘，形成复用清单 |
| 1–4h | 项目化练习/毕设推进 |

> 卡住超过 1 小时：回到“问题定义→最小复现→对比两种实现→记录结论”。

---

## 7. 路线到章节的速查表

| 目标 | 必读 | 选读 |
|---|---|---|
| 就业导向 | Part I–V、X | Part II（按栈）、IX |
| 全栈创业 | Part II–V、IV、IX | Part VII（桌面/小程序） |
| 可视化/多媒体/WASM | Part VI、V、III | Part VII |
| 前端 × AI & Web3 | Part VIII、IV、V | IX（实时/协作） |

---

## 8. 常见误区与纠偏

- **只刷语法不做项目** → 每周至少一个可运行小结（哪怕是“打印指标”）。  
- **只优化首屏不做可观测** → 任何性能改动都要记录与对比。  
- **测试缺位** → 先写 E2E 的“业务关键路径”，再补单测。

---

## 9. 下一步

选择你的目标路线（R1–R4 之一），创建 `examples/<slug>` 目录，用阶段 A 的地基与阶段 B 的工程化基线起步；每个 Sprint 结束提交“验收清单 + 截图/链接”。当两个“里程碑项目”达标，即完成本页学习路径的阶段性目标。
