# 12.2 版本回滚 / 灰度 / Feature Flags 🔁🌗🏳️

本章目标：把**上线**从“悬崖跳水”改成“有护栏的坡道”。三件套：
- **回滚**（Rollback）：遇险**快速恢复**上一稳定版本。
- **灰度**（Canary/Blue-Green/Ring）：**小流量试水**，看指标再放大。
- **Feature Flags**（特性开关）：**代码已合并，行为可控**，随时开/关/按人群投放。

---

## 0) 心智图（先定调）
- **先可回滚，后敢灰度**：没有可回滚的发布 = 生产自杀。  
- **度量驱动**：一切放量/回滚决策由指标说话（错误率、延迟、转化、崩溃率）。  
- **幂等与原子**：发布/回滚应原子化（CDN 原子切换、K8s 声明式、可重复执行）。  
- **开关不等于安全控制**：Feature Flag 不是权限系统，**鉴权在后端**。

---

## 1) 快速回滚（不同形态）⚡

### 1.1 Web 前端（静态站/SPA）
- **内容寻址/不可变产物**：构建产物按哈希命名（`main.[hash].js`），**每次部署都可追溯**。  
- **原子切换**：用“**当前版本指针**”（alias）指向某个构建：
  - Vercel：`vercel --prod` 部署并可在 UI/CLI 选回上一版。  
  - Netlify：`netlify deploy --prod --dir=dist`，可在 **Deploys** 里 **Rollback**。  
  - Cloudflare Pages：每次 build 都是独立版本，切回旧 build 即回滚。  
- **缓存策略**：HTML `no-cache` + 资产 `immutable, max-age=31536000`，避免“旧 HTML + 新 JS”的撕裂。  
- **Service Worker**：默认**不强更**。提供「有新版本，点击刷新」提示，比 `skipWaiting()` 强行切换更安全。

### 1.2 Node/BFF/后端
- **蓝绿（Blue-Green）**：两组环境同时存在，**切流量**而非重部署。  
- **K8s 回滚**：  
  ```bash
  kubectl rollout undo deploy web-api --to-revision=5
  kubectl rollout status deploy web-api
  ```
- **数据库变更**：遵循 **Expand → Migrate → Contract** 三步（见 §4）。**数据库不可回滚**是事故之源。

### 1.3 移动端
- 应用商店版本无法立刻回滚 → **远程配置/Feature Flag** 做 **Kill-Switch**（紧急关闭新功能或降级策略）。  
- 分阶段发布（Store 分流）+ 崩溃率阈值触发停更。

---

## 2) 灰度发布（Canary / Blue-Green / Ring）🌗

### 2.1 策略对比
| 策略 | 说明 | 适用 |
|---|---|---|
| **Canary** | 小比例流量上新版本，指标过线逐步放大 | Web/API 常用 |
| **Blue-Green** | 两套环境，切换路由指针 | 大改动、零停机 |
| **Ring/Rolling** | 分批按区域/用户组扩散 | 大体量、多地域 |

### 2.2 K8s（Istio 权重路由示例）
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata: { name: web }
spec:
  hosts: ["web.example.com"]
  http:
    - route:
        - destination: { host: web, subset: v2 }
          weight: 10     # 10% → 25% → 50% → 100%
        - destination: { host: web, subset: v1 }
          weight: 90
```

### 2.3 NGINX Ingress Canary（简版）
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
```

### 2.4 指标闸门（自动化放量/回滚）
- 设定 **SLO 守门**：5xx 率、P95/P99 延迟、前端错误率、核心转化。  
- 评估窗口（如 10–15 分钟）稳定后再放量；任一阈值越界 → **自动回滚到上一权重**。

---

## 3) Feature Flags（特性开关）🏳️

### 3.1 类型
- **Release Flag**：控制功能上线/灰度/回滚。  
- **Experiment Flag**：A/B / 多臂 bandit。  
- **Ops Flag**：限流、降级、Kill-Switch。  
- **Permission Flag**：按租户/角色开关（仍需服务端鉴权）。

### 3.2 设计要点
- **默认安全**：未取到配置时走**保守默认值**。  
- **一致性**：用 **稳定哈希**（userId/tenantId）做桶，避免用户刷新跳桶。  
- **最小隐私**：Flag 评估尽量在**服务端**（或 Edge），客户端仅接收**与自身相关**的判定结果。  
- **可观测**：每次评估**打曝光日志**（flag 名、变体、用户键），供实验与回滚决策使用。  
- **生命周期**：创建→上线→放量→**清理死旗**（债务）。

### 3.3 极简实现（TypeScript，演示概念）
```ts
// flags.ts
export type Variant = 'on' | 'off' | 'A' | 'B';
export type Context = { userId?: string; tenantId?: string; [k: string]: unknown };

export interface Provider {
  get(flag: string, ctx: Context): Variant | undefined;
}

export class InMemoryProvider implements Provider {
  constructor(private store: Record<string, any>) {}
  get(flag: string, ctx: Context) {
    const def = this.store[flag];
    if (!def) return undefined;
    if (def.percentage != null) {
      const key = String(ctx.userId ?? ctx.tenantId ?? 'anon');
      const bucket = (hash(key + flag) % 100);
      return bucket < def.percentage ? 'on' : 'off';
    }
    if (def.variants) {
      const key = String(ctx.userId ?? 'anon');
      const bucket = hash(key + flag) % 100;
      let acc = 0;
      for (const [name, pct] of Object.entries<number>(def.variants)) {
        acc += pct;
        if (bucket < acc) return name as Variant;
      }
    }
    return def.default ?? 'off';
  }
}

function hash(s: string) { // 一致性哈希（演示）
  let h = 2166136261;
  for (let i = 0; i < s.length; i++) h = (h ^ s.charCodeAt(i)) * 16777619;
  return Math.abs(h >>> 0);
}

// 使用
const provider = new InMemoryProvider({
  'checkout-new': { percentage: 10, default: 'off' },      // 10% 灰度
  'search-algo': { variants: { A: 50, B: 50 }, default: 'A' }
});

export function isOn(flag: string, ctx: Context) {
  return provider.get(flag, ctx) === 'on';
}
```

