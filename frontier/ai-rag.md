
# 27.2 RAG 前端与文件处理（Web Workers + WASM）🧩📄⚙️

> 把“检索增强生成”（RAG）**搬进浏览器端**：文件本地解析 → 切片 → 向量化 → 近邻检索 → 证据高亮 → 流式回答。  
> 关键词：**Web Workers**（不卡主线程）、**WASM**（PDF/OCR/向量库/Embedding）、**IndexedDB**（持久化），**可视化证据链**（与 27.1 的 TraceEvent 打通）。🚀

---

## 0）整体蓝图（前端端到端）

```
       ┌────────────┐     ┌──────────────┐    ┌──────────┐
files→ │ Parse & OCR│ →→  │ Chunk & Meta │ →→ │ Embedder │
       └─────┬──────┘     └──────┬───────┘    └────┬─────┘
             │ (WASM: pdf/ocr)          │            │ (WASM/远程)
             ▼                          ▼            ▼
        TextBlocks                   Chunks      Float32 Embeds
             └───────────────┬────────┘            │
                             ▼                      │
                        Vector Store (IndexedDB/SQLite WASM)
                             │                      ▲
                 Query→Embed │                      │
                             ▼                      │
                       TopK candidates  ── re-rank（可选：CrossEncoder）
                             │
                             ▼
       TraceEvent: cite/tool_call/decision（可视化时间轴） + 流式答案
```

---

## 1）Workers 管道：文件解析与切片 ✂️

> 在 **Worker** 里干重活：PDF 文本/图片、OCR 回退、Docx/Markdown/HTML 解析。主线程负责 UI & 流式渲染。

### 1.1 消息协议（类型约定）

```ts
// src/rag/types.ts
export type DocId = string;

export type IngestJob =
  | { type: 'ingest/start'; docId: DocId; file: File; ocr?: boolean }
  | { type: 'ingest/cancel'; docId: DocId };

export type IngestProgress =
  | { type: 'ingest/progress'; docId: DocId; phase: 'parse'|'chunk'|'embed'; pct: number; note?: string }
  | { type: 'ingest/done'; docId: DocId; chunks: number }
  | { type: 'ingest/error'; docId: DocId; error: string };

export type QueryJob =
  | { type: 'query'; q: string; topK?: number; withRerank?: boolean };

export type QueryResult = {
  type: 'query/result';
  q: string;
  items: Array<{
    docId: DocId;
    chunkId: string;
    text: string;
    score: number;
    meta?: Record<string, any>;
  }>;
};
```

### 1.2 Worker 入口（解析 + 切片）

```ts
// src/rag/worker.ts  (注册为 new Worker(new URL('./worker.ts', import.meta.url), {type:'module'}))
import { IngestJob, IngestProgress, QueryJob, QueryResult } from './types';
import { chunkCJK } from './chunker';
import { embedText } from './embedder';  // 支持 WASM/远程双模
import { db, putChunk, searchTopK } from './vectordb'; // IndexedDB 封装
import { parseAny } from './parsers';    // PDF/Docx/MD/HTML 多格式解析

self.onmessage = async (evt: MessageEvent<IngestJob | QueryJob>) => {
  const msg = evt.data as any;
  try {
    if (msg.type === 'ingest/start') {
      const { docId, file, ocr } = msg;
      post({ type:'ingest/progress', docId, phase:'parse', pct:5, note:'parsing' });

      // 1) 解析文本（PDF → 文本；图片 PDF/扫描件走 OCR）
      const { text, meta } = await parseAny(file, { ocr });

      post({ type:'ingest/progress', docId, phase:'chunk', pct:25, note:'chunking' });

      // 2) 切片（对中文/中英混合友好）
      const chunks = chunkCJK(text, { maxTokens: 512, overlap: 64 });

      // 3) 嵌入 & 入库（流式进度）
      let i = 0;
      for (const c of chunks) {
        const vec = await embedText(c.text);               // Float32Array
        await putChunk(docId, c.id, c.text, vec, { ...meta, ...c.meta });
        i++;
        if (i % 8 === 0) {
          post({ type:'ingest/progress', docId, phase:'embed', pct: 25 + Math.round(75*i/chunks.length) });
        }
      }
      post({ type:'ingest/done', docId, chunks: chunks.length });
      return;
    }

    if (msg.type === 'query') {
      const topK = msg.topK ?? 6;
      const qvec = await embedText(msg.q);
      const items = await searchTopK(qvec, topK, { withRerank: msg.withRerank });
      const res: QueryResult = { type:'query/result', q: msg.q, items };
      post(res);
      return;
    }
  } catch (e:any) {
    if ('docId' in msg) post({ type:'ingest/error', docId: msg.docId, error: e?.message || String(e) });
    else console.error(e);
  }
};

function post(data: IngestProgress | QueryResult) {
  // 尽量传可转移对象，减少拷贝
  // 这里数据量小，直接 post；大批量场景传 ArrayBuffer
  (self as any).postMessage(data);
}
```

