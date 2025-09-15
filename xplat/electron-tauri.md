
# 25.1 Electron vs Tauri ⚔️

> 用数据点和工程约束做决策，不靠“宗教战争”。下文从 **架构/性能/安全/生态/发布/原生扩展** 六个维度拆解，并附最小可用模板与避坑清单。🚀

---

## TL;DR

- **需要完整且一致的 Chromium 能力（DevTools、媒体编解码、WebGPU/WebGL、扩展生态）、Node.js 包随取随用、插件/资料极其丰富 → Electron**  
- **追求更小包体/更低内存/更强默认安全模型、愿意用 Rust 写系统能力或性能关键路径、期望复用到移动平台（v2 提供 iOS/Android 目标） → Tauri**

---

## 核心架构对比

| 维度 | Electron | Tauri |
| --- | --- | --- |
| 渲染内核 | 自带 **Chromium**（跨平台一致） | 复用 **系统 WebView**（Win: WebView2 / macOS: WKWebView / Linux: WebKitGTK） |
| 后端运行时 | **Node.js**（可用 Node-API/C++ 扩展） | **Rust**（Tauri 插件 / `#[tauri::command]`） |
| 包体/内存 | 相对更大 / 占用更高 | 相对更小 / 占用更低 |
| 能力边界 | “带浏览器来做桌面” → 能力充沛 | “最小外壳 + 原生胶水” → 能力按需开 |
| 安全模型 | 可很安全，但需**严格配置** | **默认更收敛**（能力白名单/权限分离） |
| 跨端版图 | 桌面（Win/macOS/Linux） | 桌面 +（v2）移动目标（iOS/Android） |
| 生态/资料 | 历史最久、社区极大 | 增长快、Rust crates 丰富 |

> 心法：**Electron = 自带浏览器的应用**；**Tauri = WebView 外壳 + Rust 能力按需挂载**。

---

## 性能画像

- **启动/包体/常驻内存**：Tauri 通常更优（不捆绑 Chromium）。  
- **图形前沿能力**：Electron 跟随 Chromium 更一致；Tauri 受系统 WebView 版本制约（Windows 需较新的 WebView2）。  
- **重计算/IO**：两者都建议移至原生侧：Electron 用 Node-API/C++/worker_threads；Tauri 用 Rust 线程/异步、零拷贝通路。

---

## 安全模型要点 🛡️

### Electron 最小攻击面配置
- 仅通过 **Preload + IPC** 暴露白名单 API；禁用 `remote` 与 `nodeIntegration`
- `contextIsolation: true`、`sandbox: true`、严格 **CSP**、外链统一 `shell.openExternal` 审核
- 主进程持有敏感能力，渲染进程不可直接 `require('fs')`

```ts
// electron/main.ts
const win = new BrowserWindow({
  webPreferences: {
    preload: join(__dirname, 'preload.js'),
    contextIsolation: true,
    nodeIntegration: false,
    sandbox: true,
  },
});
```

```ts
// electron/preload.ts
import { contextBridge, ipcRenderer } from 'electron';
contextBridge.exposeInMainWorld('api', {
  pickFile: () => ipcRenderer.invoke('pick-file'),
});
```

### Tauri 最小权限 + 能力白名单
- 通过 **`tauri.conf.json` allowlist** 精准授权（fs、shell、dialog 等分项）
- 仅用 `#[tauri::command]` 暴露必要函数；避免 `eval` 与任意来源加载
- 前端 `window.__TAURI__`、`invoke` 均走受控通路

```json
// src-tauri/tauri.conf.json（片段）
{
  "tauri": {
    "allowlist": {
      "dialog": { "open": true },
      "fs": { "readFile": true, "writeFile": false }
    },
    "security": { "csp": "default-src 'self'; script-src 'self';" }
  }
}
```

```rust
// src-tauri/src/main.rs
#[tauri::command]
fn pick_file() -> String { /* ... */ }

fn main() {
  tauri::Builder::default()
    .invoke_handler(tauri::generate_handler![pick_file])
    .run(tauri::generate_context!())
    .expect("error while running tauri app");
}
```

---

## 发布与自动更新

| 事项 | Electron | Tauri |
| --- | --- | --- |
| 构建/打包 | `electron-builder`/`electron-forge` | `tauri build`（内置 bundling） |
| 自动更新 | `electron-updater` 常见（需自建/三方更新源） | Tauri Updater（签名校验，需配置 feed） |
| 代码签名 | macOS `codesign` + notarize；Win `signtool` | 同上（Tauri 提供签名辅助与校验） |
| 差分/多渠道 | 社区方案丰富 | 官方方案精简，配置更“硬核” |

---

## 原生能力扩展

### Electron：Node-API / C++ 扩展
- 生态大量 **预编译** 二进制（如 `better-sqlite3`、`sharp`）
- 跨平台 ABI 管理需注意 Electron 版本与 Node-API 兼容

