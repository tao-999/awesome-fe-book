# 32.4 地图 / GIS / 可视化地图 🗺️📍📊

> 这章把“地图”从玩具变成生产：**底层坐标系 → 切片与瓦片 → 渲染引擎选型 → 数据管线（生成矢量瓦片） → 可视化图层（聚合/热力/等值面/Hexbin） → 实时轨迹与离线缓存 → 3D 地形**。给你能**开箱即跑**的代码与一套**可扩展的工程骨架**。

---

## 0）心法与落地目标

- **认清三件事**：坐标（CRS）怎么来 → 数据怎么切（Tile/Vektor/Raster） → 客户端怎么画（WebGL/Canvas）。  
- **生产级原则**：**数据驱动样式**、**CDN 瓦片**、**Worker 并行**、**端权界面（Token/域名白名单）**、**可观测**（瓦片命中率/渲染时长）。  
- **体感指标**：首屏可交互 < 2s，地图平移/缩放帧率 ≥ 50fps，10 万点交互不卡，热力/聚合随级别切换自然。

---

## 1）底层概念速查（不迷路）

- **CRS（坐标参考系）**：  
  - Web 常用 **EPSG:3857 Web Mercator**（单位米，接近圆柱投影）；精确测距用 **EPSG:4326 WGS84**（经纬度）。  
  - **中国系**：WGS84（国际 GPS）、**GCJ-02**（国测火星坐标）、**BD-09**（百度偏移）。数据源混用时要**显式转换**（见 §8）。  
- **Tiles**：  
  - **XYZ**（z/x/y）、**WMTS/WMS**（OGC 标准）。  
  - **Raster**（位图） vs **Vector Tile (MVT)**（几何+属性，客户端渲染，样式可变）。  
- **数据格式**：GeoJSON/TopoJSON（矢量）、Shapefile（传统）、GeoTIFF（栅格）、MBTiles/PMTiles（离线容器）。

---

## 2）渲染引擎选型（啥场景用谁）

| 引擎 | 核心能力 | 适用场景 |
|---|---|---|
| **MapLibre GL JS** | WebGL 矢量瓦片 + Mapbox Style | 通用地图、数据驱动样式、3D 地形 |
| **Leaflet** | 轻量、Raster/简单矢量 | 低门槛、静态底图 + 标注 |
| **OpenLayers** | WMS/WMTS/投影/测绘强 | 政企/多投影/OGC 深度 |
| **deck.gl** | WebGL 大数据叠加层（点/线/网格/热力） | 10万~百万几何可视化 |
| **CesiumJS** | 3D 球体、倾斜摄影/3D Tiles | 三维地球/地形/城市级模型 |

> 通常组合：**MapLibre** 做底图 + **deck.gl** 叠加数据层；有测绘需求用 **OpenLayers**；三维真爱上 **Cesium**。

---

## 3）最小可跑：MapLibre + 开放样式（矢量瓦片）

```html
<!-- index.html -->
<link href="https://unpkg.com/maplibre-gl@3.6.0/dist/maplibre-gl.css" rel="stylesheet" />
<div id="map" style="position:fixed;inset:0"></div>
<script type="module">
  import maplibregl from 'https://unpkg.com/maplibre-gl@3.6.0/dist/maplibre-gl.js';

  const map = new maplibregl.Map({
    container: 'map',
    style: 'https://demotiles.maplibre.org/style.json', // 任意 Mapbox Style JSON
    center: [121.4737, 31.2304], // 上海
    zoom: 10,
    hash: true // URL 同步：可分享状态
  });

  map.addControl(new maplibregl.NavigationControl(), 'top-right');
  map.on('load', () => {
    // 加一层 GeoJSON 点并随 zoom 调整半径（数据驱动）
    map.addSource('pois', { type: 'geojson', data: '/data/poi.geo.json' });
    map.addLayer({
      id: 'poi',
      type: 'circle',
      source: 'pois',
      paint: {
        'circle-radius': ['interpolate', ['linear'], ['zoom'], 5, 2, 12, 6],
        'circle-color': ['match', ['get', 'type'], 'cafe', '#ff7f50', 'park', '#22c55e', '#3b82f6'],
        'circle-opacity': 0.8
      }
    });
  });
</script>
```

