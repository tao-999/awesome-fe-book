# 24.1 React Native / Expo 📱🚀

目标：用 **React Native（RN）+ Expo** 打出移动端“从 0 到上线”的全链路：脚手架、导航、状态、原生能力、OTA 更新、CI/CD、性能优化与可观测性。  
口号：**“Web 心智写 App，原生能力不降级。”**

---

## 0) TL;DR（三句话先背）🎯
1. **工作流选型**：快速开发就用 **Expo Managed**；要自定义原生/第三方原生库，用 **Expo Bare**（或 `prebuild` 后按需修改原生项目）。  
2. **性能基线**：开启 **Hermes** 引擎，使用 **Reanimated + Gesture Handler** 做流畅动效，列表走 **FlatList/SectionList** + 复用策略。  
3. **交付**：构建与分发用 **EAS Build/Submit**；热更新用 **EAS Update（OTA）**，用 **渠道（channels）+ 运行时版本（runtimeVersion）** 精准投放。

---

## 1) 架构脑图 🧠
- **JS 运行**：JS 引擎（Hermes）执行 React 代码。  
- **原生层**：iOS/Android 原生组件 & 能力。  
- **通信**：新架构 **JSI / TurboModules / Fabric**（更低延迟、更好并发）。  
- **Expo 模块**：统一包装的原生能力（Camera、Notifications、FileSystem…）。  
- **Bundler**：Metro（默认）→ 打包 JS Bundle；资源经 Asset 机制处理。  
- **EAS**：云构建（Build/Submit）+ 热更新（Update）。

> 记法：**“JSI 直连，Fabric 负责 UI，Turbo 管模块。”**

---

## 2) 起步（TypeScript + Expo Router）🏁

```bash
# 1) 创建项目（TS 模板）
npx create-expo-app@latest awesome-app -t
cd awesome-app

# 2) 运行（iOS 模拟器 / Android 模拟器 / Web）
npx expo start

# 3) 必装（动效与手势）
npx expo install react-native-reanimated react-native-gesture-handler

# 4) 文件路由（Expo Router）
npx expo install expo-router
```

`app/_layout.tsx`
```tsx
import { Stack } from 'expo-router';
export default function RootLayout() {
  return <Stack screenOptions={{ headerShown: false }} />;
}
```

`app/index.tsx`
```tsx
import { Link } from 'expo-router';
import { Text, View } from 'react-native';
export default function Home() {
  return (
    <View style={{ flex:1, alignItems:'center', justifyContent:'center' }}>
      <Text>👋 Hello RN + Expo</Text>
      <Link href="/details/42">详情</Link>
    </View>
  );
}
```

---

## 3) 导航与状态 🔌
- **导航**：推荐 **Expo Router**（文件式路由，底层 React Navigation）；或直接 React Navigation。  
- **状态**：接口数据 → **React Query**；全局少量状态 → **Zustand/Jotai**；复杂场景 → **Redux Toolkit**。  
- **表单**：React Hook Form + Zod（类型校验）。  

```bash
# 安装
npx expo install @tanstack/react-query
npm i zustand zod @hookform/resolvers react-hook-form
```

---

## 4) 原生能力（Expo 模块）📦
常用模块及用途：
- **expo-camera / expo-image-picker**：摄像头、相册  
- **expo-file-system**：文件读写  
- **expo-clipboard / expo-sharing**：剪贴板/分享  
- **expo-location / expo-sensors**：定位/传感器  
- **expo-notifications**：推送  
- **expo-secure-store**：**敏感数据加密存储**（Token/密钥）  
- **expo-sqlite**：本地数据库  
- **expo-image**：更高性能的图片组件

示例（安全存储 Token）：
```ts
import * as SecureStore from 'expo-secure-store';

export async function saveToken(t: string) {
  await SecureStore.setItemAsync('token', t, { keychainAccessible: SecureStore.AFTER_FIRST_UNLOCK });
}
export async function getToken() {
  return SecureStore.getItemAsync('token');
}
```

---

## 5) 网络与数据层 🌐
- **请求**：`fetch` 或 `axios`，写成 **API 层**（集中拦截器/错误映射）。  
- **缓存**：`React Query` + `persistQueryClient`（MMKV/AsyncStorage）。  
- **GraphQL**：Apollo/urql；**文件上传**注意 Content-Type 与进度回调。  

