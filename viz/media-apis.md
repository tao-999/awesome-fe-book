# 22.1 MediaDevices / WebRTC / WebCodecs 🎥📡🎛️

目标：构建“**获取媒体 → 处理编解码 → 低延迟传输/录制**”的一条龙能力。聚焦浏览器原生 API：`MediaDevices`、`WebRTC`、`WebCodecs`，以及周边 `MediaStreamTrackProcessor/Generator`、`getDisplayMedia`、`Insertable Streams`。

---

## 0) TL;DR（先会这三件）🎯
1. 采集：用 `getUserMedia` / `getDisplayMedia` 拿到 `MediaStreamTrack`，**明确 constraints**（分辨率/帧率/回声消除等）并监听 `devicechange`。  
2. 传输：用 `RTCPeerConnection` 建立 P2P；开启 **Trickle ICE**、码率/分辨率自适应（simulcast/SVC），`getStats` 做健康监控。  
3. 编解：在需要深度控制时用 **WebCodecs**（`VideoEncoder/VideoDecoder`）+ `Insertable Streams`，在 Worker 中串起“**解码 → 处理 → 编码**”，实现**超低延迟**与**端到端加密（E2EE）**。

---

## 1) 能力矩阵与选型 🧭

| 场景 | 首选 | 说明 |
|---|---|---|
| 摄像头/麦克风采集 | `navigator.mediaDevices.getUserMedia` | 结合 `applyConstraints` 调整分辨率/帧率/回声等 |
| 屏幕共享 | `getDisplayMedia` | 务必加“窗口/屏幕/标签页”选项与提示 |
| 简单实时通话 | WebRTC 媒体轨 | 浏览器自带拥塞控制/重传/FEC |
| 数据/实时信令 | WebRTC DataChannel | 低延迟、可靠/不可靠可选 |
| 录制 | `MediaRecorder`（入门）→ WebCodecs（进阶） | WebCodecs 灵活可控；MediaRecorder 快速 |
| 自定义处理/特效 | `MediaStreamTrackProcessor/Generator` 或 WebCodecs | Processor/Generator 走帧级处理；复杂算法用 WebCodecs |
| 专业编解码/超低延迟 | WebCodecs + MSE/WebRTC | 选择编码器（H264/VP9/AV1/Opus）并控制码率/分片 |

---

## 2) 设备采集与约束（Constraints）🎤🎥

### 2.1 最小示例
```ts
const stream = await navigator.mediaDevices.getUserMedia({
  audio: {
    echoCancellation: true, noiseSuppression: true, autoGainControl: true,
    channelCount: 1, sampleRate: 48000
  },
  video: {
    width: { ideal: 1280 }, height: { ideal: 720 }, frameRate: { ideal: 30, max: 60 },
    facingMode: 'user' // 或 'environment'
  }
});
document.querySelector('video')!.srcObject = stream;
```

### 2.2 调整与监听
```ts
const [videoTrack] = stream.getVideoTracks();
await videoTrack.applyConstraints({ frameRate: { ideal: 24, max: 30 } });

navigator.mediaDevices.addEventListener('devicechange', async () => {
  const devices = await navigator.mediaDevices.enumerateDevices();
  // 设备列表变化：更新设备选择 UI
});
```

### 2.3 屏幕共享
```ts
const share = await navigator.mediaDevices.getDisplayMedia({
  video: { displaySurface: 'window', frameRate: { ideal: 30, max: 60 } },
  audio: true // 仅部分浏览器支持“系统音频”
});
```

> 提示：采集权限需要 HTTPS/localhost；屏幕共享要给出明显 UI 引导与“停止共享”措施（`track.stop()`）。

---

## 3) WebRTC：连接、码流与数据通道 📡

### 3.1 基本连接（Offer/Answer + Trickle ICE）
```ts
const pc = new RTCPeerConnection({
  iceServers: [{ urls: ['stun:stun.l.google.com:19302'] }] // 生产应配 STUN+TURN
});
stream.getTracks().forEach(t => pc.addTrack(t, stream));
pc.ontrack = (e) => { remoteVideo.srcObject = e.streams[0]; };

pc.onicecandidate = (e) => { if (e.candidate) signaling.send({ type:'ice', candidate: e.candidate }); };
signaling.on('ice', async c => { try { await pc.addIceCandidate(c.candidate); } catch {} });

const offer = await pc.createOffer({ offerToReceiveAudio: true, offerToReceiveVideo: true });
await pc.setLocalDescription(offer);
signaling.send({ type:'offer', sdp: pc.localDescription });

signaling.on('answer', async (ans) => { await pc.setRemoteDescription(ans.sdp); });
signaling.on('offer', async (off) => {
  await pc.setRemoteDescription(off.sdp);
  const answer = await pc.createAnswer();
  await pc.setLocalDescription(answer);
  signaling.send({ type:'answer', sdp: pc.localDescription });
});
```