### 1.3 切片器（兼容中英 & 语义边界）

```ts
// src/rag/chunker.ts
const PUNC = /([。！？；?!;]|[\n\r]+)/g;

export function chunkCJK(text: string, opt: { maxTokens: number; overlap: number }) {
  // 近似 token 估计：中文按 1 字=1 token，英文按 0.75 字
  const estTokens = (s: string) =>
    [...s].reduce((t, ch) => t + (/[\u4e00-\u9fa5]/.test(ch) ? 1 : 0.75), 0);

  const sentences = text.split(PUNC).reduce<string[]>((acc, cur) => {
    if (!cur) return acc;
    const last = acc[acc.length - 1];
    if (PUNC.test(cur)) acc[acc.length - 1] = (last || '') + cur; else acc.push(cur);
    return acc;
  }, []);

  const chunks: { id: string; text: string; meta: any }[] = [];
  let buf: string[] = [];
  let bufTok = 0;

  const flush = () => {
    if (!buf.length) return;
    const content = buf.join('');
    const id = crypto.randomUUID();
    chunks.push({ id, text: content, meta: { tokens: Math.round(bufTok) } });
    // 交叠窗口
    const keep = Math.max(1, Math.round(buf.length * opt.overlap / (opt.maxTokens || 1)));
    buf = buf.slice(-keep);
    bufTok = estTokens(buf.join(''));
  };

  for (const s of sentences) {
    const t = estTokens(s);
    if (bufTok + t > opt.maxTokens) flush();
    buf.push(s);
    bufTok += t;
  }
  flush();
  return chunks;
}
```

---

## 2）解析器（PDF/图片 OCR/Docx/Markdown/HTML）📄🔍

> PDF 文本用 **pdf.js**（WASM/Worker 版），图片/扫描件回退 **Tesseract WASM**（需 COOP/COEP 才能开线程+SIMD），Docx/MD/HTML 用轻量解析。

```ts
// src/rag/parsers.ts
import * as pdfjs from 'pdfjs-dist';           // 提供 worker & wasm 版本
// 也可用 pdfium-wasm；本文以 pdf.js 思路描述
// OCR: tesseract.js (wasm)

export async function parseAny(file: File, opt: { ocr?: boolean }) {
  const name = file.name.toLowerCase();
  if (name.endsWith('.pdf')) return parsePDF(file, opt);
  if (name.endsWith('.md'))  return parseMD(await file.text());
  if (name.endsWith('.html') || name.endsWith('.htm')) return parseHTML(await file.text());
  if (name.endsWith('.docx')) return parseDocx(file); // 可用 docx-parser/wasm
  // 图片类：png/jpg/webp/tiff → OCR
  if (/\.(png|jpe?g|webp|tiff?)$/.test(name)) return ocrImage(file);
  // 兜底纯文本
  return { text: await file.text(), meta: { type:'text' } };
}

async function parsePDF(file: File, opt: { ocr?: boolean }) {
  const buf = await file.arrayBuffer();
  const pdf = await pdfjs.getDocument({ data: buf }).promise;

  let text = '';
  for (let p = 1; p <= pdf.numPages; p++) {
    const page = await pdf.getPage(p);
    const content = await page.getTextContent();
    const pageText = content.items.map((i:any)=>i.str).join('');
    if (pageText.trim().length < 10 && opt.ocr) {
      // 回退：渲染位图 + OCR（仅示意）
      const bmp = await renderPageBitmap(page, 2 /* scale */);
      text += '\n' + await ocrBitmap(bmp);
    } else {
      text += '\n' + pageText;
    }
  }
  return { text, meta: { type:'pdf', pages: pdf.numPages } };
}

async function renderPageBitmap(page: any, scale = 2): Promise<ImageData> {
  const viewport = page.getViewport({ scale });
  const canvas = new OffscreenCanvas(viewport.width, viewport.height);
  const ctx = canvas.getContext('2d')!;
  // pdf.js render → canvas
  await page.render({ canvasContext: ctx as any, viewport }).promise;
  return ctx.getImageData(0, 0, canvas.width, canvas.height);
}

// --- 伪实现：接你的 Tesseract WASM worker ---
async function ocrBitmap(img: ImageData): Promise<string> {
  // 将 ImageData 转 Blob/PNG，再喂给 OCR
  return '[OCR_TEXT]';
}

function parseMD(md: string) {
  // 可用 marked/markdown-it 去标签后再切片
  const text = md.replace(/```[\s\S]*?```/g, '').replace(/[#>*_`]/g, '');
  return { text, meta: { type:'md' } };
}