**Leaflet 最小版（Raster + 标注）**

```html
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
<div id="map" style="height:100vh"></div>
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
  const map = L.map('map').setView([31.23, 121.47], 12);
  L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', { maxZoom: 19 }).addTo(map);
  L.marker([31.234,121.48]).addTo(map).bindPopup('Hello, 骚哥');
</script>
```

---

## 4）矢量瓦片生产管线（从 GeoJSON 到 CDN）

> 你需要**可复算**与**可回滚**的数据链路：源数据 → 清洗/简化 → 切片 → 存储 → 发布/回滚。

1) **清洗/简化**（拓扑保持 & 按级别简化）  
2) **切片**：`tippecanoe` 生成 **MBTiles**（MVT）  
3) **发布**：Tileserver-GL/tileserver-mbtiles 或转 **PMTiles** 放 CDN（HTTP Range/单文件）

```bash
# 1) 清洗（示意）：mapshaper / ogr2ogr
mapshaper input.geojson -simplify 10% keep-shapes -o clean.geojson

# 2) 切片：按属性分层、按要素类型设最小/最大级别
tippecanoe -o city.mbtiles clean.geojson \
  -Z 4 -z 14 -bg="#fff" \
  --drop-densest-as-needed --coalesce --coalesce-small --extend-zooms-if-still-dropping \
  --layer=city

# 3) 发布：Tileserver-GL 或 PMTiles
npx tileserver-gl city.mbtiles
# 或转换为 PMTiles（单文件，CDN 友好）
pmtiles convert city.mbtiles city.pmtiles
```

**样式（Mapbox Style JSON）**用 Git 管控版本，切换 **style URL** 即灰度/回滚。

---

## 5）可视化图层套路（点/线/面/栅格）

### 5.1 聚合（集群）与热力

- 小缩放级别显示**集群**，放大自动**散开**；点数巨多时**服务端预聚合**或 **Supercluster** + **Web Worker**。

```ts
// supercluster + worker 简例
import Supercluster from 'supercluster';
const idx = new Supercluster({ radius: 60, maxZoom: 16 }).load(features);
const clusters = idx.getClusters(bbox /* [w,s,e,n] */, zoom);
```

- 热力（MapLibre `heatmap` 或 deck.gl `HeatmapLayer`）：对**连续型强度**友好，避免对“只有有/无”的离散事件滥用热力。

### 5.2 等值面 / 等高线（Isoline / Contour）

- `turf.isobands / isolines` 从栅格/点插值生成；或服务端做 **IDW/Kriging**，前端只渲染等值矢量。

### 5.3 Hexbin/网格统计（deck.gl）

```ts
import { HexagonLayer } from '@deck.gl/aggregation-layers';
new HexagonLayer({
  id: 'hex',
  data, getPosition: d => [d.lng, d.lat],
  radius: 200, // 米（在 WebMercator 下）
  extruded: true, elevationScale: 2
});
```

### 5.4 栅格（GeoTIFF/NDVI/热度底图）

- `geotiff.js` 在 Worker 中读取栅格波段拼色；大量瓦片走 **WMTS** 或 **COG（Cloud Optimized GeoTIFF）** + range 请求。

---

## 6）性能圣经（十万点也丝滑）

- **矢量瓦片优先**：把体量留给服务端切片；客户端只画视域数据。  
- **Worker 并行**：聚合/空间索引/插值放 Worker；主线程只管交互与绘制。  
- **分级渲染**：缩放级别低时只画聚合/骨架；进入高 zoom 才画细节。  
- **表达式样式**：MapLibre 的 `["interpolate", ["zoom"], ...]` 控制半径/线宽/透明。  
- **空间索引**：可交互要素建 R-Tree/Flatbush，`hitTest` O(log n)。  
- **贴图与文字**：禁止大段 `text-field` 实时变化（触发 layout），用 `symbol-sort-key` 控制优先级。  
- **图层拆分**：不同绘制类型分层（线/面/标注），减少重排。

