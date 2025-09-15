# 19.2 A/B 实验与指标体系 🧪📈

目标：把“怎么做实验”“怎么读指标”“怎么做决策”一口气讲清，给到**可直接抄的落地流程、SQL/统计套路与检查表**。口号：**先定义，后试验；先设计，后统计。**

---

## 0) TL;DR（三句话记牢）🎯
1. **先建指标体系**（北极星/驱动/护栏），再谈实验；所有指标**有严格口径**与审计 SQL。  
2. **随机化与数据质量**第一优先：AA 测试 + SRM（样本配比）监控 + 曝光日志**强一致**。  
3. 统计上 **控制方差**（CUPED/分层/回归调整），**控制错误**（多重检验/FDR），**控制作恶**（预注册/不随便偷看）。

---

## 1) 基本概念与对象 🧩
- **实验单元（Unit）**：按**用户**优先，其次设备/会话；电商多选用户，内容曝光可选会话。  
- **曝光（Exposure）**：用户被分到某变体并**有机会受到处理**的那一刻（需日志记录）。  
- **触发（Trigger）**：只对“触发了功能”的子集计量（Triggered Analysis），避免被无关用户稀释。  
- **变体（Variant）**：A=控制组，B/C…=处理组；**命名唯一**，分桶**粘性**（hash 持久）。  
- **命名空间（Namespace）**：同一流量池中的多个实验**互斥管理**；跨命名空间可并行。

---

## 2) 指标体系（北极星/驱动/护栏）🧭
### 2.1 三层结构
- **北极星（NSM）**：业务最终目标（如 “28 天留存 DAU” / “订单 GMV”）。  
- **驱动指标（Input/Mechanism）**：直接可被实验影响（如 “首单转化率”、“每用户搜索次数”）。  
- **护栏指标（Guardrail）**：不允许恶化（如 “App 崩溃率”“延迟 p95”“退款率”“投诉率”）。

### 2.2 指标口径要素
- **分子/分母**（Rate vs Ratio vs Mean）  
- **时间窗**（同/跨会话、日/周/28天滚动）  
- **去重与归因**（首触/末触、会话内唯一）  
- **取分位还是均值**（p50/p75/p95 vs mean）  
- **触发口径**（只对触发人计算）  
- **方向性**（越大越好/越小越好）与**阈值**（最小可行改善 MDE；护栏底线）

> 把指标定义写成**DSL 或 SQL 视图**（dbt/Metric Layer），每次发布都跑**回归测试**。

---

## 3) 数据模型与埋点设计（前提工作）🧱
### 3.1 表与字段（最小集合）
- **exposures（曝光表）**  
  `unit_id, exp_name, variant, ts_exposed, namespace, hash_key, app_version`
- **events（事件表）**  
  `unit_id, event, ts, props(json), session_id, page, revenue, latency_ms`
- **identities（身份表）**  
  `device_id, user_id, first_seen_at, merge_rule`

要求：
- **粘性分桶**：`variant = hash(unit_id, exp_name) % K`；**上线前冻结 hash 秘钥**。  
- **AA 测试**：无处理但分桶，验证**SRM（样本比例不匹配）**与系统链路。  
- **身份并轨**：匿名→登录后的 **Identity Resolve** 策略**一锤定音**（例如：合并到 user_id，device_id 仅兜底）。

### 3.2 触发样例 SQL（只对“点击搜索框”的用户计算转化）
```sql
WITH triggered AS (
  SELECT DISTINCT unit_id
  FROM events
  WHERE event = 'search_focus' AND ts >= :start AND ts < :end
),
joined AS (
  SELECT e.unit_id, x.variant,
         COUNTIF(ev.event='purchase') > 0 AS purchased
  FROM exposures x
  JOIN triggered e     ON e.unit_id = x.unit_id AND x.exp_name = 'search_hint'
  LEFT JOIN events ev  ON ev.unit_id = e.unit_id AND ev.ts BETWEEN :start AND :end
  GROUP BY 1,2
)
SELECT variant, AVG(purchased) AS cr
FROM joined GROUP BY 1;
```

---