function parseHTML(html: string) {
  const div = new DOMParser().parseFromString(html, 'text/html');
  const text = (div.body?.innerText || '').trim();
  return { text, meta: { type:'html', title: div.title } };
}

async function parseDocx(file: File) {
  // 选用 wasm 版解析器；此处示意
  return { text: '[DOCX_TEXT]', meta: { type:'docx' } };
}
```

> **并行 OCR 提示**：Tesseract 开多线程需 **Cross-Origin-Opener-Policy** 与 **Cross-Origin-Embedder-Policy**，详见 §5。

---

## 3）Embedding：WASM 本地 or 远程双模 🧠

> 小规模/隐私优先：**本地 WASM（SIMD/threads）**，如 `xenova/transformers.js` 的 `all-MiniLM-L6-v2`。  
> 大规模/低延迟：**远程 Embedding API**，Worker 内部自动切换 + IndexedDB 缓存。

```ts
// src/rag/embedder.ts
let localReady = false;

export async function initLocalEmbedder() {
  // 懒加载 wasm 模型（threads+simd 更稳）
  // 省略细节：加载模型/缓存到 OPFS/IndexedDB
  localReady = true;
}

export async function embedText(text: string): Promise<Float32Array> {
  if (localReady) {
    // 本地推理（伪实现）
    return l2norm(new Float32Array(384).map(()=>Math.random())); // 替换为真实输出
  }
  // 远程（HTTP）——带去重缓存
  const r = await fetch('/api/embed', {
    method: 'POST',
    headers: { 'Content-Type':'application/json' },
    body: JSON.stringify({ text })
  });
  const { vector } = await r.json(); // vector: number[]
  return l2norm(Float32Array.from(vector));
}

function l2norm(v: Float32Array) {
  let s = 0; for (let i=0; i<v.length; i++) s += v[i]*v[i];
  const k = 1/Math.sqrt(s || 1);
  for (let i=0; i<v.length; i++) v[i] *= k;
  return v;
}
```

---

## 4）向量存储：IndexedDB + 余弦近邻 🔍

> 轻量可靠：**Dexie/IDB** 存文本+向量，前端直接算 TopK（几千~几万条可用）。更大规模可用 **SQLite WASM + FTS** 或 **HNSWlib-wasm**。

```ts
// src/rag/vectordb.ts
import Dexie, { Table } from 'dexie';

export interface ChunkRow {
  docId: string;
  chunkId: string;
  text: string;
  // 向量以 Blob 存储（Float32Array.buffer）
  embed: Blob;
  meta?: any;
}

class RAGDB extends Dexie {
  chunks!: Table<ChunkRow, [string,string]>;
  constructor() {
    super('RAGDB');
    this.version(1).stores({
      chunks: '[docId+chunkId], docId, text'
    });
  }
}
export const db = new RAGDB();

export async function putChunk(docId: string, chunkId: string, text: string, vec: Float32Array, meta?: any) {
  await db.chunks.put({
    docId, chunkId, text, meta,
    embed: new Blob([vec.buffer], { type: 'application/octet-stream' })
  });
}

