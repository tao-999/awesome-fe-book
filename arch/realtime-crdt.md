# 31.1 WebSocket / SSE / WebRTC / CRDT（Yjs）📡🫧🔀

> 一句话心法：  
> **SSE**=单向、轻量广播；**WebSocket**=低开销双工管道；**WebRTC**=端到端（含媒体/数据）& NAT 穿透；**Yjs**=把“协作冲突”变成“无冲突合并”。实际工程常用**组合拳**：SSE 做服务端事件、WS 做信令/房间、WebRTC 承载 P2P 数据/音视频、Yjs 做状态一致性与离线回放。

---

## 0）场景选型速查 🧭

| 诉求 | SSE | WebSocket | WebRTC（DataChannel/Media） | 备注 |
|---|---|---|---|---|
| 方向 | 服务器 → 客户端 | 双向 | 双向（端↔端） | |
| 建立/兼容 | 最容易，纯 HTTP | 较易（升级为 WS） | 难（需信令+STUN/TURN） | |
| 代理/CDN 友好 | ✅（但要关缓冲） | ✅ | ❌（大多直连） | |
| 顺序/可靠 | 有序，需自管重试 | 有序，应用层自定 ACK/重发 | 可设有序/无序、（不）可靠 | |
| 带宽/延迟 | 低，文本 | 低，文本/二进制 | 极低，二进制/媒体 | |
| 典型用法 | 服务器事件流、日志、进度 | 聊天、房间、热数据通道 | 语音视频、白板低延迟游标 | |
| 成本 | 最低 | 低 | TURN 时高 | |

> 经验法则：**先 SSE**（只需下行），**够不到就 WS**（需要上行/房间），**有端到端/媒体就上 WebRTC**；多人协作的状态一致性交给 **Yjs**。

---

## 1）SSE：单向流最顺滑 🚰

### 1.1 最小可用（Node/Edge 都能写）
```ts
// /api/sse.ts (Express 风格)
export function sse(req, res) {
  res.set({
    'Content-Type': 'text/event-stream; charset=utf-8',
    'Cache-Control': 'no-store',
    'Connection': 'keep-alive',
  });
  res.flushHeaders();
  const send = (event: string, data: any) =>
    res.write(`event: ${event}\ndata: ${JSON.stringify(data)}\n\n`);

  const itv = setInterval(() => send('tick', { t: Date.now() }), 15000); // 心跳
  send('open', { ok: true });

  req.on('close', () => { clearInterval(itv); res.end(); });
}
```

**客户端**
```ts
const es = new EventSource('/api/sse');
es.addEventListener('tick', e => console.log(JSON.parse((e as MessageEvent).data)));
```

**避坑**
- 代理需关缓冲：Nginx `proxy_buffering off;`，Cloudflare 打开“实时流”。
- Safari 会合并小块；定期心跳保持连接活性。
- 断线重连利用 `Last-Event-ID`（你需要 `id:` 字段 + 服务端记忆）。

---

## 2）WebSocket：双工房间总线 🧵

### 2.1 最小房间服（ws）
```ts
// ws-server.ts
import { WebSocketServer, WebSocket } from 'ws';
import jwt from 'jsonwebtoken';

type Client = WebSocket & { uid?: string; room?: string };
const wss = new WebSocketServer({ port: 8787 });

wss.on('connection', (ws: Client, req) => {
  // 认证（示例：URL ?token=）
  try {
    const token = new URL(req.url!, 'http://x').searchParams.get('token')!;
    const { uid } = jwt.verify(token, 'secret') as any;
    ws.uid = String(uid);
  } catch { return ws.close(4401, 'unauthorized'); }

  ws.on('message', (raw) => {
    // 简易协议：{t:'join'|'pub', room, data}
    let msg; try { msg = JSON.parse(String(raw)); } catch { return; }
    if (msg.t === 'join') ws.room = msg.room;
    if (msg.t === 'pub') {
      // 背压控制
      for (const client of wss.clients) {
        const c = client as Client;
        if (c.readyState === WebSocket.OPEN && c.room === ws.room) {
          if (c.bufferedAmount < 1_000_000) c.send(JSON.stringify({ from: ws.uid, data: msg.data }));
        }
      }
    }
  });

  // 心跳
  const ping = setInterval(() => { if (ws.readyState===WebSocket.OPEN) ws.ping(); }, 15000);
  ws.on('close', () => clearInterval(ping));
});
```

**客户端重连与背压**
```ts
function connect(url: string, onMsg: (d:any)=>void) {
  let ws = new WebSocket(url);
  let retry = 500;
  ws.onopen = () => (retry = 500);
  ws.onmessage = e => onMsg(JSON.parse(e.data));
  ws.onclose = () => setTimeout(() => connect(url, onMsg), Math.min(8000, retry*=2));
  // 发送时注意 bufferedAmount
  const send = (o:any) => ws.readyState===1 && ws.bufferedAmount<1e6 && ws.send(JSON.stringify(o));
  return { send };
}
```

