# 10.2 代码审查与静态扫描（SAST）🔍🛡️

本章目标：把**人工代码审查（Code Review）**与**静态应用安全测试（SAST）**打成组合拳：**小而频的 PR、明确的评审准则、自动化的安全扫描、可追溯的整改流程**。做到“**人改设计与意图**，**机抓模式与漏洞**”。

---

## 0) 原则（先立规矩）⚖️

- **小而频**：PR 目标单一、变更 ≤ 300 行差异（不含生成物），**越小越快**。
- **前置自动化**：把能用机器做的事（格式、lint、测试、安全扫描）**在 PR 打钩之前就跑**。
- **面向风险**：审查优先级 = **安全 > 正确性 > 可维护性 > 性能 > 风格**。
- **记录闭环**：问题 → 结论/修复 → 链接 Issue/工单，避免“口头合规”。

---

## 1) 角色与流程🧑‍💻👀

- **作者（Author）**：拆小 PR、写清动机与影响面、补充用例/截图/Benchmark。
- **审查者（Reviewer）**：从**风险**与**可维护性**出发；对风格争议**给出理由或引用规范**。
- **守门（Gate）**：CI 绿灯、必需 CODEOWNERS 通过、SAST 关键告警已处理/豁免。

**典型流水：**起 PR → 机器人跑 Lint/测试/构建/SAST → Reviewer 看 **设计/边界/复杂度** → 通过后合并（Rebase/Merge 依团队策略）。

---

## 2) 审查清单（通用 + 前端专项）📋

### 通用
- [ ] 需求/动机清晰（PR 模板三问：**为何改、改了啥、如何验证**）。
- [ ] 责任边界单一：**是否能再拆**？
- [ ] 命名/抽象可读：**模块接口稳定**，避免泄露实现细节。
- [ ] 错误处理与日志：失败路径可追；**不要吞错**。
- [ ] 单测/端测覆盖关键路径；回归风险有说明。

### 前端专项
- [ ] **XSS 面**：是否写入 `innerHTML` / `dangerouslySetInnerHTML`？必须的话是否**白名单+消毒**？
- [ ] **URL 注入/跳转**：`location.href`/`window.open` 是否校验域名与协议？
- [ ] **敏感数据**：是否在客户端**持久化**了 token/PII？是否最小化采集？
- [ ] **权限/鉴权**：前端仅做**体验层**校验，**关键授权由服务端**实现。
- [ ] **a11y**：键盘可达、语义标签、`aria-*`、`prefers-reduced-motion`。
- [ ] **性能**：是否只动 `transform/opacity` 动画；是否懒加载大依赖与路由分包。
- [ ] **可观测性**：错误上报、埋点是否覆盖关键事件；**不上传 PII**。

### Node/BFF（若有）
- [ ] 外部输入统一校验（zod/joi/类验证器）。
- [ ] 禁用危险 API（`child_process.exec`、`vm`、`eval`）或最小化能力沙箱。
- [ ] 文件/路径拼接用 `path.join`，避免目录穿越。
- [ ] 上游请求超时/重试/断路；下游错误分类。

---

## 3) PR 模板与 CODEOWNERS 🧱

**.github/PULL_REQUEST_TEMPLATE.md**
```md
## 动机
- 解决了什么问题 / 用户价值

## 变更点
- [x] 功能/接口/Schema 变更
- [x] UI 截图（如有）
- [x] 风险与回滚方案

## 验证
- [x] 单测通过（覆盖率变更：xx→xx）
- [x] E2E/手测步骤
- [x] 性能/安全要点：无 `dangerouslySetInnerHTML`；新增接口已做输入校验
```

**CODEOWNERS**
```
# 关键目录必须有守门人
apps/web/src/security/*   @sec-team
packages/ui/**            @design-systems
```

---

## 4) 自动化闸门（CI 任务顺序）🤖