export async function searchTopK(q: Float32Array, k=6, opt?: { withRerank?: boolean }) {
  // 读取全部（可按 docId/时间分区）；实际可做分桶/倒排/粗排
  const rows = await db.chunks.toArray();
  const scores = rows.map((r) => ({
    row: r,
    score: dot(q, r.embed)
  }));
  scores.sort((a,b)=> b.score - a.score);
  const top = scores.slice(0, k);

  // 可选：re-rank（远程 Cross-Encoder）
  if (opt?.withRerank) {
    const reranked = await rerank(q, top.map(t=>t.row.text));
    top.sort((a,b)=> reranked.get(b.row.text)! - reranked.get(a.row.text)!);
  }
  return top.map(t => ({
    docId: t.row.docId,
    chunkId: t.row.chunkId,
    text: t.row.text,
    score: t.score,
    meta: t.row.meta
  }));
}

function dot(q: Float32Array, blob: Blob): number {
  // 直接把 Blob 读成 ArrayBuffer（可 cache）
  // 注意：这里是同步简化，生产里做缓存 & 批量读取
  // @ts-expect-error
  const buf = (blob as any).__cache || (blob as any).__cache = undefined;
  const p = buf ? Promise.resolve(buf) : blob.arrayBuffer().then(b => ((blob as any).__cache=b, b));
  // 这段返回 Promise，searchTopK 简化为串行 await；上方为了简洁写了同步风格
  // 可将 rows 分块并行以提升吞吐
  throw new Error('示意：实际实现请改为 async 路径');
}
```

> 说明：上面 `dot` 为示意。**推荐真正实现：**在 `searchTopK` 内将所有 Blob `arrayBuffer()` 并行读取为 `Float32Array`，然后计算余弦相似度；对 1~5 万条数据可接受。更大规模切换 **HNSWlib-wasm** 或服务端向量库（Milvus/PGVector/LanceDB）。

---

## 5）查询 → 证据 → 流式回答（与 27.1 串联）🪄

**主线程**：

```ts
// src/app/query.ts
import { TraceEvent } from '../ai/trace'; // 见 27.1
const worker = new Worker(new URL('../rag/worker.ts', import.meta.url), { type: 'module' });

export async function ask(q: string, onToken: (t:string)=>void, onTrace:(e:TraceEvent)=>void) {
  // 1) 前检索
  const res = await queryRAG(q); // 走 worker → searchTopK
  onTrace({ type:'plan', brief:`检索到 ${res.items.length} 条候选` });
  res.items.forEach((it,i)=>{
    onTrace({ type:'cite', source:`${it.docId}#${it.chunkId}`, snippet: it.text.slice(0,160) });
  });

  // 2) 组织上下文（截断到 X tokens）
  const context = res.items.map((x,i)=>`[${i+1}] ${x.text}`).join('\n\n');

  // 3) 流式调用 LLM（SSE）
  const body = { prompt: buildPrompt(q, context) };
  const resStream = await fetch('/api/chat/stream', { method:'POST', body: JSON.stringify(body) });
  const reader = resStream.body!.getReader();
  const decoder = new TextDecoder();
  while (true) {
    const {value, done} = await reader.read();
    if (done) break;
    onToken(decoder.decode(value, { stream:true }));
  }
  onTrace({ type:'final', summary:'回答已完成（含引用）' });
}

function buildPrompt(q: string, ctx: string) {
  return `你是严谨的助手。优先使用“上下文片段”回答；无法从上下文得到答案时，明确说“不确定”。

# 上下文片段
${ctx}

# 问题
${q}

# 输出要求
- 先给结论，再给引用编号，如[1][3]
- 不编造未在片段里的细节`;
}

