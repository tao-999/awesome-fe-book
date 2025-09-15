# 32.5 直播 / 连麦 / 低延迟传输 📺🎤⚡️

> 目标：把“能播起来”升级为“**可规模、低时延、可观测、可回滚**”。覆盖 **协议选型（WebRTC / LL-HLS / RTMP / SRT）**、**连麦（SFU）**、**编码与自适应**、**安全与风控**、**录制回放** 与 **成本优化**。给你能抄的工程骨架与参数表。🧪

---

## 0）延迟地图（Glass-to-Glass）🧭

| 路线 | 常见栈 | 典型端到端时延 |
|---|---|---|
| 大规模直播优先 | **RTMP ingest → HLS/DASH**（CDN） | 6–20s（传统 HLS） |
| “准实时”直播 | **LL-HLS / LL-DASH**（CMAF 部分切片） | 1.5–5s |
| 互动 / 连麦 | **WebRTC（SFU）** | 150–800ms |
| 弱网上行稳定 | **SRT/RIST ingest → 转 WebRTC/HLS** | 0.5–3s（取决于 ARQ/latency） |

> 工程常见 **混合架构**：**主播/连麦**走 WebRTC（SFU），**大盘观众**走 LL-HLS；必要时提供“VIP 互动位”临时切到 WebRTC 观看通道。

---

## 1）三种可落地架构（你总能命中一个）

### A. 规模优先（直播电商/赛事）
```
推流端(RTMP/SRT) → Ingest(Origin) → Transcoder(多码率/封装) → CDN(LL-HLS)
观众(H5/小程序/TV) ← hls.js/系统播放器
```
- 优点：**海量分发**超便宜；兼容广。
- 缺点：端到端 1.5–5s（LL-HLS 已经很香，但不适合连麦）。

### B. 互动优先（连麦/课堂/会议）
```
主播/观众端(WebRTC) ↔ SFU(选择转发) ↔ 多节点集群
                       ↘ 录制/混流 ↘
```
- 优点：**亚秒级**；上行自适应（Simulcast/SVC）。
- 缺点：百万观众不经济（TURN/带宽/连接数）。

### C. 混合（推荐）
```
主播/连麦：WebRTC ↔ SFU → 录制/混流 → 封装 CMAF → CDN(LL-HLS)
普通观众：LL-HLS
```
- 优点：互动体验与规模成本两全；“一处转码，多处消费”。

---

## 2）编码与传输参数（稳中求快）

**视频（H.264 常规）**
- Profile **High**；**GOP = 1–2s**；**无 B 帧**（低时延）；**场景切换强制 I 帧**。
- 分辨率/码率建议（帧率 30fps）：
  - 360p：600–900kbps
  - 540p：1.2–1.8Mbps
  - 720p：2–3Mbps
  - 1080p：4–6Mbps
- Keyframe 对齐：多码率 ABR **关键帧对齐**，方便无缝切换。

**音频**
- **Opus 48kHz**，32–64kbps；AEC/NS/AGC 全开（设备端）。
- 混音：N-1（自己声音去回声环路）。

**拥塞控制**
- WebRTC：GCC/SCReaM（厂商实现）；启用 **NACK + PLI/FIR + RTX**；必要时 **ULPFEC**（有开销）。
- 上行码率自适应：动态降分辨率/帧率（prefer **分辨率优先降**）。

**Simulcast / SVC**
- **Simulcast**：同帧不同层（360p/540p/720p）；SFU 侧按订阅挑选。
- **SVC**：层级编码（VP9/AV1 可用），**带宽波动更丝滑**；兼容性略差。

---

## 3）连麦（多人上麦）设计要点 🎙️

- **房间与角色**：host / cohost / audience；seat 管理（申请、上麦、下麦、禁麦）。
- **音频优先**：音视频分轨；网抖优先保音频。
- **布局与混流**：
  - 客户端拼接（成本低，录制复杂）。
  - **云端混流（Composer）**：统一画面 → 录制/推 CDN。
