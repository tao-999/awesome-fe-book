# 19.1 错误上报 / 性能追踪（OpenTelemetry） 🧭🐛⚡

目标：在浏览器里把 **错误、性能、链路** 三件事一次性打通：  
- 统一 **Trace（链路） / Metrics（指标） / Logs（日志）** 三板斧  
- 首屏与交互 **Web Vitals**（LCP/CLS/INP/TBT）进指标  
- 错误自动上报并 **与 Trace 关联**（一键从报错跳到上下文链路）  
- **跨前后端**传递 `traceparent`，服务端链路“接得上”  
- 隐私合规 & 低开销 & 可观测到位

口号：**“错误是事件，性能是分布，链路是故事。”**

---

## 0) TL;DR（先抄再说）🎯

- 浏览器端用 **OpenTelemetry JS**：  
  - **Trace**：`@opentelemetry/sdk-trace-web` + `@opentelemetry/instrumentation-{fetch,xhr,document-load,user-interaction}`  
  - **Metrics**：`@opentelemetry/api` + `@opentelemetry/sdk-metrics`（直上 OTLP HTTP）  
  - **Logs（可选）**：`@opentelemetry/sdk-logs`  
  - **Exporter**：`@opentelemetry/exporter-{trace,metrics,logs}-otlp-http`
- Collector（或 APM 平台）接收 **OTLP/HTTP**，开启 **batch**，前端采样 ≤ **5%**，服务端可 **tail sampling**。
- 同步 **Web Vitals** 到 OTel Histogram，并将 API/FETCH 的 latency 归档；错误作为 **Span Event** + `status: ERROR`。
- **CORS & 安全**：Collector 开 CORS；后端允许 `traceparent` / `tracestate` / `baggage` 请求头；前端做 **PII 脱敏**。
- **Source Map**：给错误栈做符号化（在可控后端/平台端完成），别把源码暴露给公网。

---

## 1) 最小可跑骨架（浏览器端）🧱

> 适用于 Vite/Next/Nuxt 等现代打包；确保开启 **Corepack/锁版本**。

### 1.1 安装

```bash
pnpm add @opentelemetry/api \
  @opentelemetry/sdk-trace-web \
  @opentelemetry/sdk-metrics \
  @opentelemetry/sdk-logs \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions \
  @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/exporter-metrics-otlp-http \
  @opentelemetry/exporter-logs-otlp-http \
  @opentelemetry/instrumentation \
  @opentelemetry/instrumentation-document-load \
  @opentelemetry/instrumentation-user-interaction \
  @opentelemetry/instrumentation-fetch \
  @opentelemetry/instrumentation-xml-http-request
```

### 1.2 初始化（`otel.ts`）