**扩展**
- 水平扩展用 **Redis/NATS PubSub** 做房间广播；或上 **云原生 WS 网关**（ELB Sticky / Cloudflare Durable Objects）。
- 业务协议：强烈建议统一 **envelope**（`type/room/seq/payload`），便于路由与观测。

---

## 3）WebRTC：端到端传输（含 DataChannel）📺🕹️

> WebRTC 需要**信令**（可用你上面的 WS）、**STUN**（拿公网候选）、**TURN**（打洞失败时中继，成本高）。

### 3.1 DataChannel 最小骨架（点对点文本/游标）
```ts
// signaling: 用 WebSocket 交换 offer/answer/ice
const pc = new RTCPeerConnection({ iceServers: [{ urls: ['stun:stun.l.google.com:19302'] }] });
const dc = pc.createDataChannel('chat', { ordered: true, maxRetransmits: 3 });

dc.onmessage = (e) => console.log('peer says:', e.data);
dc.onopen = () => dc.send('hello');

pc.onicecandidate = (e) => ws.send(JSON.stringify({ t:'ice', cand: e.candidate }));
ws.onmessage = async (e) => {
  const msg = JSON.parse(e.data);
  if (msg.t==='offer') { await pc.setRemoteDescription(msg.sdp); await pc.setLocalDescription(await pc.createAnswer()); ws.send(JSON.stringify({ t:'answer', sdp: pc.localDescription })); }
  if (msg.t==='answer') await pc.setRemoteDescription(msg.sdp);
  if (msg.t==='ice' && msg.cand) await pc.addIceCandidate(msg.cand);
};

// 主叫
(async () => {
  await pc.setLocalDescription(await pc.createOffer());
  ws.send(JSON.stringify({ t:'offer', sdp: pc.localDescription }));
})();
```

**要点**
- **多人**建议用 **SFU**（如 mediasoup/livekit/janus）转发媒体；DataChannel 也可走 SFU 的数据中转。
- **可靠性/顺序**：`ordered: false, maxRetransmits: 0` 适合高频游标；文档内容走可靠通道。
- **安全**：天然 DTLS-SRTP 加密；TURN 凭证设置**短 TTL**。

---

## 4）CRDT with **Yjs**：把“冲突”变“合并”🧩

> **CRDT**（Conflict-free Replicated Data Type）在每个副本本地编辑后**最终一致**；Yjs 提供 **Y.Text / Y.Array / Y.Map**、高效二进制更新 `Y.Update`、**Awareness**（在线状态/光标）等。

### 4.1 前端最小协作（y-websocket + IndexedDB）
```ts
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';
import { IndexeddbPersistence } from 'y-indexeddb';

// 1) 文档与持久化
export const ydoc = new Y.Doc();
new IndexeddbPersistence('doc-article-42', ydoc); // 离线缓存

// 2) 连接到房间
const provider = new WebsocketProvider('wss://your-yws.example.com', 'room-article-42', ydoc, { connect: true });

// 3) 协作文本
export const ytext = ydoc.getText('content');
ytext.observe(e => { /* 把 ytext.toString() 映射到编辑器 */ });

// 4) Awareness（在线状态/光标）
const aw = provider.awareness;
aw.setLocalStateField('user', { id: 'u123', name: '骚哥', color: '#6cf' });
aw.on('change', (_, changes) => { /* changes.added/updated/removed */ });
```

### 4.2 P2P 变体（y-webrtc）
```ts
import { WebrtcProvider } from 'y-webrtc';
const rtc = new WebrtcProvider('room-42', ydoc, { signaling: ['wss://signal.example.com'], password: undefined });
```
> 说明：小房间可 P2P，大房间/公司内网/防火墙环境用 **y-websocket** 更稳定；也可两者并用（P2P 优先，失败回退 WS）。

### 4.3 服务端持久化（可选）
- 官方 `y-websocket` 服务可接 **leveldb**、**redis** 存储 update；或你在服务端消费 `update` 做**快照**与**增量归档**。  
- 生产建议：定期把 `ydoc.encodeStateAsUpdate()` 存档，重放 update 前先套用最新快照。

### 4.4 与编辑器集成
- ProseMirror / TipTap / CodeMirror / Slate 都有社区绑定（y-prosemirror / y-codemirror）。
- **渲染优化**：用增量 patch，不要全量 `toString()` ；大文档开启**分段渲染/虚拟列表**。

---

## 5）组合架构蓝图（能落地的几套）🧬