function queryRAG(q:string) {
  return new Promise<any>((resolve) => {
    const onMsg = (e: MessageEvent) => {
      if (e.data?.type === 'query/result') {
        worker.removeEventListener('message', onMsg);
        resolve(e.data);
      }
    };
    worker.addEventListener('message', onMsg);
    worker.postMessage({ type:'query', q, topK:6, withRerank:false } as any);
  });
}
```

**高亮引用**：渲染 Markdown 时，扫描文末 `[1]` 样式，把对应段落高亮/折叠展开。

---

## 6）性能关键点（线程/拷贝/SIMD/并行）⚡️

- **跨源隔离（COOP/COEP）**：开启 **SharedArrayBuffer / WASM 线程+SIMD**，OCR/Embedding 才飞起来。  
  - Dev（Vite）：
    ```ts
    // vite.config.ts
    export default defineConfig({
      server: {
        headers: {
          'Cross-Origin-Opener-Policy': 'same-origin',
          'Cross-Origin-Embedder-Policy': 'require-corp',
        }
      }
    });
    ```
  - Prod（Nginx）：
    ```
    add_header Cross-Origin-Opener-Policy same-origin always;
    add_header Cross-Origin-Embedder-Policy require-corp always;
    ```
- **可转移对象**：`postMessage(data, [data.buffer])` 传 `ArrayBuffer`，避免大对象深拷贝。
- **并行策略**：解析/嵌入分批并行；控制并发（e.g. 3-4）避免内存爆。
- **内存回收**：大 `Float32Array` 用完置 `null`；WASM 模块周期 `terminate()`/重建避免泄漏。
- **索引分区**：大库分 docId 或日期桶，先粗排（BM25/关键词）再精排（向量）。

---

## 7）安全 & 隐私 🛡️

- **默认本地**：解析/嵌入/检索都在浏览器落地，不上传文件内容。  
- **远程选项**：仅当用户同意时走远程 Embedding/重排；传输前做**片段脱敏**。  
- **日志最小化**：保留 `TraceEvent` 摘要，不存全文；必要时开启临时缓存 TTL。

---

## 8）可视化：证据—结论的“拉线图”🧵

- 左侧时间轴：`plan → tool_call(parse/chunk/embed) → cite → decision → final`。  
- 右侧证据卡片：片段、打分、来源（docId/页码）。  
- 搜索结果可切换 **“热力图”**（关键词/Embedding 相似度渐变色）。

---

## 9）脚手架清单（最小可跑）

```
src/
  rag/
    worker.ts          # 解析/切片/嵌入/检索
    parsers.ts         # PDF/Docx/MD/HTML/OCR
    chunker.ts         # CJK 切片 + overlap
    embedder.ts        # 本地WASM/远程双模
    vectordb.ts        # IndexedDB/HNSW 适配
    types.ts
  ui/
    IngestDropzone.tsx # 拖拽导入 + 进度
    SearchBox.tsx      # 查询输入 + 结果
    ChatWithRAG.tsx    # 结合流式回答 + 引用高亮
  ai/
    stream.ts          # SSE 流读取（见 27.1）
    trace.ts           # TraceEvent（见 27.1）
```

---

## 10）常见坑位与补救 🕳️🩹

- **SharedArrayBuffer 类型报错** → 没开 COOP/COEP；或 Safari 版本太低，改成单线程模型。  
- **PDF 提取乱码** → 切换 `pdf.js` 字体映射/用 PDFium WASM；或渲染位图走 OCR。  
- **长文掉帧** → 结果虚拟化：分段渲染 + `requestIdleCallback` 切片解析。  
- **向量库太慢** → 先 BM25 粗排 50 条，再向量精排 TopK。  
- **浏览器存储上限** → 分片压缩（gzip/brotli）、图片不入库、只存嵌入 + 片段指针。

---

## 11）升级路线 🧗

- 替换向量检索为 **HNSWlib-wasm** 或 **SQLite-WASM + IVF/邻近索引**。  
- 本地 Embedding 升级到 **SIMD + Threads** 模式，模型缓存 OPFS。  
- Re-rank 用轻量 Cross-Encoder（远程），只传 Top50 文本。  
- 多模态：图片 OCR → 图文 RAG；音频 → 语音识别（Whisper WASM/远程）。

---

## 小结

- **Workers** 扛住解析与向量化，**WASM** 提供 PDF/OCR/Embedding 算力，**IndexedDB** 兜底持久化。  
- 把 **RAG 证据链** 串到 27.1 的 **TraceEvent**，用户看到的不仅是答案，还有“为什么”。  
- 先用轻量实现“能跑”，再按数据规模替换子组件（ANN、OCR、Embedder）就是正确的进化方式。🦾