---

## 7）交互与状态管理（React/Vue 稳态落法）

- **URL 哈希同步**（`#15.8/121.47/31.23`）：可分享、可回放。  
- **选中态**：**hover 光标**用命中测试（索引），**选中**通过要素 `id` 高亮 + 属性面板。  
- **绘制/编辑**：Mapbox-GL-Draw / OpenLayers Draw；复杂编辑走**临时图层 + 提交事务**。  
- **地理围栏**：`pointInPolygon`（turf），大规模围栏做 R-Tree + 多边形预处理（三角剖分/凸分解）。

---

## 8）坐标转换（WGS84 / GCJ-02 / BD-09 / 自定义投影）

```ts
// proj4 示例：任意投影互转
import proj4 from 'proj4';
proj4.defs('EPSG:3857', '+proj=merc +lon_0=0 +k=1 ...');
proj4.defs('EPSG:4326', '+proj=longlat +datum=WGS84 +no_defs');
const [x, y] = proj4('EPSG:4326','EPSG:3857',[121.47,31.23]);

// GCJ-02/BD-09 ↔ WGS84：使用公开实现（注意法律合规与精度误差）
```

> **合规提醒**：在部分地区，公开提供**纠偏后高精度**底图/坐标转换需合规资质与授权；内部应用也要遵守当地法规与许可条款。

---

## 9）实时轨迹（运输/骑手/IoT）

- **上行**：设备 → MQTT/HTTP（Protobuf 压缩）→ 后端；  
- **清洗/抽稀**：Douglas-Peucker / Visvalingam；加**地图匹配**（OSRM/Valhalla map-matching）。  
- **下行**：客户端 WebSocket 订阅，`map.easeTo` + `Marker.setLngLat` 平滑插值。  
- **轨迹分段**：按停靠与异常切片；支持回放（时间轴 + 速度控制）。

---

## 10）搜索/路径/等时圈（服务接口）

- **地理编码**：Nominatim/Photon（OSM）或商业服务（Mapbox/高德/腾讯）。  
- **路径规划**：OSRM/GraphHopper/Valhalla（驾/步/骑，避让参数）；  
- **等时圈（Isochrone）**：基于路网耗时的可达区域（外呼/配送规划神器）。

---

## 11）离线与弱网策略

- **离线底图**：MBTiles/PMTiles（单文件），首启预拉热点区域；  
- **Service Worker**：瓦片缓存（Cache-Storage），按 z/x/y 设配额与淘汰策略（LRU）。  
- **断点续传**：大数据集分层分片；失败重试 + 指数退避。  
- **移动端**：控制 memory footprint（禁用太多 label/图片符号）。

---

## 12）三维地形与城市模型

- **MapLibre**：`terrain` + `raster-dem` 实现地形起伏（叠加阴影/坡度着色）。  
- **Cesium**：3D Tiles / glTF / 地形数据（Cesium ION 或自建）。  
- **倾斜摄影**：大体量数据用 3D Tiles 分层与裁剪；相机/拾取优化与瓦片裁剪阈值调参是关键。

---

## 13）安全与隐私

- **Token 范围**：地图服务密钥**域名绑定** + 只读权限 + 请求速率；  
- **Referer/CORS 白名单**：防盗链；  
- **坐标模糊**：对用户位置**模糊化**（如 3 位小数 ≈ 100m），敏感场景按需加噪或延迟显示；  
- **合规版权**：底图/路网/POI 的**许可证**要看清（OSM/商业提供商）；切片/样式再分发要遵守条款。

---

## 14）可观测与测试

- **前端指标**：`map_load_ms`、`tiles_requested/loaded/failed`、`fps`、`frame_time_p95`。  
- **服务端**：`tile_hit_ratio`（CDN 命中率）、`tile_gen_duration`、`cache_miss_top_paths`。  
- **回放**：保存 **viewState（lon/lat/zoom/bearing/pitch）** 与查询参数，问题复现一步到位。  
- **视觉回归**：固定样式/相机 → 截图对比（Playwright + pixelmatch）。

