# 17.2 IndexedDB / 持久化存储 🗄️

目标：把 **离线数据、可扩展缓存、可靠写入** 做到稳、快、可演进。你将拿到：**最小可用封装、Schema 迁移套路、配额与持久化策略、二进制存储、同步与冲突化解**。

---

## 0) TL;DR 何时用 IndexedDB（IDB）？ 🎯
- **离线优先 / 草稿箱 / 队列**：表单草稿、购物车、待同步操作。
- **可控缓存**：API 结果、本地搜索索引、图片/音视频切片。
- **大对象**：`Blob` / `ArrayBuffer`，避免 Base64 膨胀。
- 不适合：强一致跨设备数据库、复杂多表 JOIN（可做索引与游标，但别当关系库）。

---

## 1) 基础概念速记 📚
- **Database 版本**：整库的 **Schema 由版本号控制**；升级在 `onupgradeneeded` 中迁移。
- **Object Store（表）**：键值仓库，键由 `keyPath` 或自增生成器提供。
- **Index（索引）**：`createIndex(name, keyPath, { unique, multiEntry })`；数组字段配 `multiEntry`。
- **Transaction（事务）**：`readonly` / `readwrite`，**同事务内原子性保证**。
- **Key Range**：`IDBKeyRange.only/bound/upperBound/lowerBound`；配合游标 `openCursor()` 扫描。

---

## 2) 原生 API 最小样例（无库版） 🧪
~~~ts
// db.ts
const DB_NAME = 'awesome';
const DB_VER  = 3;

export function openDB(): Promise<IDBDatabase> {
  return new Promise((resolve, reject) => {
    const req = indexedDB.open(DB_NAME, DB_VER);

    req.onupgradeneeded = () => {
      const db = req.result;

      // v1：文章与标签
      if (!db.objectStoreNames.contains('posts')) {
        const posts = db.createObjectStore('posts', { keyPath: 'id' });
        posts.createIndex('by_updatedAt', 'updatedAt');
        posts.createIndex('by_tags', 'tags', { multiEntry: true });
      }
      // v2：草稿箱
      if (!db.objectStoreNames.contains('drafts')) {
        db.createObjectStore('drafts', { keyPath: 'id' });
      }
      // v3：二进制资源
      if (!db.objectStoreNames.contains('blobs')) {
        db.createObjectStore('blobs', { keyPath: 'key' });
      }
    };

    req.onsuccess = () => resolve(req.result);
    req.onerror = () => reject(req.error);
    req.onblocked = () => console.warn('IDB upgrade blocked（有旧页面占用）');
  });
}

export async function putPost(post: { id: string; title: string; updatedAt: number; tags: string[] }) {
  const db = await openDB();
  return txPut(db, 'posts', post);
}

export async function getRecentPosts(limit = 20) {
  const db = await openDB();
  return new Promise<any[]>((resolve, reject) => {
    const tx = db.transaction('posts', 'readonly');
    const store = tx.objectStore('posts');
    const idx = store.index('by_updatedAt');
    const range = IDBKeyRange.lowerBound(0);
    const req = idx.openCursor(range, 'prev'); // 倒序
    const out: any[] = [];
    req.onsuccess = () => {
      const cur = req.result;
      if (cur && out.length < limit) { out.push(cur.value); cur.continue(); }
      else resolve(out);
    };
    req.onerror = () => reject(req.error);
  });
}

function txPut<T>(db: IDBDatabase, store: string, value: T) {
  return new Promise<void>((resolve, reject) => {
    const tx = db.transaction(store, 'readwrite');
    tx.objectStore(store).put(value);
    tx.oncomplete = () => resolve();
    tx.onerror = () => reject(tx.error);
  });
}
~~~

> 小心：**升级被阻塞**常见于多标签页。可用 `BroadcastChannel` 通知老页面刷新，或在 UI 提示“有新版本”。

---

## 3) 用库更省心：`idb` / `Dexie` 🧰
- **idb**：轻薄 Promise 包装，无 ORM，贴近原生，适合自己管迁移。
- **Dexie**：声明式表结构与索引、强力批量/查询、内置升级钩子，开发体验最佳。

~~~ts
// Dexie 快速上手
import Dexie, { Table } from 'dexie';

export interface Post { id: string; title: string; updatedAt: number; tags: string[] }