- **跨房 PK / 串联**：SFU 间拉流或云混合并回注；注意 **时钟对齐** 和 **回声**。

---

## 4）WebRTC 落地骨架（WHIP/WHEP 或 SDK）

### 4.1 纯 WebRTC + WHIP/WHEP（HTTP 版信令，易接边缘）
```ts
// 推流：WHIP（POST SDP 对换）
const pc = new RTCPeerConnection({ iceServers: [{ urls: 'stun:stun.l.google.com:19302' }] });
const stream = await navigator.mediaDevices.getUserMedia({ video: { width:1280, height:720, frameRate:30 }, audio: { echoCancellation:true, noiseSuppression:true, autoGainControl:true } });
stream.getTracks().forEach(t => pc.addTrack(t, stream));
const offer = await pc.createOffer({ offerToReceiveAudio:false, offerToReceiveVideo:false });
await pc.setLocalDescription(offer);
const resp = await fetch('https://origin.example.com/whip', { method:'POST', headers:{ 'Content-Type':'application/sdp' }, body: offer.sdp });
await pc.setRemoteDescription({ type:'answer', sdp: await resp.text() });
// 维护 ICE（WHIP 可用 PATCH/ICE candidates 扩展）
```

```ts
// 拉流：WHEP（GET/POST SDP）
const pc = new RTCPeerConnection({ iceServers:[{ urls:'stun:stun.l.google.com:19302' }] });
pc.ontrack = e => video.srcObject = e.streams[0];
const offer = await pc.createOffer({ offerToReceiveVideo:true, offerToReceiveAudio:true });
await pc.setLocalDescription(offer);
const ans = await fetch('https://edge.example.com/whep', { method:'POST', headers:{'Content-Type':'application/sdp'}, body: offer.sdp });
await pc.setRemoteDescription({ type:'answer', sdp: await ans.text() });
```

### 4.2 用托管 SFU（LiveKit / mediasoup / Janus 等示例：LiveKit）
```ts
import { connect, createLocalTracks, Room } from 'livekit-client';

const room: Room = await connect('wss://sfu.example.com', 'JWT_FROM_SERVER', { adaptiveStream:true, dynacast:true });
const tracks = await createLocalTracks({ audio:{ echoCancellation:true }, video:{ resolution:'720p' } });
await room.localParticipant.publishTrack(tracks[0]); // audio
await room.localParticipant.publishTrack(tracks[1]); // video
room.on('trackSubscribed', (track, pub, participant) => attach(track, participant.sid));
```
- 打开 `adaptiveStream + dynacast` → **按可见性与带宽**只发/只收需要的层。

---

## 5）LL-HLS（准实时大规模分发）⚙️

**CMAF + 部分切片（part）**：把 2–4s 的 segment 再细分为 **200–500ms** 的 part，以 **chunked transfer** 推送。

**播放端（hls.js）**：
```html
<script src="https://cdn.jsdelivr.net/npm/hls.js@^1"></script>
<video id="v" playsinline muted></video>
<script>
  if (Hls.isSupported()) {
    const hls = new Hls({ lowLatencyMode: true, backBufferLength: 30, maxLiveSyncPlaybackRate: 1.5 });
    hls.loadSource('https://cdn.example.com/live/stream.m3u8');
    hls.attachMedia(document.getElementById('v'));
  }
</script>
```

**M3U8 片段（示意）**
```
#EXTM3U
#EXT-X-VERSION:9
#EXT-X-TARGETDURATION:2
#EXT-X-PART-INF:PART-TARGET=0.333
#EXT-X-SERVER-CONTROL:CAN-BLOCK-RELOAD=YES,PART-HOLD-BACK=1.0
#EXT-X-MAP:URI="init.cmfv"
#EXTINF:2.000,
file0.m4s
#EXT-X-PART:DURATION=0.333,URI="file1.part0.m4s"
#EXT-X-PART:DURATION=0.333,URI="file1.part1.m4s"
#EXTINF:2.000,
file1.m4s
#EXT-X-PRELOAD-HINT:TYPE=PART,URI="file2.part0.m4s"
```