1. **Fmt/Lint/Typecheck**（Prettier、ESLint、TS）  
2. **单测 + 覆盖率阈值**  
3. **构建**（产物校验）  
4. **SAST**（Semgrep/CodeQL）  
5. **SCA 依赖扫描**（OSV/npm audit）  
6. **Secrets 扫描**（Gitleaks 等）

> 任何**高危**告警（安全/秘密泄露）默认阻塞合并；可由安全角色临时豁免并记录。

---

## 5) SAST 工具矩阵（前端友好）🧪

| 目标 | 工具 | 场景 |
|---|---|---|
| 代码模式/漏洞规则 | **Semgrep** | 轻量、规则易写，JS/TS/React/Vue/Node 友好 |
| 语义级安全分析 | **CodeQL** | 代码语义图，深层数据流（taint）分析 |
| 全量质量/Sonar 规则 | **SonarQube/Lint** | 质量门（Quality Gate）、安全/坏味道 |
| ESLint 安全补强 | `eslint-plugin-security` / 自定义规则 | 快速补洞（如禁 `eval`、`Function`） |
| Secrets 扫描 | **Gitleaks** / truffleHog | API Key/Token 提前拦截 |
| 供应链（SCA） | **OSV-Scanner** / `npm audit` | 依赖漏洞、许可证核查 |

---

## 6) 高危模式（JS/DOM 常见）🚨

- DOM XSS：`element.innerHTML = userInput`、`insertAdjacentHTML(user)`。  
- React：`dangerouslySetInnerHTML` 未用可信模板；`useEffect` 内直接插入脚本。  
- 模板注入：字符串拼 html/JS 事件（`<a onclick="${user}">`）。  
- Open Redirect：`location.href = query.redirect` 未校验。  
- 原生 `fetch`：组装 URL 时未限制协议/主机；`headers` 注入。  
- 反序列化：`JSON.parse` 处理**不可信**字符串并直接写 DOM。  
- 运行时执行：`eval`、`new Function`、`setTimeout(user, 0)`（函数字符串）。  
- Node 侧：`exec`/`spawn` 拼接 shell；`fs` 基于用户路径直读写；模板引擎**未关闭**不安全选项。

---

## 7) Semgrep 最小可用规则集（示例）🧰

**semgrep.yml**
```yaml
rules:
  - id: dom-xss-innerhtml
    languages: [javascript, typescript]
    message: "避免直接使用 innerHTML 写入不可信数据（DOM XSS）"
    severity: ERROR
    patterns:
      - pattern: $EL.innerHTML = $X
      - metavariable-pattern:
          metavariable: $X
          pattern-not: |
            sanitize($Y)
    metadata: { cwe: 'CWE-79' }

  - id: open-redirect
    languages: [javascript, typescript]
    message: "潜在 Open Redirect：请校验 redirect 域名或使用 allowlist"
    severity: WARNING
    patterns:
      - pattern-either:
          - pattern: location.href = $X
          - pattern: window.location = $X
      - metavariable-pattern:
          metavariable: $X
          pattern-not: allowedRedirect($Y)

  - id: no-eval-function-ctor
    languages: [javascript, typescript]
    message: "禁用 eval/new Function"
    severity: ERROR
    pattern-either:
      - pattern: eval($X)
      - pattern: new Function($ARGS, $BODY)
```

**CI 运行**
```bash
pnpm dlx semgrep --config semgrep.yml --error
```

---

## 8) CodeQL 工作流（GitHub Actions 片段）🧬

**.github/workflows/codeql.yml**
```yaml
name: CodeQL
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
jobs:
  analyze:
    permissions:
      security-events: write
      actions: read
      contents: read
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with: { languages: javascript-typescript }
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3
```

> 大仓/Monorepo 可针对关键路径设置矩阵与路径过滤，减少误报与耗时。

---

## 9) ESLint 安全护栏（补丁式）🧯

**安装**
```bash
pnpm add -D eslint-plugin-security eslint-plugin-no-secrets
```

