# 32.3 电商 / 支付 / 风控（真实可上线版）🛒💳🛡️

> 目标：做一套**抗高并发、可观测、能灰度、可回滚**的交易闭环，从 **下单 → 支付 → 履约/发货 → 退款/对账 → 风控** 全链路跑通。强调**一致性**、**幂等**、**签名校验**、**库存保护** 与 **风控分层**。  
> 读完你能落地：**模型、流程、状态机、接口、作业** + 能抄的代码骨架。

---

## 0）业务 SLO & 真相指标

- **下单 P50 < 150ms / P99 < 800ms**（同区），**支付确认** ≤ 5s（卡组织/三方通道除外）。  
- **库存一致性**：无超卖（或可控 ≤ 1ppm 且能补差）。  
- **资金安全**：支付回调**100% 验签** + **幂等**；对账差异 **<万分之一**。  
- **风控**：拦截率、误杀率、被盗卡退款率（CBR），三者设目标并 A/B 受控。

---

## 1）系统总图（服务划分与事件流）

```
Client/H5/小程序
   │  下单/支付/查询
   ▼
API Gateway ──► Auth / Permission / RateLimit
   │
   ├─► Checkout(下单) ──► Pricing(价税促) ─► Inventory(库存)
   │         │                   │                │ 保留/释放
   │         └─► Risk(下单前初审)│                ▼
   │                              └─► Coupon/Points(优惠抵扣)
   │
   ├─► Order(订单域) ──► Payment(支付域) ──► PG 适配(Alipay/WX/Stripe/等)
   │         │              │                        ▲
   │         │              └─ 回调 Webhook ──────────┘ 验签+幂等
   │         └─► Fulfillment(履约/发货) ─► WMS/快递
   │
   ├─► Refund/AfterSale(售后) ──► Payment 退款 ──► PG
   ├─► Reconciliation(对账引擎) ──► Ledger(资金账) ──► DataLake
   └─► Notifications(站内/短信/邮件/小程序订阅)

Event Bus(Kafka/NATS): order.created / payment.succeeded / inventory.reserved / refund.completed ...
Outbox & Saga 确保可靠投递 / 跨域一致性
```

---

## 2）核心域模型（简表）

- **SKU/Stock**：`sku(id, title, price, tax_class)`；`stock(sku_id, total, reserved)`.
- **Order**：`order(id, user_id, amount_total, amount_payable, status)`；`order_item(order_id, sku_id, qty, price, promo_id?)`.
- **Payment**：`payment(id, order_id, channel, amount, state, idempotency_key, provider_txn_id)`.
- **Refund**：`refund(id, order_id, payment_id, amount, state, reason)`.
- **Ledger**：`ledger_entry(id, account, amount, ref_type, ref_id, direction, ts)`.
- **Risk**：`risk_case(id, order_id, user_id, score, decision, features)`。
- **Inventory Reservation**：`inv_reservation(id, sku_id, qty, order_id, ttl, state)`。

> **幂等键**：`idempotency_key` （每订单每动作唯一）+ DB 唯一键，所有外部回调/重试都走它。

---

## 3）状态机（订单 / 支付 / 退款）

### 3.1 订单状态机

```
CREATING → PENDING_PAYMENT → PAID → FULFILLING → FULFILLED → DONE
     ↘︎ (风控拒绝) REJECTED
PENDING_PAYMENT ↔ CANCELLED (超时/用户取消)
PAID → CANCELLED_AFTER_PAY (失败补偿/人工) → 退款流
```

### 3.2 支付状态机

```
INIT → PROCESSING → SUCCEEDED
           ↘︎ FAILED (可重试/换通道) 
```

### 3.3 退款状态机

```
REQUESTED → PROCESSING → SUCCEEDED
                   ↘︎ FAILED (可再次发起/人工)
```

---

## 4）下单流程（含库存保护与风控）

**理想路径**

```
客户端 → /checkout/preview  (计算价税促)
       → /orders/create     (生成订单，预占库存) 
       → Risk.preAuth       (低阻塞初审：黑名单/设备/地址/频率)
       → /payments/create   (统一支付下单，返回支付凭证/跳转)
       → 用户完成支付
       ← Webhook 回调 → Payment 验签 + 幂等 → 更新订单为 PAID → 释放预占 → 锁定占用
       → Fulfillment 创建出库单/发货
```

**库存保护**  
- 使用 **“预留-确认”二阶段**：  
  1) `inv_reservation` 写入（`state=reserved, ttl=15m`），`stock.reserved += qty`；  
  2) 支付成功后 **确认**：将预留转正（扣减 `total`、归零 `reserved`），或超时自动释放。  