### 3.2 数据通道（可靠/不可靠）
```ts
const dc = pc.createDataChannel('chat', { ordered: true, maxRetransmits: 0 /* 不可靠低延迟 */ });
dc.onopen = () => dc.send('hello');
pc.ondatachannel = (e) => { e.channel.onmessage = (m) => console.log(m.data); };
```

### 3.3 码率与分层（Simulcast / SVC）
- **Simulcast**：同一视频多路不同分辨率/码率，由 SFU 挑选转发。  
- **SVC（可分级编码）**：单码流内含层，AV1/VP9 支持更好。  
- 设置示例（取决于浏览器支持）：
```ts
const sender = pc.getSenders().find(s => s.track?.kind === 'video')!;
await sender.setParameters({
  encodings: [
    { rid: 'f', maxBitrate: 2500_000, scaleResolutionDownBy: 1 },
    { rid: 'h', maxBitrate: 1200_000, scaleResolutionDownBy: 2 },
    { rid: 'q', maxBitrate:  350_000, scaleResolutionDownBy: 4 },
  ]
});
```

### 3.4 统计与健康检查
```ts
const stats = await pc.getStats();
stats.forEach(r => {
  if (r.type === 'outbound-rtp' && r.kind === 'video') {
    // 关键指标：bitrate、framesEncoded、frameWidth/Height、nackCount、pliCount、qualityLimitationReason...
  }
});
```

> 生产建议：接入 **RTT/jitter/丢包率** 可视化；在网络变差时**降低分辨率或帧率**，并记录自适应事件。

---

## 4) MediaRecorder vs WebCodecs（录制与转码）💾

### 4.1 快速录制（入门）
```ts
const rec = new MediaRecorder(stream, { mimeType: 'video/webm;codecs=vp9,opus' });
const chunks: BlobPart[] = [];
rec.ondataavailable = e => chunks.push(e.data);
rec.onstop = () => { const blob = new Blob(chunks, { type: rec.mimeType }); download(blob, 'record.webm'); };
rec.start(1000); // 每秒切片
```

### 4.2 WebCodecs（可控且低延迟）
- 直接控制编码器参数（码率、分片、关键帧），更适合**实时串联**到 MSE 或 WebRTC。  
- 通常放到 **Worker**，避免阻塞主线程。

```ts
// 编码：将 <canvas> 帧编码成 H264/VP9/AV1
const encoder = new VideoEncoder({
  output: (chunk, meta) => muxer.feed(chunk, meta), // 交给 MSE/自定义封装
  error: (e) => console.error(e)
});
encoder.configure({ codec: 'vp9', width: 1280, height: 720, bitrate: 1_500_000, framerate: 30 });

const track = stream.getVideoTracks()[0];
const processor = new MediaStreamTrackProcessor({ track });
const reader = processor.readable.getReader();

(async () => {
  let ts = 0;
  while (true) {
    const { value: frame, done } = await reader.read();
    if (done) break;
    encoder.encode(frame, { keyFrame: (ts % 3000) === 0 }); // 每 ~3s 关键帧
    ts += 1000 / 30;
    frame.close();
  }
})();
```

> **解码**类似：`VideoDecoder` 输出 `VideoFrame` → 绘制到 `canvas` 或 `VideoFrame.copyTo()` 供 WebGL/WebGPU 使用。

---

## 5) Insertable Streams 与端到端加密（E2EE）🔐

### 5.1 对 **编码后的 RTP** 做处理（如加密/滤波）
```ts
// 旧接口：sender.createEncodedStreams()
const sender = pc.getSenders().find(s => s.track?.kind === 'video')!;
const { readable, writable } = (sender as any).createEncodedStreams();

const key = await crypto.subtle.generateKey({ name: 'AES-GCM', length: 128 }, true, ['encrypt','decrypt']);
const enc = readable.pipeThrough(new TransformStream({
  async transform(chunk, controller) {
    const cipher = await crypto.subtle.encrypt({ name: 'AES-GCM', iv: chunk.getMetadata().synchronizationSource.toString().padStart(12,'0') }, key, chunk.data);
    chunk.data = new Uint8Array(cipher);
    controller.enqueue(chunk);
  }
}));
await enc.pipeTo(writable);
```

> 说明：不同浏览器对 insertable streams API 命名/可用性略有差异（`RTCRtpScriptTransform`/`createEncodedStreams`），需做特性检测与降级。

---

## 6) Processor/Generator：帧级处理流水线 🧪

```ts
const vTrack = stream.getVideoTracks()[0];
const proc = new MediaStreamTrackProcessor({ track: vTrack });
const gen = new MediaStreamTrackGenerator({ kind: 'video' });

const transformer = new TransformStream({
  async transform(frame: VideoFrame, controller) {
    // 在这里处理像素：可用 WebGL/WebGPU/Canvas 2D；示例：直接转发
    controller.enqueue(frame);
  }
});
proc.readable.pipeThrough(transformer).pipeTo(gen.writable);

// 将处理后的 track 再送回 WebRTC 或 <video>
const processed = new MediaStream([gen]);
pc.addTrack(processed.getVideoTracks()[0], processed);
```