class AppDB extends Dexie {
  posts!: Table<Post, string>;
  constructor() {
    super('awesome');
    this.version(3).stores({
      posts: 'id, updatedAt, *tags',   // *tags => multiEntry
      drafts: 'id',
      blobs: 'key'
    });
  }
}
export const db = new AppDB();

// 查询
export const recent = (limit=20) =>
  db.posts.orderBy('updatedAt').reverse().limit(limit).toArray();
~~~

---

## 4) 数据建模与 Schema 迁移 🧱
- **键选择**：资源天然主键（如 `id`），或自增；跨端合并建议使用 **ULID/UUID v7**。
- **索引**：常用排序字段建单值索引；数组字段用 `multiEntry`；复合查询拆分为**索引 + 过滤**。
- **迁移**：**只在升级中改 Schema**；版本号单向增长，迁移函数**幂等**。

~~~ts
// Dexie 迁移示例：v2 -> v3 补字段
this.version(3).upgrade(async (tx) => {
  await tx.table('posts').toCollection().modify(p => {
    if (!('tags' in p)) p.tags = [];
  });
});
~~~

---

## 5) 事务与并发 🔒
- **短事务**：把所有操作放进**单次事务**，减少锁持有时间；不要在事务中做长异步。
- **跨表一致**：同事务里对多个 store `readwrite`；失败会整体回滚。
- **冲突与幂等**：写入使用**乐观版本**（`version`/`updatedAt`），或在同步层处理冲突（见 §10）。

---

## 6) 存储配额 / 持久化（别被浏览器“回收站”送走） 🧯
- **估算容量**：`navigator.storage.estimate()` 返回 `quota/usage`。
- **申请持久化**：`await navigator.storage.persist()`，成功后**减少被逐出概率**（用户手动清理仍会删）。
- **大小控制**：按库/表设 **LRU** 清理；大对象分块；避免把 JSON 转 Base64（体积 +33%）。
- **Safari / 隐私模式**：可能返回 **0 KB** 或易被清空；关键数据要**云端备份**或导出。

~~~ts
export async function ensurePersisted() {
  if (navigator.storage && 'persist' in navigator.storage) {
    const persisted = await navigator.storage.persisted?.();
    if (!persisted) await navigator.storage.persist();
  }
}

export async function quotaInfo() {
  const est = await navigator.storage.estimate();
  const usedMB  = (est.usage || 0) / 1024 / 1024;
  const quotaMB = (est.quota || 0) / 1024 / 1024;
  return { usedMB, quotaMB, ratio: usedMB / quotaMB };
}
~~~

---

## 7) 二进制存储（图片/音视频/加密块） 🧩
- 直接存 `Blob` / `ArrayBuffer`，**别用 Base64**。
- 大文件：切片（如 1–4MB）+ 纪录清单（manifest），必要时 **Range** 合并。
- 加密：WebCrypto `AES-GCM`；记得把 **Nonce** 与 **版本**写入 metadata。

~~~ts
// 存图
export async function putBlob(key: string, blob: Blob) {
  const db = await openDB();
  const tx = db.transaction('blobs', 'readwrite');
  await tx.objectStore('blobs').put({ key, blob, createdAt: Date.now() });
  await tx.complete;
}

// 取图 -> ObjectURL
export async function getBlobURL(key: string) {
  const db = await openDB();
  const rec: any = await new Promise((resolve, reject) => {
    const tx = db.transaction('blobs', 'readonly');
    const req = tx.objectStore('blobs').get(key);
    req.onsuccess = () => resolve(req.result);
    req.onerror = () => reject(req.error);
  });
  return rec ? URL.createObjectURL(rec.blob as Blob) : null;
}
~~~

---

## 8) 查询与游标 🧭
- 范围：`IDBKeyRange.bound(a, b, /*openLower*/false, /*openUpper*/true)`。
- 方向：`next/prev/nextunique/prevunique`。
- 计数：`store.count()`；分页：记录上次游标 Key，继续 `continue(key)`。

~~~ts
// 根据标签查最新 50 条
export async function queryByTag(tag: string, limit=50) {
  const db = await openDB();
  return new Promise<any[]>((resolve, reject) => {
    const tx = db.transaction('posts', 'readonly');
    const idx = tx.objectStore('posts').index('by_tags');
    const range = IDBKeyRange.only(tag);
    const out: any[] = [];
    const curReq = idx.openCursor(range, 'prev');
    curReq.onsuccess = () => {
      const c = curReq.result;
      if (c && out.length < limit) { out.push(c.value); c.continue(); }
      else resolve(out);
    };
    curReq.onerror = () => reject(curReq.error);
  });
}
~~~