- 热点 SKU 用 Redis 原子脚本/分布锁 + DB 冷热一致性校验。  
- 大促**限购**与**扣减策略**（如“先到先得/按支付成功扣减”）需提前定调，避免“已下单未支付占坑太久”。

---

## 5）支付域：统一适配与回调幂等

### 5.1 Channel 抽象

```ts
// payment/provider.ts
export interface PayProvider {
  name: 'alipay'|'wechat'|'stripe'|'...';
  create(request: {
    orderId: string; amount: number; currency: string; 
    subject: string; returnUrl?: string; notifyUrl: string; 
    meta?: Record<string, any>;
  }): Promise<{ clientSecret?: string; qrCodeUrl?: string; payUrl?: string }>;

  // Webhook 验签 & 解析标准事件
  verifyAndParse(rawBody: Buffer, headers: Record<string,string>): Promise<{
    ok: boolean; event: 'payment_succeeded'|'payment_failed'|'refund_succeeded'|...;
    providerTxnId: string; amount: number; orderId: string; signatureValid: boolean;
  }>;
}
```

### 5.2 Webhook 处理（**签名 + 幂等 + 事务**）

```ts
// routes/payment-webhook.ts
app.post('/webhook/:channel', rawBody(), async (req, res) => {
  const provider = pickProvider(req.params.channel);
  const evt = await provider.verifyAndParse(req.body, req.headers as any);
  if (!evt.ok || !evt.signatureValid) return res.status(400).end('invalid');

  // 幂等：idempotency_key = providerTxnId + event
  const idemKey = `${evt.providerTxnId}:${evt.event}`;
  const already = await db.idempotency.findUnique({ where: { key: idemKey } });
  if (already) return res.status(200).end('ok');

  await db.$transaction(async (tx) => {
    await tx.idempotency.create({ data: { key: idemKey } });

    if (evt.event === 'payment_succeeded') {
      const pay = await tx.payment.update({
        where: { orderId: evt.orderId },
        data: { state: 'SUCCEEDED', providerTxnId: evt.providerTxnId, amount: evt.amount }
      });
      await tx.order.update({ where: { id: evt.orderId }, data: { status: 'PAID' } });
      // 确认库存预留
      await tx.$executeRaw`SELECT confirm_reservation(${evt.orderId})`;
      // 记资金账
      await tx.ledger_entry.create({
        data: { account: 'receivable', amount: evt.amount, ref_type: 'payment', ref_id: pay.id, direction: 'credit' }
      });
      // 发布事件
      await bus.publish('payment.succeeded', { orderId: evt.orderId });
    }

    if (evt.event === 'payment_failed') {
      await tx.payment.update({ where: { orderId: evt.orderId }, data: { state: 'FAILED' } });
    }
  });

  res.status(200).end('ok');
});
```

> **验签**：  
> - 支付宝/微信：公钥/平台证书链校验 + 时间窗。  
> - Stripe：`Stripe-Signature` header + secret。  
> - 所有回调**禁止走业务反向查询替代验签**（防中间人伪造）。  

### 5.3 3DS / 风控挑战 / 分期

- 卡支付可引入 **3-D Secure**（强身份认证），或按风控分层触发 **挑战**（验证码、短信、人机）。  
- 大额订单支持 **预授权（auth）→ capture**；可做**部分 capture**（分批发货）。  
- 多货币：内部记账用**清算币种**统一；展示用用户币种。

---

## 6）对账与资金账（钱要算得明白）

- **三账对齐**：**订单账**（订单/退款）、**支付账**（PG 回调/对账单）、**资金账**（Ledger）。  
- **日对账作业**：拉取通道对账单 → 归档 → 匹配差异（少单/多单/金额不符） → 生成差异工单。  
- **费用**：通道费率、退款费、结算延迟；生成 **P&L** 报表。  

```ts
// reconcile.ts (伪)
for (const row of providerStatement) {
  const local = await db.payment.findUnique({ where: { providerTxnId: row.txn } });
  if (!local || local.amount !== row.amount || local.state !== map(row.status)) {
    await db.recon_issue.create({ data: { provider: 'alipay', txn: row.txn, kind: diffKind(local, row) } });
  }
}
```

---

## 7）风控（规则 + 模型 + 人工）🛡️

### 7.1 数据与特征

- **账户**：注册时长、下单频率、退货率、历史风控记录。  
- **设备/网络**：设备指纹、IP ASN、代理/VPN、地理/时区偏差。  
- **支付**：卡 BIN、国家、历史失败/拒付、是否 3DS、姓名/地址/电话一致性。  
- **收货**：地址黑名单、团伙地址、异常收件人画像。  
- **行为**：页面停留、填表速度、改价/切换地址次数、优惠券使用轨迹。  

### 7.2 决策层级（实时）