```ts
// React Query Provider
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
const qc = new QueryClient();
export const withQuery = (C: React.FC) => () => (
  <QueryClientProvider client={qc}><C/></QueryClientProvider>
);
```

---

## 6) 图片、字体与资源 📸
- 用 `expo-image`（解码与缓存更优），大图支持渐进加载与占位符。  
- 字体：`expo-font` 预加载，Splash → AppLayout 平滑切换。  
- 资源打包：小于阈值的资源随 Bundle，较大资源经 **EAS 资产托管/CDN**。

```ts
// app/_layout.tsx (字体预加载)
import { useFonts } from 'expo-font';
import { SplashScreen, Stack } from 'expo-router';
export default function Layout() {
  const [loaded] = useFonts({ Inter: require('../assets/Inter.ttf') });
  if (!loaded) return <SplashScreen />;
  return <Stack screenOptions={{ headerShown:false }} />;
}
```

---

## 7) 性能实践 🏎️
- **Hermes**：默认启用，启动更快、内存更省。  
- **列表**：`FlatList` 传 `keyExtractor/getItemLayout/maxToRenderPerBatch/windowSize`；超大列表考虑 `recyclerlistview`。  
- **动画**：**Reanimated**（UI 线程执行） + Gesture Handler；避免 JS 线程抖动。  
- **避免重渲染**：`memo`、`useCallback/useMemo`、拆分组件；样式对象常量化。  
- **图片**：合适尺寸、WebP/AVIF；缓存策略。  
- **监控**：接入 **Flipper**（网络/性能/日志）与自定义埋点（异常率、冷启时长）。

---

## 8) 推送通知（expo-notifications）🔔
```bash
npx expo install expo-notifications
```
```ts
import * as Notifications from 'expo-notifications';

export async function registerPushToken() {
  const { status } = await Notifications.requestPermissionsAsync();
  if (status !== 'granted') return null;
  const token = (await Notifications.getExpoPushTokenAsync()).data; // 服务端换 APNs/FCM 发送
  return token;
}
```
> 生产：iOS 需要 APNs 证书/密钥，Android 需 FCM；在 **EAS** 项目设置好凭证。

---

## 9) Deep Link & 路由 🔗
- `app.json`/`app.config.ts` 里配置 `scheme`；iOS **Universal Links**（`associatedDomains`），Android **intentFilters**。  
- **Expo Router** 自动兼容 `scheme://` 与 Universal Links，参数映射到文件路由。

```ts
// app.config.ts（示意）
export default { expo: {
  scheme: "awesomeapp",
  ios: { associatedDomains: ["applinks:links.awesome.app"] },
  android: { intentFilters: [{ action:["VIEW"], data:[{ scheme:"https", host:"links.awesome.app" }], category:["BROWSABLE","DEFAULT"] }] }
}};
```

---

## 10) OTA 更新（EAS Update）⚡
```bash
# 安装 CLI
npm i -g eas-cli

# 登录并配置
eas login
eas init

# 首次构建原生外壳
eas build -p ios
eas build -p android

# 之后推 OTA（根据 channel）
eas update --branch production --message "hotfix: fix crash"
```

`eas.json`（简例）
```json
{
  "cli": { "version": ">= 10.0.0" },
  "build": {
    "production": { "distribution": "store", "developmentClient": false }
  },
  "submit": {
    "production": { "ios": { "appleId": "you@example.com" } }
  }
}
```

`app.config.ts`（关键字段）
```ts
export default { expo: {
  runtimeVersion: { policy: "appVersion" }, // 运行时版本，确保二进制与 JS 兼容
  updates: { url: "https://u.expo.dev/<project-id>" } // EAS Update URL
}};
```

> 用 **channels/branches** 做分环境灰度：`staging`、`production`，必要时**回滚**到上一个更新。

---

## 11) 构建与上架（EAS Build/Submit）📦
- **Build**：云端编译出 `.ipa/.aab`；支持自定义 `prebuild` 钩子。  
- **Submit**：将构建产物直传 **App Store Connect / Google Play Console**。  
- **证书**：交给 EAS 托管或手动上传（公司合规选手动）。  

```bash
eas build --platform ios --profile production
eas submit --platform ios --latest
```

---

## 12) 原生扩展（Expo Modules / TurboModules）🧩
> 先尝试 **Expo 模块**（门槛低、跨端一致）；再考虑 TurboModules（更底层）。