---

## 9) 性能建议 ⚡
- **批量**：一次事务里 `put` 多条；Dexie 的 `bulkPut` 更快。
- **去重写**：读命中则跳过写；或比较 `updatedAt/version`。
- **懒加载**：列表只取必要字段（可以把冗余列表字段单独存 store）。
- **压缩**：文本可用 `CompressionStream('gzip')`；注意兼容性与 CPU 成本。

---

## 10) 离线同步与冲突解决 🔄
**通用策略**（BFF 有幂等）：
1) 本地每次写入都记录 **`op`**（增/改/删）+ **客户端时间戳** + **幂等键**。
2) 联网后按顺序 **重放**到服务器（Background Sync / 前台重试）。
3) 服务器返回 **资源新版本**；本地再 **对账合并**。
4) 冲突策略：  
   - **LWW（最后写入胜出）**：简单但可能丢改动。  
   - **字段级合并**：如富文本基于段落/区块合并。  
   - **CRDT**：协作类（Yjs/Automerge）可把快照存 IDB、网络传增量。

---

## 11) 与 Service Worker 的配合 🛠️
- **SW 读缓存**：API GET → SWR；IDB 存副本；页面优先本地回显。
- **写入队列**：页面失败的写请求入 IDB 队列；SW `sync` 事件重放；幂等键避免重复扣费。
- **消息桥**：`postMessage` 告知页面“同步完成/失败”，刷新视图。

---

## 12) 清理 / 备份 / 导入导出 🧹
- **LRU**：在写入时维护 `lastAccess`，超额时删最旧。
- **导出**：游标 → `ReadableStream` → 文件（JSONL / NDJSON）；大对象分桶导出。
- **导入**：分批 `bulkPut`，限制每批 500–2000 条，避免长事务阻塞 UI。

---

## 13) 测试与诊断 🔬
- 单元：Dexie 提供内存适配；或在 `jest-environment-jsdom` 下跑。
- 端到端：Cypress/Playwright 断网测试（`browserContext.setOffline(true)`），验证离线操作与恢复重放。
- 观测：记录 **失败率、重放成功率、回收清理次数、配额占比**。

---

## 14) 反模式与纠偏 🧨
| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 每次打开都 `deleteDatabase` | 数据丢失 | 版本迁移；保留历史，幂等升级 |
| Base64 存大图 | 体积 +33% | 直接 `Blob` / `ArrayBuffer` |
| 事务里做长异步 | 卡死/超时 | 事务短平快；大活拆批 |
| 没有 LRU | 被浏览器清空 | `estimate()` + 上限 + 定期清理 |
| 只本地不备份 | Safari/隐私模式丢数据 | 关键数据云端同步或导出 |
| 不处理 `blocked` | 升级失败 | 提示刷新；BroadcastChannel 驱离旧页面 |
| 一个 Store 放一切 | 查询慢 | 分表 + 索引；列表冗余字段单独存 |

---

## 15) 验收清单 ✅
- [ ] 有 **Schema 版本**与 **幂等迁移**；升级阻塞有 UI 提示。  
- [ ] 读写封装 Promise 化；批量 API；错误上报。  
- [ ] 图片/二进制走 `Blob`；大对象有**分块**与**清单**。  
- [ ] 配额监控与 **持久化申请**；LRU 清理可用。  
- [ ] 离线写入进入 **队列**；联网后重放成功；冲突策略已定义。  
- [ ] 有导入/导出路径；E2E 断网恢复用例通过。  

---

## 16) 练习 🏋️
1. 为“文章列表”实现 **SWR 缓存**：首屏离线可读，联网后静默更新。  
2. 实现一个 **图片 Blob 缓存**：优先读 IDB，未命中再拉网并回填；配 LRU 200 条。  
3. 写一个 **写入队列**（Dexie + Background Sync），模拟断网下提交表单，恢复网络自动重放；服务端使用 **Idempotency-Key**。  

---

**小结**：IndexedDB 是浏览器里的“迷你持久层”。把 **版本迁移、批量事务、配额与持久化、二进制、同步与幂等** 这几件事捋顺，你的 Web App 就能在没有网络时一样靠谱，在有网络时更快更省。🚀