```
score < T1  → Accept（低风险）
T1 ≤ score < T2 → Challenge（验证码/3DS/人工复核）
score ≥ T2 → Reject（拒绝下单/支付）
```

> **最佳实践**：  
> - 下单前做**轻审**（挡住明显垃圾/羊毛），支付前后做**重审**（金额/风险升维）。  
> - **风控不可影响幂等**：拒绝/挑战的结果也要落库并可回放。  
> - **促销风控**：券码限设备/账号/支付工具；多账号同设备速率限制；防薅毛脚本。  

### 7.3 规则 + 模型融合

```ts
// risk/engine.ts (极简伪代码)
export function riskScore(ctx: OrderContext): { score: number; reasons: string[] } {
  let s = 0; const r: string[] = [];
  if (ctx.user.ageDays < 1) s += 20, r.push('new_user');
  if (ctx.ip.isProxy) s += 25, r.push('proxy');
  if (ctx.addr.isBlack) s += 80, r.push('address_black');
  if (ctx.payment.cardCountry !== ctx.ip.country) s += 30, r.push('geo_mismatch');
  s += modelPredict(ctx.features) * 50; // 0..1 映射
  return { score: s, reasons: r };
}
```

- **名单系统**：黑/灰/白名单优先级 > 规则 > 模型。  
- **图谱**：手机号/设备/收货地址/支付工具构图，识别团伙（连通子图异常密度）。  
- **人工复核**：抽样 & 高风险必审；**SLA < 30min**；审单结果反哺训练。

---

## 8）一致性与可靠性（Saga / Outbox / 幂等）

- **Saga**（示例：下单）  
  1) 创建订单 `CREATING`  
  2) 预占库存（失败→补偿：取消订单）  
  3) 生成支付单（失败→释放预占 + 取消订单）  
  4) 回调成功→确认库存 + 订单 `PAID`  
  5) 超时→释放预占 + 订单 `CANCELLED`  

- **Outbox**：写入同事务中记录事件，后台可靠投递到 Bus，防丢。  
- **幂等**：下单/支付/退款接口统一 `Idempotency-Key`；Webhook 用 `providerTxnId`。  
- **重试**：指数退避 + 上限 + 死信队列；幂等键保证“至多一次副作用”。

---

## 9）安全与合规

- **Webhook 安全**：IP 白名单 + 签名 + 时效；**原始 body** 验签；失败返回 5xx（让通道重试）。  
- **CSRF**：支付发起用 `POST + CSRF Token`；回跳页校验 `state`。  
- **Secrets**：KMS 托管；密钥滚动；不同通道不同凭据。  
- **PCI DSS & 隐私**：尽量 **Tokenization**，不落原卡号，前端直连通道；个人信息做**脱敏/最小化**。  
- **合规**：GDPR/CCPA 导出/删除；发票/税务合规（按国家地区配置）。  

---

## 10）价格 / 税 / 促销（避免“赔本赚吆喝”）

- **可重复计算**：价税促引擎**纯函数**（输入 SKU、地区、等级、券码、时段），输出**明细**（基础、折扣、税）。  
- **券/积分**：幂等扣减；并发防重；**发券与核销**走 Outbox。  
- **价格签名**：前端下单带 `price_signature`（后端生成 HMAC），防被改价。

```ts
// pricing/sign.ts
export function signPricing(cartHash: string, ts: number) {
  return hmacSHA256(`${cartHash}:${ts}`, process.env.PRICING_SECRET);
}
```

---

## 11）观察与告警

- QPS、下单/支付/退款 P95/P99、库存失败率、风控拦截/误杀、Webhook 成功率、对账差异、通道可用性。  
- **业务漏斗**：进入 checkout → 下单 → 发起支付 → 支付完成 → 发货 → 完成。  
- **资金监控**：资金账余额、未结算余额、退款时延分布。  
- **灰度/回滚**：PG 适配/风控策略/价格引擎都要支持版本化与逐流量放量。

---

## 12）DDL（简化版，可改 ORM）