**eslint.config.mjs 增补**
```js
import security from "eslint-plugin-security";
import noSecrets from "eslint-plugin-no-secrets";

export default [
  // ...
  {
    plugins: { security, "no-secrets": noSecrets },
    rules: {
      "security/detect-eval-with-expression": "error",
      "security/detect-new-buffer": "error",
      "security/detect-non-literal-fs-filename": "warn",
      "no-secrets/no-secrets": ["warn", { tolerance: 4.2 }] // 粗测密钥（配合 Gitleaks 更稳）
    }
  }
];
```

---

## 10) Secrets 扫描（Gitleaks）🔑

**配置**
```bash
pnpm dlx gitleaks protect --staged
```

**Git 钩子（Husky）**
```bash
npx husky add .husky/pre-commit "pnpm dlx gitleaks protect --staged --verbose"
```

**CI（可选）**
```yaml
- name: Gitleaks
  uses: gitleaks/gitleaks-action@v2
  with: { args: "--redact --verbose" }
```

---

## 11) 供应链（SCA）与许可证合规 🧩

**OSV-Scanner**
```bash
pnpm dlx osv-scanner --sbom=<(pnpm dlx syft dir:. -o cyclonedx-json)
# 或直接扫描锁文件
pnpm dlx osv-scanner -L pnpm-lock.yaml
```

**npm audit（快速兜底）**
```bash
pnpm audit --audit-level=high
```

> 许可证核查：对“强传染”协议（如 GPL 系列）保持警惕；在商业发行前跑一次 SBOM + 许可证报告。

---

## 12) 误报处置与基线管理 🧘

- **先修真问题**，再谈豁免。  
- 误报需：**证据**（为何安全）+ **到期时间** + **负责人**；不要永久豁免。  
- 用 **baseline** 文件记录已知告警，**新告警即红线**（守住增量）。

---

## 13) 指标与改进 📈

- **PR 周转**：中位审查时长（目标 ≤ 1 工作日）。  
- **PR 尺寸**：平均/95 分位 diff 行数（越低越好）。  
- **安全债务**：未关闭的高危告警数、平均关闭时间。  
- **覆盖率**：关键模块单测/端测覆盖率；回归缺陷率。  
- **自动化命中率**：CI 缓存命中、SAST 误报率。

---

## 14) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| “LGTM 文化” | 走过场、问题漏网 | 模板与清单强制；关键目录强制双审 |
| 大一统巨型 PR | 审不动、回滚困难 | 切功能旗标，拆小步合并 |
| 人工查格式 | 浪费时间 | Prettier/ESLint 前置，审**意图** |
| 只看 happy path | 边界缺失 | 审查失败路径、网络波动、异常态 |
| SAST 全绿即安全 | 过度信任工具 | **工具 + 评审 + 测试** 三位一体 |
| 永久豁免 | 安全债堆积 | 豁免需时效与负责人，定期复盘 |

---

## 15) 提交前检查清单 ✅

- [ ] PR 目标单一、差异量合理；模板信息完整。  
- [ ] CI 绿灯：Fmt/Lint/Type/测试/构建全过。  
- [ ] SAST 无高危；中危有处置说明或临时豁免（带到期）。  
- [ ] 未引入 `innerHTML`/`eval` 等高危模式；URL 跳转已校验。  
- [ ] 新接口有输入校验与错误处理；埋点/日志不含敏感数据。  
- [ ] 必要截图/录屏、性能对比、回滚方案已附。

---

## 16) 练习 🏋️

1. 给项目加上 **Semgrep** 与 3 条自定义规则（DOM XSS、Open Redirect、Eval），在 PR 阶段阻塞高危。  
2. 为关键目录设 **CODEOWNERS**；把 PR 模板接入并添加“回滚方案”段落。  
3. 在 Husky **pre-commit** 接上 **Gitleaks** 与 **lint-staged**；在 CI 中跑 **CodeQL** 与 **OSV-Scanner**，并产出一份 SBOM。

---

**小结**：**代码审查**确保“方向与意图正确”，**SAST**确保“模式与细节不出事”。把两者接上自动化与指标，你的仓库会又干净又安全，像洗过的代码一样丝滑。🧴✨