### Tauri：Rust 插件 / 命令
- 性能/并发极佳，直接复用 crates（如 `reqwest`/`tokio`/`serde`）
- 更易保证内存与线程安全；类型边界清晰

---

## 最小可用模板（可直接对比）

**Electron**

```ts
// main.ts
import { app, BrowserWindow, ipcMain, dialog } from 'electron';
import { join } from 'path';

const create = () => {
  const win = new BrowserWindow({
    width: 1200, height: 800,
    webPreferences: {
      preload: join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: true,
    },
  });
  win.loadURL(process.env.VITE_DEV_SERVER_URL || `file://${join(__dirname, 'index.html')}`);
};

app.whenReady().then(() => {
  ipcMain.handle('pick-file', async () => {
    const { filePaths } = await dialog.showOpenDialog({ properties: ['openFile'] });
    return filePaths[0] ?? '';
  });
  create();
});
```

```ts
// preload.ts
import { contextBridge, ipcRenderer } from 'electron';
contextBridge.exposeInMainWorld('api', {
  pickFile: () => ipcRenderer.invoke('pick-file'),
});
```

```ts
// renderer.ts
document.querySelector('#pick')!.addEventListener('click', async () => {
  const fp = await (window as any).api.pickFile();
  console.log('picked:', fp);
});
```

**Tauri**

```rust
// src-tauri/src/main.rs
#[tauri::command]
async fn pick_file() -> Option<String> {
  use rfd::FileDialog;
  FileDialog::new().pick_file().map(|p| p.display().to_string())
}

fn main() {
  tauri::Builder::default()
    .invoke_handler(tauri::generate_handler![pick_file])
    .run(tauri::generate_context!())
    .expect("run tauri");
}
```

```ts
// web/src/app.ts
import { invoke } from '@tauri-apps/api/tauri';
document.querySelector('#pick')!.addEventListener('click', async () => {
  const fp = await invoke<string | null>('pick_file');
  console.log('picked:', fp);
});
```

---

## 常见业务场景建议

- **重前端渲染/需要新特性（WebGPU、复杂 DevTools）**：Electron 更稳  
- **轻量工具/托盘常驻/系统集成（剪贴板/文件监控/通知）**：Tauri 更优  
- **需要高性能本地处理（音视频/图像/压缩/加密）**：Tauri（Rust）或 Electron + C++ 均可，前者上手 Rust，后者沿用既有 Node 生态  
- **要覆盖到移动端**：倾向 Tauri（v2 提供移动目标）

---

## 性能实战清单 ⚡️

- **Electron**
  - 仅开必要的 `BrowserWindow`，其余懒加载；避免隐藏窗口常驻占用
  - 重计算放 `worker_threads`/原生扩展；渲染线程保持轻量
  - 资源按需加载，禁用不必要的 Chromium 特性

- **Tauri**
  - Rust 侧用异步/线程池处理 IO 与重计算，`invoke` 仅传必要数据
  - 大文件用流式/分块；避免在前端一次性读写巨块 Buffer
  - 确保目标机器 WebView 版本充足（特别是 Windows WebView2）

---

## 典型坑点与排查 🕳️

- **Electron**
  - 忘记关掉 `nodeIntegration` / 不设 `contextIsolation` → 直接 RCE 风险
  - 主进程兜底 URL 校验缺失，`window.open`/`target=_blank` 外链未控
  - 原生扩展 ABI 与 Electron 版本不匹配，导致启动崩溃

- **Tauri**
  - `allowlist` 未开对应能力导致前端“无声失败”
  - Windows 未正确部署 WebView2 运行时，或版本过旧
  - 长任务阻塞 Rust 主线程；应移到新线程/异步任务

---

## 迁移与共存

- **UI 层**：React/Vue/Svelte 均可复用；主要迁移在 **原生能力桥接层**  
- **IPC 层对照**：Electron（`ipcMain`/`ipcRenderer` + Preload） ↔ Tauri（`invoke` + `#[command]`）  
- **策略**：先抽象“平台接口”（如 FS、Updater、Tray、Shell、Dialog），再各自实现

---

## 选型决策清单 ✅

- 需要完整 Chromium 能力 / 调试一致性 → **Electron**  
- 更在意包体/占用/默认安全边界、能写 Rust → **Tauri**  
- 老项目/插件依赖 Node-API 大量包 → **Electron**（成本低）  
- 新项目/工具类、对系统集成与性能敏感 → **Tauri**

---

## 参考实践模板

- Electron：`electron + vite + electron-builder + electron-updater`  
- Tauri：`tauri + vite + rust crates（tokio/serde/anyhow/rfd） + tauri-updater`

> 结论不是“谁胜谁负”，而是你的约束条件指向哪个最小风险面。能跑、好维护、可观测、可发布，才是王道。🏁