```bash
# 生成 Expo 原生模块脚手架（示意）
npx expo generate module AwesomeNative
```

`AwesomeNativeModule.ts`
```ts
import { requireNativeModule } from 'expo-modules-core';
type AwesomeNativeType = { multiply(a: number, b: number): number };
export default requireNativeModule<AwesomeNativeType>('AwesomeNative');
```

iOS（Swift 简化）
```swift
import ExpoModulesCore

public class AwesomeNativeModule: Module {
  public func definition() -> ModuleDefinition {
    Name("AwesomeNative")
    Function("multiply") { (a: Double, b: Double) -> Double in a * b }
  }
}
```

Android（Kotlin 简化）
```kotlin
class AwesomeNativeModule : Module() {
  override fun definition() = ModuleDefinition {
    Name("AwesomeNative")
    Function("multiply") { a: Double, b: Double -> a * b }
  }
}
```

JS 调用：
```ts
import AwesomeNative from 'awesome-native';
AwesomeNative.multiply(6, 7); // 42
```

---

## 13) 测试与质量 🧪
- **单测**：Jest + React Native Testing Library（组件渲染/交互）。  
- **端到端**：Detox / Maestro。  
- **类型**：TypeScript 严格模式；公共 API 写 d.ts。  
- **静态检查**：ESLint + Prettier；可加 **eslint-plugin-react-native**。

---

## 14) 可观测性与崩溃收集 🧭
- **日志/网络**：Flipper 插件。  
- **崩溃**：Sentry/Firebase Crashlytics；Source Map/符号表上传。  
- **指标**：冷/热启动时长、FPS、JS 线程卡顿、页面首可交互。

---

## 15) Monorepo 与工程化 🧰
- **包管理**：pnpm/yarn workspaces；共享 `ui`/`utils` 包。  
- **路径别名**：`tsconfig.paths.json` + Babel module-resolver。  
- **Expo + Next.js** 共仓：共享 UI/业务逻辑，平台差异通过适配层解决。

---

## 16) 安全与合规 🛡️
- **敏感信息**：用 **SecureStore/Keychain/Keystore**；**不要**放 AsyncStorage。  
- **证书校验/Pinning**：需要时走原生库（Bare/自定义模块）。  
- **隐私权限**：在 `app.json` 写清 `NSCameraUsageDescription` 等；参见本书 §18.* 章节。

---

## 17) 反模式与纠偏 🧨
| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 所有状态塞 Redux | 模型臃肿 | 请求态用 React Query，局部用 Zustand/Jotai |
| 动画全在 JS 线程 | 丢帧卡顿 | 用 Reanimated（UI 线程）+ Gesture Handler |
| FlatList 不配优化项 | 滚动卡 | `getItemLayout/windowSize` + keyExtractor |
| 大图直传 | 包体/内存暴涨 | 合理尺寸 + 缓存 + 渐进加载 |
| 明文存 Token | 风险大 | SecureStore/Keychain/Keystore |
| 只做本地打包 | 上架可重复劳动 | 用 EAS Build/Submit + Update 灰度 |

---

## 18) 验收清单 ✅
- [ ] Hermes 开启，Reanimated & Gesture Handler 正常。  
- [ ] 列表/图片经优化；关键页面 ≥ 60FPS。  
- [ ] 推送、Deep Link、权限文案通过自测。  
- [ ] EAS Build/Submit 成功；EAS Update 渠道可灰度/回滚。  
- [ ] 崩溃收集与性能指标接通；Source Map 上传。  
- [ ] SecureStore 存敏感数据；依赖许可证清单可审计。

---

## 19) 练习 🏋️
1. 用 **Expo Router** 搭两页：列表页（分页 + 下拉刷新）、详情页（图片缓存 + Skeleton）。  
2. 用 **React Query** 做离线优先缓存；切网断网场景依然可读缓存。  
3. 接 **expo-notifications**，在 **staging** 渠道做 A/B 文案测试。  
4. 配一条 **EAS Update**：`staging → 10% → 50% → 100%` 灰度，并演练一键回滚。

---

**小结**：Expo 让 RN 的“开发—构建—分发—更新”一步到位；再把 **性能三件套（Hermes/Reanimated/列表优化）** 与 **工程三件套（EAS/监控/安全）** 落地，你的移动端就能既**快**又**稳**地迭代。🔥