**服务端关键点**
- Segmenter（如 ffmpeg/packager）：**CMAF、对齐关键帧、短 targetDuration、part-hold-back 合理**。
- CDN：支持 **chunked** 与 **HTTP/2/3**；**高命中率**是成本生命线。

---

## 6）音视频处理（端侧体验不翻车）

- **AEC/NS/AGC**：浏览器约束开启；外接麦走声卡直入；回声强时用回声消除器（硬件/外置）。
- **设备测试**：上麦前跑回环测试（采集→播放），提示设备/带宽评分。
- **美颜/虚拟背景**：WebAssembly 或 GPU（WebGL/WebGPU）。注意 **额外 5–15ms 延迟** 与 CPU/GPU 占用。
- **字幕/翻译**：ASR/翻译流经边缘服务，返回 WebVTT/SRT 或叠加图层；注意延迟预算（200–600ms）。

---

## 7）录制 / 回放 / 混流

- **录制方式**：
  - **SFU 原轨录制**（每个参与者一路）：后期自由混剪。
  - **云混流 + 录制**：直播时就做版面，回放即得成片。
- **切片归档**：封装为 HLS VOD（2–4s 段），产 **缩略图 / 章节** 与 **弹幕时间轴**。
- **回放弹幕**：按时间窗口批量加载，客户端 **插帧补齐**，避免 UI 抖动。

---

## 8）安全与合规 🛡️

- **鉴权**：推/拉流 URL **签名（ts + token）**，过期失效；WebRTC 用 **短期 JWT**。
- **SRTP/DTLS**：WebRTC 天然加密；SRT/RIST 也有 **AES**。
- **防盗链与限速**：CDN Referer/IP 白名单、User-Agent 过滤、速率与并发限制。
- **DRM/水印**：LL-HLS 可用 **AES-128/SAMPLE-AES**；WebRTC 端端 E2EE（Insertable Streams）或**叠加动态水印**。
- **内容安全**：实时 ASR + 视觉识别 + 关键词/图像检测；建立 **停播/静音** 快捷路径与审计留痕。

---

## 9）可观测与调优（指标一览）📈

- **端**：join_time、RTT、packet_loss、jitter、outbound/inbound_bitrate、decode/render_time、rebuffer_ratio、**glass_to_glass**。
- **边/服务**：SFU 转发时延、订阅层级命中率、TURN 占比、录制丢帧、CDN 命中率与 95/99 延迟。
- **AB 实验**：Simulcast vs SVC、GOP 长度、LL-HLS part 长度、播放器缓冲策略。
- **排障工具**：WebRTC `getStats()`、SFU 运行台、端到端 traceID 串联（推→转→播）。

---

## 10）成本与容量（别被账单 PTSD）

- **最大头**：CDN **下行出网**；其次 **转码 GPU/CPU**；WebRTC 的 **TURN**。
- **节流招**：
  - 直播观众走 **LL-HLS**；**VIP/互动**少量走 WebRTC。
  - **动态码率层控制**（dynacast/adaptiveStream）：观众不可见的层不转发。
  - 录制 **按需触发**；预先设置**最长直播时长**。
  - 编码 **预设调平**：x264 `veryfast/medium` 在画质/成本取平衡；高并发用 **NVENC**。

---

## 11）真实“连麦”流程（端到端）

1) 观众申请上麦 → 鉴权/席位检查 → 返回临时 JWT + 目标 SFU 节点  
2) 获取媒体（摄像头/麦克风），采样参数（720p/30fps / Opus 48k）  
3) 发布轨道（audio 必须；video 可按设备与网况打开）  
4) SFU 分发：其他连麦者订阅；云混流 → CDN（普通观众看 LL-HLS）  
5) 下麦：取消发布/回收席位；清理轨道与 UI  
6) 异常：检测无声/黑帧/过载，自动降级（关视频/降分辨/转听）  