```sql
CREATE TABLE orders (
  id              UUID PRIMARY KEY,
  user_id         UUID NOT NULL,
  status          TEXT NOT NULL,              -- CREATING/PENDING_PAYMENT/PAID/...
  amount_subtotal BIGINT NOT NULL,
  amount_discount BIGINT NOT NULL DEFAULT 0,
  amount_tax      BIGINT NOT NULL DEFAULT 0,
  amount_payable  BIGINT NOT NULL,
  currency        TEXT NOT NULL,
  created_at      TIMESTAMP NOT NULL DEFAULT now(),
  updated_at      TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE order_item (
  id UUID PRIMARY KEY,
  order_id UUID REFERENCES orders(id),
  sku_id UUID NOT NULL,
  qty INT NOT NULL,
  price BIGINT NOT NULL,                      -- 单价（分）
  promo_id UUID
);

CREATE TABLE payment (
  id UUID PRIMARY KEY,
  order_id UUID UNIQUE REFERENCES orders(id),
  channel TEXT NOT NULL,
  amount BIGINT NOT NULL,
  state TEXT NOT NULL,                        -- INIT/PROCESSING/SUCCEEDED/FAILED
  provider_txn_id TEXT,                       -- 唯一
  idempotency_key TEXT UNIQUE,
  created_at TIMESTAMP DEFAULT now()
);

CREATE UNIQUE INDEX uniq_provider_txn ON payment(provider_txn_id);

CREATE TABLE inv_reservation (
  id UUID PRIMARY KEY,
  order_id UUID NOT NULL,
  sku_id UUID NOT NULL,
  qty INT NOT NULL,
  ttl TIMESTAMP NOT NULL,
  state TEXT NOT NULL                         -- reserved/confirmed/released
);
```

---

## 13）库存预留存储（Redis + SQL 双写）

```ts
// inventory/reserve.ts
// Redis Lua 原子：检查余量 → incr reserved
-- KEYS: totalKey, reservedKey
-- ARGV: need
local total = tonumber(redis.call('GET', KEYS[1]) or '0')
local reserved = tonumber(redis.call('GET', KEYS[2]) or '0')
local need = tonumber(ARGV[1])
if total - reserved < need then return 0 end
redis.call('INCRBY', KEYS[2], need) return 1
```

- 成功后写 `inv_reservation`（DB），失败回滚 Redis。  
- 任务：**定时释放过期预留**（TTL 到期）。

---

## 14）退款流程（原路退回 + 对账）

```
用户/售后 → Refund.create(idem) → Payment.refund(channel)
 → 等待 PG 回调(verify + idem) → refund.succeeded → 记账/通知/库存恢复（如需）
```

- **部分退款**：跟随 item 或按金额分配；Ledger 记负向条目。  
- **超时/失败**：人工升级，保证账实相符。  

---

## 15）端到端示例（下单 → 支付创建）

```ts
// orders.controller.ts
export async function createOrder(req, res) {
  const idem = req.header('Idempotency-Key'); // 客户端生成
  const exist = await db.payment.findUnique({ where: { idempotency_key: idem } });
  if (exist) return res.json({ orderId: exist.orderId }); // 幂等返回

  await db.$transaction(async (tx) => {
    // 1) 校验购物车/价税促签名
    // 2) 风控（轻审）
    const order = await tx.orders.create({ data: { ...calc, status: 'PENDING_PAYMENT', user_id: req.user.id } });
    // 3) 预占库存（失败抛错）
    await reserveAll(tx, order.id, req.body.items);
    // 4) 创建支付单
    await tx.payment.create({ data: { id: crypto.randomUUID(), order_id: order.id, amount: calc.amount_payable, channel: req.body.channel, state: 'INIT', idempotency_key: idem } });
    // 5) 调起通道
  });

  const payInfo = await provider.create({ orderId, amount, currency: 'CNY', notifyUrl, subject });
  res.json({ orderId, pay: payInfo });
}
```

---

## 16）大促 & 灰度策略（保命指南）

- **影子流量**：提前开真实链路但不计费，压测与对账演练。  
- **读写隔离**：热点读走缓存（带失效）；写路径严控事务时长。  
- **开关**：限购开关/库存预留 TTL/支付通道权重/风控阈值配置中心化。  
- **降级**：促销页 → 静态化；推荐 → 兜底；支付 → 替换备通道；履约 → 延迟承诺。  

---

## 17）Checklist（上线前最后一眼）✅

- [ ] 订单/支付/退款状态机全覆盖，边界用例通过  
- [ ] Webhook **原始体**验签、**幂等**、失败可重试  
- [ ] 库存**预留-确认**两阶段，超时释放，压测无超卖  
- [ ] 价格签名/幂等键/限购/频率限制已开启  
- [ ] 风控分层：名单/规则/模型/人工，挑战路径验证  
- [ ] 对账作业跑通：差异工单 + 资金账一致  
- [ ] 日志/指标/告警：漏斗、P95、失败拓扑、通道可用性  
- [ ] 配置中心：通道切换、阈值、灰度、回滚  
- [ ] 演练：失败回调/重复回调/延迟回调/部分退款/强制回滚  

---

### 小结

电商交易闭环不是“能支付就行”，而是**一致性 + 安全 + 可观测**的系统工程：  
把**幂等与签名**作为底线，把**库存与对账**作为生命线，把**风控**嵌进每个关键节点。  
这样的大盘，遇上大促也能稳住阵脚，遇到异常也能**可解释、可修复、可回放**。🚀