## 4) 实验设计与执行流程 🛠️
1. **预注册（Pre-registration）**：写清**假设、指标、触发口径、MDE、样本量、停表规则**。  
2. **分层/分桶**：按地区/设备/新老用户**分层随机化**，提升平衡与方差控制。  
3. **流量爬坡**：1% → 5% → 10% → 50% → 100%，每步通过**护栏阈值**再升。  
4. **并发实验**：用命名空间管理互斥，或在同一流量里做**分层互斥**。  
5. **AA Test**：定期（每月）做一次；SRM 报警、延迟/错误/曝光量**全部通过**才允许新实验。  
6. **冷启动与新奇效应**：设置**观察期**（如 3 天）避免刚上线的异常波动。  
7. **干扰与泄露**：社交/市场类产品注意**网络效应**与**跨用户干扰**（SUTVA 违背）；必要时做**地理分割**或**集群随机化**。

---

## 5) 统计推断（Frequentist / Bayesian）📐
### 5.1 核心选择
- **二项率**（转化率/留存率）：Z 检验 / Wilson 区间 / 贝叶斯 Beta-Binomial。  
- **均值型**（时长/ARPU）：t 检验（Welch）/ Bootstrap / 贝叶斯 Normal-Inverse-Gamma。  
- **分布差异**：Mann–Whitney U / KS；  
- **聚类方差**：按**用户聚合**后再比（Cluster-Robust SE）。

### 5.2 样本量与功效（Power）
- **MDE**（最小可检出改善）：由业务阈值决定；**不设 MDE 的实验是无底洞**。  
- 二项率近似样本量公式（双侧检验）：
```text
n_per_group ≈ 2 * (z_{1-α/2} + z_{power})^2 * p*(1-p) / (Δ^2)
# p = 预估基线转化率，Δ = 期望提升（绝对值，如 +0.5pp）
```

### 5.3 方差降低
- **CUPED**（协变量回归预消除）：用实验前的同口径变量作协变量，能显著降方差。  
- **分层/回归调整**：对设备、地区、历史行为做线性/广义线性调整。  

CUPED 伪代码：
```python
# y: 实验期指标（每用户），x: 实验前同口径指标
theta = cov(y, x) / var(x)
y_cuped = y - theta * x
# 再对 y_cuped 做 A/B 比较
```

### 5.4 顺序检验与“偷看”
- **频率学**：使用**群组序贯**（O’Brien–Fleming / Pocock）或 **α-spending** 控制一再查看的 I 型错误。  
- **始终有效 p 值**：mSPRT / e-value 框架。  
- **贝叶斯**天然支持顺序更新，但需预先定义**决策阈值**与**损失函数**。

### 5.5 多重检验
- 同时看很多指标/子群时，采用 **FDR（Benjamini–Hochberg）** 或 **Holm–Bonferroni** 控制假阳性。

---

## 6) 指标计算细节（别被分母骗）🧮
- **分子分母一致性**：如“每触发用户的支付率”，分母必须是**触发用户数**。  
- **比率的置信区间**：比率=分子/分母时，优先**Delta Method**或**Bootstrap**。  
- **长尾数值**：看**分位数**，配合 **Hodges–Lehmann** 中位数差估计。  
- **留存**：采用**队列（cohort）**与**固定窗**，避免滚动窗把分母越滚越大。  

Bootstrap 置信区间（伪代码）：
```python
import numpy as np
def boot_ci(a, b, stat, B=5000, alpha=0.05):
    diffs = []
    n, m = len(a), len(b)
    for _ in range(B):
        s1 = np.random.choice(a, n, True)
        s2 = np.random.choice(b, m, True)
        diffs.append(stat(s2) - stat(s1))
    return (np.percentile(diffs, 100*alpha/2), np.percentile(diffs, 100*(1-alpha/2)))
# stat = np.mean / 自定义比率函数
```

---

## 7) 决策与发布（从统计到策略）🧑‍⚖️
- **判定规则**（例）：  
  - 北极星或主要驱动 **显著向好**（≥MDE），护栏**不恶化**；  
  - 若护栏恶化，**回滚或重设策略**（如节流、灰度）。  
- **异质性效应**：若整体无差，但在关键子群（新用户/特定渠道）明显向好，可**定向发布**。  
- **代价与收益**：估算 **年化增益** 与 **技术/运营成本**，统计显著 ≠ 经济合理。  
- **结果沉淀**：记录**假设→结果→可复用启发**到知识库，避免“轮子 A/B 再发一次”。