---

## 15）项目骨架（前后端协作版）

```
maps/
  ├─ web/                     # 前端
  │   ├─ components/
  │   │   ├─ MapView.tsx      # MapLibre 初始化 + 控件
  │   │   ├─ Layers/          # 业务图层（聚合/热力/Hexbin/Route）
  │   │   ├─ InspectPanel.tsx # 属性检查/选择状态
  │   │   └─ DeckOverlay.tsx  # deck.gl 叠加
  │   ├─ workers/
  │   │   ├─ cluster.worker.ts # Supercluster/Flatbush
  │   │   └─ raster.worker.ts  # GeoTIFF 解码/调色
  │   └─ styles/map-style.json # Mapbox Style（版本化）
  ├─ tiles/
  │   ├─ sources/             # 原始数据(GeoJSON/Shape/GeoTIFF)
  │   ├─ pipeline/            # 清洗/简化/切片脚本（tippecanoe）
  │   ├─ dist/                # MBTiles/PMTiles 产物
  │   └─ server/              # tileserver-gl / pmtiles server
  └─ services/
      ├─ routing/             # OSRM/Valhalla 适配
      ├─ geocoding/           # Nominatim 代理
      └─ realtime/            # 轨迹 WebSocket
```

---

## 16）实战代码片段合集

**A）deck.gl 叠加到 MapLibre**

```ts
import { MapboxOverlay } from '@deck.gl/mapbox/typed';
import { ScatterplotLayer } from '@deck.gl/layers/typed';

const overlay = new MapboxOverlay({ interleaved: true, layers: [] });
map.addControl(overlay);

overlay.setProps({
  layers: [
    new ScatterplotLayer({
      id: 'pts',
      data: '/data/points.json',
      getPosition: d => [d.lng, d.lat],
      getRadius: d => Math.sqrt(d.value) * 10,
      getFillColor: d => [255, 99, 71, 180],
      pickable: true
    })
  ]
});
map.on('click', e => {
  const info = overlay.pickObject({ x: e.point.x, y: e.point.y });
  if (info?.object) showPanel(info.object);
});
```

**B）GeoTIFF 栅格渲染（Worker）**

```ts
// raster.worker.ts
import { fromUrl } from 'geotiff';
self.onmessage = async (e) => {
  const t = await fromUrl(e.data.url);
  const image = await t.getImage();
  const raster = await image.readRasters(); // Float32Array[]
  // 伪彩色映射 → ImageData → postMessage(transfer)
};
```

**C）GCJ-02 ↔ WGS84（示意）**

```ts
// 注意：实际实现请使用经过审阅的库，并遵守相关法律/许可。
export function gcj2wgs(lng:number, lat:number): [number, number] {
  // ... 省略，放置占位
  return [lng, lat];
}
```

---

## 17）Checklist（上线前最后一眼）✅

- [ ] 数据来源/许可证清晰；CRS 一致或转换明确  
- [ ] 瓦片：矢量优先；CDN 命中率 ≥ 90%；样式版本可回滚  
- [ ] 前端：Worker 聚合/索引；层级渲染策略；URL 状态同步  
- [ ] 轨迹：抽稀 + 地图匹配 + 平滑；弱网重试  
- [ ] 隐私：位置模糊/脱敏；Token 域名绑定与限速  
- [ ] 观测：map_load、tile_hit_ratio、错误与 FPS 监控  
- [ ] 大屏/移动兼容：文字避让、低端机降级（Raster/禁阴影）  
- [ ] 3D（可选）：地形/3D Tiles 性能与相机参数调优

---

### 小结

地图不是“背景图”，而是**数据产品的舞台**。把**CRS/瓦片/引擎/样式/数据管线**这五环打通，再加上**Worker 并行与 CDN 瓦片**，你的可视化地图就能在**精度、性能、可维护**之间达到甜点位。接下来你可以把它接入**RAG 检索（地理范围）**、**时空分析**、**三维城市**，让地图成为你应用的“地理操作系统”。🚀