```ts
// otel.ts
import { diag, DiagConsoleLogger, DiagLogLevel } from '@opentelemetry/api';
import { Resource } from '@opentelemetry/resources';
import { SEMRESATTRS_SERVICE_NAME, SEMRESATTRS_SERVICE_VERSION } from '@opentelemetry/semantic-conventions';
import { WebTracerProvider } from '@opentelemetry/sdk-trace-web';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { MeterProvider, PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-http';
import { LoggerProvider, BatchLogRecordProcessor } from '@opentelemetry/sdk-logs';
import { OTLPLogExporter } from '@opentelemetry/exporter-logs-otlp-http';

import { registerInstrumentations } from '@opentelemetry/instrumentation';
import { DocumentLoadInstrumentation } from '@opentelemetry/instrumentation-document-load';
import { UserInteractionInstrumentation } from '@opentelemetry/instrumentation-user-interaction';
import { FetchInstrumentation } from '@opentelemetry/instrumentation-fetch';
import { XMLHttpRequestInstrumentation } from '@opentelemetry/instrumentation-xml-http-request';

const isDev = import.meta.env.DEV;
diag.setLogger(new DiagConsoleLogger(), isDev ? DiagLogLevel.DEBUG : DiagLogLevel.ERROR);

// 资源标识
const resource = new Resource({
  [SEMRESATTRS_SERVICE_NAME]: 'awesome-fe-book-web',
  [SEMRESATTRS_SERVICE_VERSION]: __APP_VERSION__ || '0.0.0',
  deploymentEnvironment: import.meta.env.MODE,
});

// -------- Traces --------
const traceProvider = new WebTracerProvider({ resource });
traceProvider.addSpanProcessor(
  new BatchSpanProcessor(new OTLPTraceExporter({
    url: 'https://otel.example.com/v1/traces',
    headers: { 'x-api-key': import.meta.env.VITE_OTLP_KEY ?? '' },
  }), { maxExportBatchSize: 64, scheduledDelayMillis: 2_000 })
);
traceProvider.register(); // 注册全局 Tracer

// -------- Metrics --------
const meterProvider = new MeterProvider({ resource });
meterProvider.addMetricReader(new PeriodicExportingMetricReader({
  exporter: new OTLPMetricExporter({
    url: 'https://otel.example.com/v1/metrics',
    headers: { 'x-api-key': import.meta.env.VITE_OTLP_KEY ?? '' },
  }),
  exportIntervalMillis: 10_000,
}));

// -------- Logs（可选） --------
const loggerProvider = new LoggerProvider({ resource });
loggerProvider.addLogRecordProcessor(new BatchLogRecordProcessor(new OTLPLogExporter({
  url: 'https://otel.example.com/v1/logs',
  headers: { 'x-api-key': import.meta.env.VITE_OTLP_KEY ?? '' },
})));

// -------- 自动埋点 --------
registerInstrumentations({
  instrumentations: [
    new DocumentLoadInstrumentation(),
    new UserInteractionInstrumentation(),
    new FetchInstrumentation({
      propagateTraceHeaderCorsUrls: [/./], // 给所有跨域请求带 traceparent
      applyCustomAttributesOnSpan: (span, request, result) => {
        span.setAttribute('http.request.body_size', request.headers?.get('content-length') ?? 0);
        if (result) span.setAttribute('http.response.body_size', result.headers?.get('content-length') ?? 0);
      },
    }),
    new XMLHttpRequestInstrumentation(),
  ],
});
```

> 检查浏览器 **Network**：任一 API 请求应自动携带 `traceparent` 头。若服务端跨域，需允许该头通过 CORS（见 §4.3）。

---

## 2) 错误上报（统一到 Trace）🧨

### 2.1 捕获方式

- **自动**：`Fetch`/`XHR` 失败 → Span `status=ERROR`。  
- **脚本错误**：`error` / `unhandledrejection` → 记录为 **Span Event** + `exception.*` 语义属性。  
- **手动**：在当前活跃 Span 上 `.recordException()` / `.setStatus({ code: ERROR })`。

```ts
// errors.ts
import { context, trace, SpanStatusCode } from '@opentelemetry/api';

function activeSpan() {
  return trace.getSpan(context.active());
}

window.addEventListener('error', (e) => {
  const span = activeSpan();
  if (span) {
    span.recordException({ name: e.error?.name || 'Error', message: e.message, stack: e.error?.stack });
    span.setStatus({ code: SpanStatusCode.ERROR, message: 'window.onerror' });
  }
});

window.addEventListener('unhandledrejection', (e) => {
  const span = activeSpan();
  const reason = (e as PromiseRejectionEvent).reason;
  if (span) {
    span.recordException({ name: reason?.name || 'UnhandledRejection', message: String(reason), stack: reason?.stack });
    span.setStatus({ code: SpanStatusCode.ERROR, message: 'unhandledrejection' });
  }
});
```

> 建议：错误 **“去重键”** = `name + message + (top frame)`，附带 `release`/`service.version`，避免告警风暴。

---

## 3) 性能追踪（Web Vitals + API 延迟）⚡

### 3.1 采集 Web Vitals → Metrics