---

## 7) 性能工程与 Worker/OffscreenCanvas ⚙️

- **渲染在 Worker**：`OffscreenCanvas` + 2D/WebGL，将绘制与 UI 解耦。  
- **编解码在 Worker**：WebCodecs API 可在 Worker 使用；用 **Transferable（ArrayBuffer）** 传输数据，避免复制。  
- **零拷贝**：`VideoFrame` → `transferToImageBitmap()` / `copyTo()`；`postMessage` 时传 `[[frame]]`。  
- **降级策略**：在 `getStats`/自定义指标触发阈值时，降低帧率/分辨率/码率或切换仅音频。

---

## 8) 网络与抗差策略 🌐

- **拥塞/丢包**：WebRTC 内置 BWE、NACK、PLI、FEC；在 `qualityLimitationReason` 为 `bandwidth` 时触发自适应。  
- **弱网优化**：  
  - 限制关键帧周期（2–4 s）与关键帧大小；  
  - 优先 **音频可用**（Opus 16–24 kbps）；  
  - 移动端优先 540p@24/30 或更低。  
- **TURN**：对称 NAT/企业网环境务必部署 TURN 服务器，保证联通性。  

---

## 9) 安全、权限与 UX 🧩

- **HTTPS 强制**，权限提示清晰，提供“测试麦克风/摄像头”页。  
- **自动播放策略**：静音视频可自动播放；有声需用户手势；可用 `video.muted = true` + `playsInline`。  
- **隐私**：避免把 PII 放进 SDP/信令；对录制/上传给出明确提示与开关；遵守本书 §18.* 隐私章节。

---

## 10) 调试与观测 🔍

- **WebRTC Internals**（浏览器内置）查看 SDP/统计。  
- `pc.getStats()`：关键监控项  
  - 发送：`outbound-rtp` → `bitrate`, `framesEncoded`, `qualityLimitationReason`  
  - 接收：`inbound-rtp` → `jitter`, `framesDecoded`, `packetsLost`  
  - 连接：`candidate-pair` → `currentRoundTripTime`, `availableOutgoingBitrate`  
- **日志**：信令/ICE/重协商事件、编码器切换、分层选择变更。

---

## 11) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 只用默认约束采集 | 画面糊、回声重 | 明确分辨率/帧率与 `echoCancellation` 等 |
| 不部署 TURN | 跨网/企业网连不上 | 配 STUN+TURN，检测失败自动降级 |
| 一路大码流硬怼 | 弱网卡顿、花屏 | Simulcast/SVC + 自适应降级 |
| 主线程做解码/绘制 | 滚动/输入卡顿 | WebCodecs/OffscreenCanvas + Worker |
| 无监控 | 出了问题无从下手 | 接入 `getStats` 看板与告警 |
| 全量清晰度固定 | 无法适应设备/CPU | 动态码率/分辨率/帧率 |

---

## 12) 验收清单 ✅
- [ ] 设备采集与切换稳定；`applyConstraints` 有效。  
- [ ] WebRTC：Offer/Answer、Trickle ICE、重协商（切换屏幕/摄像头）可用。  
- [ ] 码率/分辨率自适应：Simulcast 或 SVC 已启用；弱网有降级。  
- [ ] 录制：入门（MediaRecorder）与进阶（WebCodecs）两套链路通。  
- [ ] Worker 化处理：编解码/绘制不阻塞 UI；零拷贝传输。  
- [ ] 监控：`getStats` 指标上报，异常（高丢包/高 RTT/卡顿）告警。  
- [ ] 安全与 UX：HTTPS、权限与自动播放策略、停止采集与隐私文案完备。

---

## 13) 练习 🏋️
1. 做一个“设备测试页”：摄像头/麦克风/屏幕共享 + 约束切换 + 波形/直方图显示。  
2. 实现一个“一发多收”的 SFU Demo：发送端启用 **Simulcast**，接收端根据带宽自动选择层。  
3. 用 **WebCodecs** 将摄像头帧做高斯模糊后编码，再通过 **Insertable Streams** 端到端加密送入 WebRTC。  
4. 采集 `getStats` 指标，做“弱网模拟”（丢包/限速），编写降级策略并出具复盘报告。

---

**小结**：借助 `MediaDevices` 拿到原始输入，`WebRTC` 负责**可靠实时**传输，`WebCodecs` 提供**编解码与帧级控制**。把处理搬到 Worker、用分层与自适应抗弱网、用指标闭环，你就能交付一个**清晰、稳定、低延迟**的实时音视频体验。