---

## 8) 何时用 Bandit（多臂老虎机）🎰
- **短生命周期页位**、实时竞价/推荐位、沉没成本高的探索。  
- 选型：  
  - **ε-greedy / UCB / Thompson Sampling**；  
  - **上下文 Bandit** 适合有高维特征、想个性化分配的场景。  
- 注意：Bandit **优化在线收益**，但**推断整体效应较弱**；关键策略仍建议 A/B 离线确认。

---

## 9) 工具与流水线（参考架构）🧰
- **实验服务**：分桶/命名空间/粘性/爬坡控制/SRM 报警。  
- **指标层**：统一在仓库（dbt/metrics layer）定义指标 SQL + 单元测试。  
- **看板**：转化/留存/收入/护栏（延迟、崩溃、错误率） + SRM/AA。  
- **告警**：SRM、护栏越界、爬坡前置检查未通过→自动阻断。

示例：SRM 检查（卡方）
```sql
WITH c AS (
  SELECT variant, COUNT(DISTINCT unit_id) AS n
  FROM exposures
  WHERE exp_name = 'checkout_tweak' AND ts BETWEEN :start AND :end
  GROUP BY 1
),
t AS (
  SELECT SUM(n) AS total FROM c
),
e AS (
  SELECT c.variant, c.n,
         (t.total / (SELECT COUNT(*) FROM c))::float AS expected
  FROM c CROSS JOIN t
)
SELECT SUM( (n-expected)*(n-expected) / NULLIF(expected,0) ) AS chi2
FROM e;
-- 与 df=k-1 的卡方分布比较，得到 p 值
```

---

## 10) 反模式与纠偏 🧨
| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 未定义指标就开测 | 结果难解释 | 先写**指标口径/触发/MDE**，预注册 |
| 不做 AA 与 SRM | 一上来就假阳性/假阴性 | 上线前做 **AA**，持续监控 **SRM** |
| 分桶不粘 | 用户变体来回跳 | **hash 粘性 + 冻结秘钥** |
| 频繁偷看 | 显著性虚高 | **序贯设计/α-spending** 或贝叶斯阈值 |
| 多指标都看 | 总有一个显著 | **FDR/多重检验控制** |
| 用会话当用户 | 重度用户放大会话数 | **优先用户为单位**，或做聚类稳健 SE |
| 分母不一致 | Rate 被稀释 | **触发口径** + 分子分母一致 |
| 只看均值 | 长尾误导 | **分位数/鲁棒估计** |
| 忽略护栏 | 收益里掺毒 | 按护栏阈值**硬阻断爬坡** |

---

## 11) 验收清单 ✅
- [ ] 指标层：北极星/驱动/护栏**有口径与审计 SQL**。  
- [ ] 实验服务：**粘性分桶**、命名空间、爬坡、曝光日志强一致。  
- [ ] 质量：AA 测试通过、SRM 监控上线、身份并轨规则固定。  
- [ ] 统计：**MDE/样本量**、**序贯或固定观测窗**、**多重检验**策略确定。  
- [ ] 方差控制：启用 **CUPED/分层/回归调整**。  
- [ ] 看板：主指标 + 护栏 + SRM + AA；报警能阻断爬坡。  
- [ ] 复盘：决策记录与可复用启发沉淀。

---

## 12) 练习 🏋️
1. 为“搜索提示”功能写**预注册**：假设、触发、指标、MDE、样本量、停表；并生成审计 SQL。  
2. 跑一次 **AA 测试**，实现 SRM 实时告警（卡方 p<0.001 报警）。  
3. 用 **CUPED** 给“次日留存率”降方差，比较 CI 宽度。  
4. 设计一次 **Bandit → A/B** 的两阶段实验：先在线 Bandit 探索，再用 A/B 固化策略并量化整体效应。

---

**小结**：A/B 的价值不只在于“统计显著”，而在于**定义清晰的指标体系 + 高质量随机化 + 低方差估计 + 可复用的决策流程**。把这四件事做稳，实验结果才配得上被上线。🚀