---

## 12）端到端 Demo 片段合集

**A）浏览器上麦（自适应参数）**
```ts
const caps = await navigator.mediaDevices.getSupportedConstraints();
const stream = await navigator.mediaDevices.getUserMedia({
  audio: { echoCancellation:true, noiseSuppression:true, autoGainControl:true },
  video: { width:{ideal:1280}, height:{ideal:720}, frameRate:{ideal:30, max:30} }
});
// 网络差时：降分辨
function downgrade(track: MediaStreamTrack) {
  const sender = pc.getSenders().find(s => s.track === track);
  sender?.setParameters({ encodings: [{ scaleResolutionDownBy: 1.5 }, { maxBitrate: 900_000 }] });
}
```

**B）FFmpeg → LL-HLS（CMAF）**
```bash
ffmpeg -re -i input.mp4 \
  -c:v libx264 -preset veryfast -tune zerolatency -profile:v high -g 60 -keyint_min 60 -sc_threshold 0 -bf 0 -b:v 3000k \
  -c:a aac -ar 48000 -b:a 128k \
  -f hls -hls_time 2 -hls_list_size 6 -hls_flags independent_segments+split_by_time \
  -hls_segment_type fmp4 -hls_playlist_type event \
  -master_pl_name master.m3u8 -var_stream_map "v:0,a:0" \
  -hls_fmp4_init_filename init.m4s -hls_segment_filename "v%03d.m4s" out.m3u8
# 注意：LL-HLS 需启用 EXT-X-PART（可换 packager 实现）
```

**C）Insertable Streams（WebRTC 端到端加密/水印示意）**
```ts
// 仅示例：真正加密需密钥协商/轮转
const sender = pc.getSenders().find(s => s.track?.kind === 'video');
const transformer = new TransformStream({
  transform: async (encodedFrame, controller) => {
    // 在这里做简单标记/水印（例如修改部分字节）
    controller.enqueue(encodedFrame);
  }
});
sender?.transform = { readable: transformer.readable, writable: transformer.writable };
```

---

## 13）风控与合规（直播专属）

- **内容安全**：实时 ASR + CV 审核（涉黄涉暴涉政），命中策略 → **静音/遮挡/下线**。
- **礼物/打赏**：风控限额/地域/支付风控联动；异常退款闭环。
- **反滥用**：连麦席位限时、设备/账号速率、跨房跳转冷却、礼物刷屏折叠。
- **合规**：区域性监管（备案/许可证/未成年人保护），按地区**特性开关**。

---

## 14）Checklist（上线前最后一眼）✅

- [ ] 架构：互动 WebRTC + 大盘 LL-HLS 的 **混合** 路线  
- [ ] 编码：GOP=1–2s、无 B 帧、关键帧对齐、多码率/Simulcast  
- [ ] 连麦：席位/角色/混流策略、N-1 音频、弱网降级  
- [ ] 观测：端到端延迟、RTT/丢包、TURN 占比、CDN 命中、rebuffer  
- [ ] 安全：URL 签名/JWT、SRTP/DTLS、DRM/水印、内容审核  
- [ ] 录制：原轨/混流录制 → HLS VOD、弹幕时间轴  
- [ ] 成本：观众走 LL-HLS、dynacast、录制按需、CDN 缓存优化  
- [ ] 回滚：样式/编码/ABR 参数/节点伸缩可回滚，通道可切换  
- [ ] 压测：端上行弱网、万人并发观看、连麦 6–12 人稳定帧/音  

---

### 小结

直播不是单一协议的选择题，而是**延迟—规模—成本**三角之间的动态平衡。  
把 **WebRTC（互动）** 与 **LL-HLS（规模）** 组合，配上 **Simulcast/SVC、自适应、观测与风控**，你的系统才能既顶住大促又保证“**说到就到**”的互动体验。🎯