### A. 协作文档（文本+游标+评论）
```
Editor UI  ←→  Yjs Doc (IndexedDB)  ←→  y-websocket
                         ↑ (awareness)
```
- 文本/评论走 **Yjs**；在线状态/光标用 **Awareness**；只读订阅用 SSE 推系统事件（如“某人申请编辑”）。

### B. 白板/脑图（低延迟轨迹）
- 形状/属性走 Yjs；**手绘轨迹**走 WebRTC DataChannel（无序/不可靠更丝滑）；回放记录周期性固化为帧。

### C. 直播带货聊天室
- 文本聊天/弹幕走 **WS**；公示榜单/库存广播走 **SSE**；连麦视频走 **WebRTC + SFU**。

---

## 6）一致性与存储策略 🧪

- **Yjs 一致性**：最终一致；单编辑器局部立即一致。冲突自动合并（按时序/因果）；你只关注**业务约束**（如字数/配额）。
- **快照与回滚**：每 N 分钟固化 `Y.Doc`；`diffUpdate` 做“版本对比”；误操作可回滚至最近快照并重播后续 update（或开分支）。
- **审计**：保留 **update 流** 的 hash/作者/时间戳（不要存明文思维链），支持追责与回放。

---

## 7）稳定性与可观测 📈

- **心跳**：WS `ping/pong`、SSE 心跳注释行、WebRTC `getStats()` 监控丢包 RTT。
- **背压**：检查 `ws.bufferedAmount`，超过限额丢旧/降采样；DataChannel 同理 `bufferedAmountLowThreshold`。
- **重连**：指数退避 + 抖动；SSE 自带 `retry:`，但你最好仍做应用层“继续点位”（事件 id）。
- **指标**：连接数、平均 RTT、房间 fanout 耗时、重连率、TURN 占比与带宽成本。

---

## 8）安全与权限 🛡️

- **鉴权**：连接前签发 **短期 JWT**；WS/SSE 放到 `?token=` 或 `Authorization: Bearer`；WebRTC 信令同一套 token。
- **房间权限**：在服务端维护 `room → ACL`，每条消息校验（写权限、速率）。
- **CORS/Origin**：WS/SSE 检查 `Origin`；WebRTC 限制信令来源与 TURN 域名。
- **隐私**：WebRTC 天生端到端；协作文档存储只保留 update（必要脱敏/加密）。

---

## 9）性能小抄 ⚡️

- **编码**：文本 JSON → **MsgPack/CBOR**；二进制帧省 20–40%。
- **合并包**：高频小消息 → 周期性批量（10–30ms 窗口），减少 syscalls。
- **局部广播**：只推给“可见用户/同房间”；不要全局广播。
- **索引**：Yjs 片段太多时做段落索引，渲染命中段落；游标独立通道。

---

## 10）端到端样例（Chat + 协作文本）🧪

### 10.1 客户端组合
```ts
// 1) 连接房间总线（WS）
const bus = connect('wss://api.example.com/ws?token=JWT', msg => {
  if (msg.t==='chat') appendMessage(msg);
  if (msg.t==='signal') handleWebRTCSignal(msg);
});

// 2) 文本协作（Yjs over y-websocket）
import { ydoc, ytext } from './yjs';
ytext.observe(() => renderEditorDiff());

// 3) 游标（WebRTC DataChannel，不可靠）
const rtc = setupRtc(bus); // 用 bus 做信令
rtc.data.onmessage = e => drawCursor(JSON.parse(e.data));
```

### 10.2 服务器拼装
```
[SSE /api/events]  ← 推系统公告/异步进度
[WS  /ws]          ← 聊天/信令/房间/鉴权/限速
[WebRTC P2P/SFU]   ← 媒体/游标
[Y-WebSocket]      ← 文档状态+持久化
```

---

## 11）Checklist ✅

- [ ] 选型：只下行→SSE；双工→WS；端到端/媒体→WebRTC；状态一致→Yjs  
- [ ] 代理：SSE 关缓冲；WS 做心跳；RTC 有 TURN 兜底  
- [ ] 协议：统一 envelope（type/room/seq/payload）与版本号  
- [ ] 背压：bufferedAmount 阈值 + 丢旧/降采样  
- [ ] 重连：指数退避 + 事件点位/快照恢复  
- [ ] 权限：JWT 短期 + 房间 ACL + 速率限制  
- [ ] 观测：connect/rtx/turn_ratio/latency/tps  
- [ ] Yjs：IndexedDB 离线 + 周期快照 + 审计 update  
- [ ] WebRTC：信令安全、STUN/TURN 凭证短 TTL、SFU 大房间

> 把“通信”与“状态一致性”分开思考：**WS/SSE/RTC 是水管**，**Yjs 是水的结构**。水管可以多根，水的形状却要统一，这就是高质量协作产品的底层秩序。🧪🌊