```ts
// vitals.ts
import { onLCP, onCLS, onINP, onTTFB } from 'web-vitals';
import { metrics } from '@opentelemetry/api';

const meter = metrics.getMeter('web-vitals');
const LCP_H = meter.createHistogram('web_vitals_lcp_ms', { unit: 'ms' });
const CLS_H = meter.createHistogram('web_vitals_cls');
const INP_H = meter.createHistogram('web_vitals_inp_ms', { unit: 'ms' });
const TTFB_H = meter.createHistogram('web_vitals_ttfb_ms', { unit: 'ms' });

onLCP((v) => LCP_H.record(v.value, { element: v.element?.tagName || 'unknown', url: location.pathname }));
onCLS((v) => CLS_H.record(v.value, { url: location.pathname }));
onINP((v) => INP_H.record(v.value, { url: location.pathname }));
onTTFB((v) => TTFB_H.record(v.value, { url: location.pathname }));
```

### 3.2 API 延迟、失败率 → Metrics

```ts
// 可在 FetchInstrumentation 的 applyCustomAttributesOnSpan 里加：
// 同时我们也创建一个直方图用于可视化分布
import { metrics } from '@opentelemetry/api';
const meter = metrics.getMeter('api');
const HTTP_LATENCY = meter.createHistogram('http_client_duration_ms', { unit: 'ms' });
const HTTP_ERRORS = meter.createCounter('http_client_errors_total');

export function onHttpSpanEnd(span: any) {
  const dur = Number(span.duration?.[0] ?? 0) / 1e6; // ns -> ms（不同SDK实现可直接取）
  const attrs = span.attributes || {};
  HTTP_LATENCY.record(dur, { method: attrs['http.request.method'], route: attrs['http.route'] || attrs['http.url'] });
  if (attrs['http.response.status_code'] >= 400) {
    HTTP_ERRORS.add(1, { code: String(attrs['http.response.status_code']), route: attrs['http.route'] || 'unknown' });
  }
}
```

> 也可以直接在 Collector 侧把 **Span 转 Metrics**（Latency/Apdex/错误率），减少前端代码复杂度。

---

## 4) 端到端追踪（把链路接起来）🔗

### 4.1 传播与采样

- 浏览器端默认 **W3C Trace Context**，自动发送 `traceparent` / `tracestate`。  
- **前端建议 head-sampling ≤ 5%**；服务端可 **tail-sampling**（错误优先、慢请求优先）。  
- 采样策略应可 **动态下发**（feature flag / 远端配置）。

### 4.2 服务端接续

- 任意 OTel 服务端（Node/Go/Java 等）会自动从请求头取 `traceparent` 创建子 Span。  
- 你可以把 `trace_id` 注入到日志 MDC/labels，log→trace 一键互跳。

### 4.3 CORS 配置（关键）

- 前端发 `traceparent` 属于自定义头，跨域时需允许：  
  - `Access-Control-Allow-Headers: traceparent, tracestate, baggage`  
  - `Access-Control-Expose-Headers: traceparent`（可选）  
- Collector（若直连）需开启 **CORS 扩展**（见 §5 YAML）。

---

## 5) OpenTelemetry Collector（最小配置）🛠️

```yaml
# collector.yaml
extensions:
  health_check: {}
  pprof: {}
  zpages: {}
  cors:
    allowed_origins: ["https://your-website.example.com"]
    allowed_headers: ["*"]
    max_age: 7200

receivers:
  otlp:
    protocols:
      http: {}  # 前端走 OTLP/HTTP

processors:
  batch: {}
  probabilistic_sampler:
    sampling_percentage: 5  # 前端再采样一层（可选）
  attributes/scrub:
    actions:
      - key: http.request.header.authorization
        action: delete
      - key: user.email
        action: delete

exporters:
  otlp:
    endpoint: tempo:4317    # 例：发到 Tempo/Jaeger-OTLP 网关（gRPC）；或改 http(s)
    tls:
      insecure: true
  logging:
    loglevel: warn

service:
  extensions: [health_check, pprof, zpages, cors]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [attributes/scrub, probabilistic_sampler, batch]
      exporters: [otlp, logging]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
```

> 生产建议：在后端或 Collector 做 **tail_sampling**（按错误、服务、路由采样），更省流量、更聚焦。