> 生产建议：采用 **OpenFeature** 标准或托管服务（LaunchDarkly、Unleash、Flagd、ConfigCat 等），具备审计、目标人群、可视化与 SDK。

### 3.4 前端使用（React 例）
```tsx
{isOn('checkout-new', { userId }) ? <NewCheckout /> : <LegacyCheckout />}
```

---

## 4) 数据库与回滚（**Expand → Migrate → Contract**）🧱

1. **Expand**：先**加列/表/索引**，新老代码都能读写。  
2. **Migrate**：后台迁移数据（幂等、可重试）；双写/读时**验证一致性**。  
3. **Contract**：确认无老读写后**移除旧字段**。  

> 任何需要 **回滚** 的变更，都必须确保旧版本在 Expand 阶段仍可工作（避免“不可回滚的 DDL”）。

---

## 5) 自动化放量与一键回滚（CI/CD 配置片段）🤖

### 5.1 GitHub Actions：阶段放量（伪代码）
```yaml
jobs:
  canary:
    steps:
      - run: kubectl apply -f k8s/virtualservice-10.yaml
      - run: node scripts/wait-slo-ok.js --window=15m --p95<300 --err<0.5%
      - run: kubectl apply -f k8s/virtualservice-25.yaml
      - run: node scripts/wait-slo-ok.js --window=15m
      - run: kubectl apply -f k8s/virtualservice-50.yaml
  rollback_on_fail:
    if: failure()
    steps:
      - run: kubectl apply -f k8s/virtualservice-0.yaml   # 全量回退到 v1
```

### 5.2 灰度与开关联动
- 灰度阶段**同步**开 `release flag`，一旦指标越界，流水线自动：**关 Flag → 降权重/回滚**。  
- **幂等脚本**：多次执行结果一致；失败可重试。

---

## 6) 运营与实验 📊
- **曝光/点击/转化**打点与 flag 变体**强关联**；不合并数据会导致错判。  
- **样本守恒**：分桶后不得再按用户属性“偏置筛选”，否则实验失真。  
- **停留时间**：给实验足够时长覆盖峰谷周期；避免只看短期噪声。

---

## 7) 安全与合规 🛡️
- **不要把敏感逻辑仅靠 Flag 隐藏在前端**；服务端仍需**硬鉴权**。  
- Flag 名/键不泄露机密（比如「`flag-enable-secret-experiment`」）。  
- 变体评估日志**脱敏**，遵守隐私法规（GDPR/CCPA 等）。

---

## 8) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| “一刀切全量上” | 大故障 | 先 5–10% Canary，指标 OK 再放大 |
| 前端强制 SW 立更 | 白屏/资源错版 | 弹窗提示刷新；原子切换 + 缓存策略 |
| DB 直改/删列 | 无法回滚 | **Expand/Migrate/Contract** |
| Flag 无默认值 | 离线就炸 | 默认 **off**；失败容忍并降级 |
| 永久不清旗 | 代码债堆积 | 为每个旗设 **到期清理**任务 |
| 把 Flag 当权限 | 越权风险 | 权限在后端，Flag 仅控制 UI/行为 |
| 灰度无指标 | 放量凭感觉 | 定义 **SLO 阈值** + 自动化闸门 |

---

## 9) 提交前检查清单 ✅
- [ ] 本次变更具备**单键回滚**（上一版本可直接切回）。  
- [ ] CDN/缓存策略与构建产物**原子切换**校验通过。  
- [ ] 灰度计划：阶段、阈值、观察窗口、回滚条件已配置。  
- [ ] 相关 Feature Flag 已创建：默认值、安全回退、审计记录。  
- [ ] 数据库变更按 **Expand/Migrate/Contract** 执行，迁移脚本幂等。  
- [ ] 观测项（错误率、延迟、关键业务转化）已在看板与报警中。  
- [ ] 回滚剧本（Runbook）在 README/值班手册，团队演练过一次。

---

## 10) 练习 🏋️
1. 为「搜索改造」加一个 **release flag**，按 10%→25%→50% 放量；超过“错误率 0.5%”或“P95>300ms”自动回滚。  
2. 给「结算页」做一次 **Blue-Green** 发布演练，并在 CDN 上验证静态资源**原子切换**。  
3. 按 **Expand/Migrate/Contract** 为 `orders` 增加 `country` 字段，完成双写迁移并清列回收。  

---

**小结**：**回滚**保命、**灰度**试水、**Flag**控节奏。三者协作，把“上线”这件事从一次性冒险变成可度量、可回退、可优化的工程过程。🚀