---

## 6) 隐私与安全（必须做）🔒

- **PII 脱敏**：Span 属性过滤规则（如 `password/token/authorization/email/phone`），一律不发送。  
- **URL 规范化**：把 `/users/123` → `/users/{id}`，避免在 `http.url` 里泄露标识。  
- **Baggage 慎用**：不要把个人数据放进 `baggage`。  
- **Source Map 符号化**：  
  - 构建产出 source map，但**不要公网暴露**；上传到你的 APM/符号化服务；  
  - 导出时附 `service.version`，服务端用它定位正确的 map。  
- **CSP/nonce**：对第三方 SDK 控制白名单，防止篡改。

---

## 7) 与框架/平台对接 🧩

- **Next.js**：在 `pages/_app.tsx` 或 `app/layout.tsx` 引入 `otel.ts`；SSR 侧另配服务端 OTel。  
- **Nuxt**：插件机制 `plugins/otel.client.ts`；SSR 侧 `server/plugins/otel.server.ts`。  
- **Vite**：入口 `main.ts` 首行导入；生产只在允许的环境变量下启用。  
- **Edge/Service Worker**：SW 请求转发需手动把 `traceparent` 带过去（大多框架已支持）。

---

## 8) 仪表与看板（你该看什么）📊

- **RUM 总览**：LCP/INP/CLS/TTFB p75；慢/重资源前十，设备/地域分布。  
- **API 体验**：`http_client_duration_ms` p50/p90/p99；错误率（4xx/5xx）与路由 TopN。  
- **错误聚类**：按 `error.type + message + topframe`；关联 **Trace 链路**与 **最近发布版本**。  
- **链路拓扑**：前端 → BFF → 微服务 → 数据库；慢点分解。  
- **采样/丢弃率**：确保热点有样本，冷门不浪费。

---

## 9) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 只上报 JS 错误，不连 Trace | 错误看得到，根因看不到 | 错误→Span Event，带 `trace_id` |
| 所有数据全上 | 成本爆炸、隐私风险 | 采样 + 属性过滤 + URL 规范化 |
| 直接公网暴露 source map | 代码泄露 | map 仅上传到受控符号化服务 |
| 忽略 CORS | 前端 traceparent 被拦 | 允许 `traceparent,tracestate,baggage` |
| 指标只看均值 | 被长尾欺骗 | 看 p75/p90/p99 + 分位看板 |
| 仅 head sampling | 错误/慢请求没样本 | 后端加 **tail sampling**（错误/慢请求保留） |

---

## 10) 验收清单 ✅

- [ ] `traceparent` 已随 `fetch/xhr` 发出，服务端链路**接得上**。  
- [ ] JS 错误 / UnhandledRejection 自动记录为 **Span Event**，`status=ERROR`。  
- [ ] LCP/INP/CLS/TTFB 已入 **Histogram**，API 延迟/错误率有指标。  
- [ ] 采样策略：前端 ≤5%，服务端 **tail sampling** 覆盖错误与慢请求。  
- [ ] PII 过滤、URL 规范化、CORS 允许、Source Map 非公网。  
- [ ] 看板：RUM、API、错误聚类、拓扑、发布对齐。  

---

## 11) 练习 🏋️

1. 在你的站点引入 `otel.ts` + `vitals.ts`，验证 `traceparent` 贯通到 BFF/后端。  
2. 制作“首屏仪表”：LCP p75、首屏 API p90、JS 错误数；一键跳转到 Trace。  
3. 在 Collector 加 `tail_sampling`，策略：**错误 100%**、**>2s 请求 20%**、其余 5%。  
4. 做一次“发布回归”：发布前后对比 LCP/INP 与错误聚类变化，写复盘。  

---

**小结**：用 OpenTelemetry 把 **错误→Trace**、**性能→Metrics**、**日志→上下文** 织成一张网，才能从“哪里慢/哪里崩”追到“为什么”。把采样、隐私、CORS、符号化和看板一次做好，你的前端可观测性就不再是玄学。🚀
