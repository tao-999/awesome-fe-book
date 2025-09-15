# 24.2 Flutter for 前端工程师 🦋📱

目标：把 Flutter 变成前端同学的“**跨端大杀器**”，从心智映射 → 工程化 → 性能稳定 → 上架交付，一条龙搞定。  
口号：**“用 Widget 写像素，用 Dart 写性能。”**

---

## 0) TL;DR（三条就够上手）🎯
1. **心智映射**：React 组件 ≈ Flutter **Widget**；`setState` ≈ `StatefulWidget.setState`；路由推荐 **go_router**；状态推荐 **Riverpod**。  
2. **性能基线**：**Impeller/Skia** 渲染；避免在 `build()` 做重活；长列表用 `ListView.builder`；重绘隔离用 `RepaintBoundary`；重计算丢到 **Isolate**。  
3. **交付**：多环境 **flavors**（dev/stg/prod）+ **CI（GitHub Actions/Codemagic）**；崩溃/日志接 **Crashlytics/Sentry**；迭代靠 **Remote Config + Feature Flags** 灰度。

---

## 1) 前端→Flutter 心智映射 🧠

| Web/React | Flutter 对应 | 备注 |
|---|---|---|
| Component（函数/类） | **Widget（Stateless/Stateful）** | 所有 UI 皆 Widget（甚至路由/主题） |
| JSX | Widget 树 | 没模板语法，纯 Dart 代码 |
| setState/useState | `StatefulWidget.setState()` / Riverpod | 更推荐业务状态用 Riverpod |
| Context/Provider | **InheritedWidget** / Provider / **Riverpod** | Riverpod 编译期安全更高 |
| Router (SPA) | **go_router**（Router 2.0） | 深链/守卫/嵌套路由优雅 |
| CSS/Flex | **Row/Column/Flex/Expanded** | 约束（constraints）> 尺寸（size）> 布局（layout） |
| Canvas/WebGL | **CustomPainter / Scene / Shader** | 有 `dart:ui`，可写自定义绘制 |
| 生命周期 | `initState / didChangeDependencies / dispose` | `build` 纯函数（尽量无副作用） |
| 包管理 | `pubspec.yaml` | `flutter pub add xxx` |

---

## 2) 起步（TypeScript 同学的“Dart 速通”）🏁

```bash
# 安装 SDK 略（确保 flutter doctor 全绿）
flutter create awesome_app
cd awesome_app
flutter run -d chrome    # or -d ios / -d android / -d macos
```

最小骨架：
```dart
void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      theme: ThemeData(useMaterial3: true),
      home: const HomePage(),
    );
  }
}

class HomePage extends StatefulWidget { const HomePage({super.key}); @override State<HomePage> createState() => _HomePageState(); }
class _HomePageState extends State<HomePage> {
  int count = 0;
  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: const Text('Hello Flutter')),
    body: Center(child: Text('count = $count')),
    floatingActionButton: FloatingActionButton(
      onPressed: () => setState(() => count++), child: const Icon(Icons.add)),
  );
}
```

---

## 3) 布局三板斧：约束/Flex/Stack 🧱

- **约束优先**：父给子约束（最大/最小尺寸）→ 子在约束内决定大小 → 父据此定位。  
- **Flex 系列**：`Row/Column` + `Expanded/Flexible` 控制分配；栅格用 `Wrap`/`GridView`.  
- **叠放**：`Stack + Positioned` 做悬浮/贴边。  

示例：
```dart
Widget dashboard() => Padding(
  padding: const EdgeInsets.all(16),
  child: Column(
    children: [
      Row(children: const [
        Expanded(child: _Card(title: 'PV', value: '12.3k')),
        SizedBox(width: 12),
        Expanded(child: _Card(title: 'UV', value: '4.6k')),
      ]),
      const SizedBox(height: 12),
      Expanded(child: _ChartPlaceholder()),
    ],
  ),
);
```

---

## 4) 设计系统与主题 🎨

```dart
final theme = ThemeData(
  useMaterial3: true,
  colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF6750A4)),
  textTheme: const TextTheme(titleLarge: TextStyle(fontWeight: FontWeight.w600)),
);
```

- **暗黑**：`theme` + `darkTheme` + `ThemeMode.system`。  
- **Design Tokens**：自建 `AppTheme` 单例/`extension` 输出 spacing/shape 等语义变量。  
- **国际化**：见 §12。

---

## 5) 路由（go_router，强烈推荐）🗺️

```yaml
# pubspec.yaml
dependencies:
  go_router: ^14.0.0
```

```dart
final router = GoRouter(
  routes: [
    GoRoute(path: '/', builder: (c, s) => const HomePage()),
    GoRoute(path: '/detail/:id', builder: (c, s) => DetailPage(id: s.pathParameters['id']!)),
  ],
  redirect: (c, s) {
    // 鉴权守卫示例
    final authed = false; 
    if (!authed && s.matchedLocation != '/login') return '/login';
    return null;
  },
);
```

```dart
MaterialApp.router(routerConfig: router);
```

---

## 6) 状态管理（Riverpod 是真香）🌊

```yaml
dependencies:
  flutter_riverpod: ^2.5.0
```

```dart
// provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

final counterProvider = StateProvider<int>((ref) => 0);

class TodosNotifier extends AsyncNotifier<List<String>> {
  @override Future<List<String>> build() async => ['foo', 'bar'];
  Future<void> add(String t) async => state = AsyncData([...state.value!, t]);
}
final todosProvider = AsyncNotifierProvider<TodosNotifier, List<String>>(() => TodosNotifier());
```

```dart
// main.dart
void main() => runApp(const ProviderScope(child: MyApp()));

// 页面里
Consumer(builder: (context, ref, _) {
  final count = ref.watch(counterProvider);
  return Text('count = $count');
});

// 更新
ref.read(counterProvider.notifier).state++;
```

> Riverpod 优点：编译期安全、无需 BuildContext、易测试；复杂业务可用 `StateNotifier/AsyncNotifier`。

---

## 7) 网络/数据/序列化 🌐

```yaml
dependencies:
  dio: ^5.5.0
  freezed_annotation: ^2.4.1
  json_annotation: ^4.9.0
dev_dependencies:
  build_runner: ^2.4.0
  freezed: ^2.5.0
  json_serializable: ^6.8.0
```

```dart
// model.dart
import 'package:freezed_annotation/freezed_annotation.dart';
part 'model.freezed.dart'; part 'model.g.dart';

@freezed
class User with _$User {
  const factory User({required int id, required String name}) = _User;
  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}
```

```dart
// api.dart
final dio = Dio(BaseOptions(baseUrl: 'https://api.example.com', connectTimeout: const Duration(seconds: 5)));

Future<User> fetchUser(int id) async {
  final r = await dio.get('/users/$id');
  return User.fromJson(r.data as Map<String, dynamic>);
}
```

> 缓存：`dio_cache_interceptor` / `flutter_query`；离线：`isar/hive` 本地存储。

---

## 8) 表单与校验 📋

```yaml
dependencies:
  flutter_form_builder: ^9.3.0
  form_builder_validators: ^9.1.0
```

```dart
FormBuilder(
  key: _formKey,
  child: Column(children: [
    FormBuilderTextField(name: 'email', validator: FormBuilderValidators.email()),
    FormBuilderTextField(name: 'pwd', obscureText: true, validator: FormBuilderValidators.minLength(6)),
    ElevatedButton(onPressed: () { if (_formKey.currentState!.saveAndValidate()) {/*submit*/} }, child: const Text('Submit')),
  ]),
);
```

---

## 9) 本地存储与安全 🔐

- **首选**：`shared_preferences`（轻量 KV）、`hive/isar`（本地 DB）。  
- **敏感信息**：`flutter_secure_storage`（Keychain/Keystore），不要放平明文本。  
- **配置**：远端配置/灰度推荐 `firebase_remote_config` / 自建配置服务 + Feature Flag。

---

## 10) 原生能力与通信（Platform Channels）🧩

Dart 侧：
```dart
const channel = MethodChannel('awesome/native');

Future<String> deviceName() async {
  final r = await channel.invokeMethod<String>('deviceName');
  return r ?? 'unknown';
}
```

Android（Kotlin）：
```kotlin
class MainActivity: FlutterActivity() {
  private val CHANNEL = "awesome/native"
  override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
    super.configureFlutterEngine(flutterEngine)
    MethodChannel(flutterEngine.dartExecutor.binaryMessenger, CHANNEL).setMethodCallHandler { call, result ->
      if (call.method == "deviceName") result.success(Build.MODEL) else result.notImplemented()
    }
  }
}
```

iOS（Swift）：
```swift
let channel = FlutterMethodChannel(name: "awesome/native", binaryMessenger: controller.binaryMessenger)
channel.setMethodCallHandler { call, result in
  if call.method == "deviceName" { result(UIDevice.current.name) } else { result(FlutterMethodNotImplemented) }
}
```

> 能模组化就插件化：`flutter create --template=plugin ...`

---

## 11) 动画与手势 🎞️🖐️

- **隐式动画**：`AnimatedContainer/Opacity/Positioned`（无 Controller，简单稳）。  
- **显式动画**：`AnimationController` + `Tween` + `AnimatedBuilder`（完全可控）。  
- **物理交互**：`Draggable/LongPressDraggable/ReorderableListView`。  

```dart
class Spin extends StatefulWidget { const Spin({super.key, required this.child}); final Widget child; @override State<Spin> createState() => _SpinState(); }
class _SpinState extends State<Spin> with SingleTickerProviderStateMixin {
  late final controller = AnimationController(vsync: this, duration: const Duration(seconds: 2))..repeat();
  @override void dispose() { controller.dispose(); super.dispose(); }
  @override Widget build(BuildContext c) => RotationTransition(turns: controller, child: widget.child);
}
```

---

## 12) 国际化（l10n）🌍

```bash
flutter gen-l10n
```

`pubspec.yaml`：
```yaml
flutter:
  generate: true
  gen-l10n:
    arb-dir: lib/l10n
    output-dir: lib/gen
    untranslated-messages-file: l10n_untranslated.txt
```

`lib/l10n/app_en.arb` / `app_zh.arb`：
```json
{ "hello": "Hello", "@hello": { "description": "Greet" } }
```

使用：
```dart
import 'gen/app_localizations.dart';
Text(AppLocalizations.of(context)!.hello);
```

---

## 13) 性能工程（不丢帧指南）🏎️

- **帧预算**：60FPS ≈ 16.67ms；UI+GPU 同时达标。  
- **避免在 `build`/`didChangeDependencies` 做重 IO/计算**，放 `initState`/`FutureBuilder`/Isolate。  
- **列表**：`ListView.builder/Separated`；懒加载分页；图片用 `cached_network_image`。  
- **重绘隔离**：把复杂子树包 `RepaintBoundary`；Canvas 重绘用 `shouldRepaint` 精准触发。  
- **常量化**：能 `const` 就 `const`，减少 diff。  
- **Shader 抖动**：开启 Impeller（iOS 默认），Android 按需；或 SkSL 预热（profile 导出）。  
- **Isolate**：重计算丢到 Isolate，避免 UI 线程卡顿。

```dart
// 计算卸载：compute 会开短 Isolate
final sum = await compute<List<int>, int>((list) => list.fold(0, (a, b) => a + b), [1,2,3]);
```

DevTools：打开性能叠层（Performance Overlay）、帧时间线、内存快照，查 Jank 根因。

---

## 14) 可观测性与崩溃收集 🧭

- **日志&性能**：`flutter/devtools`、`logger`。  
- **崩溃**：Firebase Crashlytics 或 Sentry。

```yaml
dependencies:
  sentry_flutter: ^8.2.0
```

```dart
void main() async {
  await SentryFlutter.init((o) => o.dsn = 'https://<dsn>');
  runApp(const MyApp());
}
```

---

## 15) Flavors / CI / 上架 🚀

**Flavors**：
```
lib/
  main_dev.dart   // runApp(Env(dev))
  main_prod.dart
android/app/src/
  dev/AndroidManifest.xml
  prod/AndroidManifest.xml
ios/Runner/Configs/Dev.xcconfig / Prod.xcconfig
```

GitHub Actions（简化）：
```yaml
name: build
on: [push]
jobs:
  android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with: { flutter-version: 'stable' }
      - run: flutter pub get
      - run: flutter build appbundle --release --flavor prod -t lib/main_prod.dart
```

上架：
- Android：生成 keystore、`build.gradle` 配置签名、`flutter build appbundle` → Play Console。  
- iOS：设置签名 Team、Bundle ID、`flutter build ipa`（或 Xcode Archive）→ App Store Connect。  
- 灰度：商店内部测试/开放测试；**Remote Config + Feature Flags** 做运行时开关。

---

## 16) 图片/媒体/文件 📸

- **缓存**：`cached_network_image` + 占位/失败图。  
- **视频**：`video_player`（配合 `chewie` 控件）。  
- **文件**：`file_picker`、`path_provider`；大文件上传配分片与重试（Dio 拦截器）。

---

## 17) Web/桌面 适配 🌐🖥️

- **自适应**：`LayoutBuilder` / `MediaQuery`；响应式断点封装 `Responsive` 组件。  
- **鼠标/键盘**：`MouseRegion`、快捷键 `Shortcuts/Actions`。  
- **桌面**：窗口最小尺寸、拖拽、系统菜单用第三方包（如 `bitsdojo_window`）。

---

## 18) 反模式与纠偏 🧨

| 反模式 | 症状 | 纠偏 |
|---|---|---|
| 在 `build()` 里发请求 | 抖动 & 无限重建 | 放 `initState/FutureBuilder` |
| 大组件一坨 | 小改大抖 | 组件拆分 + `const` + `Selector`/`Consumer` |
| 滥用全局单例状态 | 依赖地狱 | Riverpod 拆 Provider，域内管理 |
| 图片原图直塞 | 内存爆 & 卡顿 | 限制分辨率 + 缓存 + 占位 |
| 长列表全量渲染 | 初屏卡 | `ListView.builder` + 分页 |
| 自己滚动数值动画 | 不稳 | 用 `Animated*` / `AnimationController` |
| UI 线程做密集计算 | 丢帧 | `compute/Isolate` |
| 直接靠“热更新”替代审查 | 上架风险 | 按平台规约，功能开关走 Remote Config |

---

## 19) 验收清单 ✅

- [ ] 路由：`go_router` 深链/守卫/嵌套可用。  
- [ ] 状态：Riverpod 归口管理，异步态 `AsyncValue` 兜底错误/加载。  
- [ ] 性能：首屏 < 1s（冷启动设备视情况），长列表不卡；Jank < 1%。  
- [ ] 可观测：崩溃/日志/性能指标上报，DevTools 日常使用。  
- [ ] 构建：flavors & CI 产物稳定；签名/上架流程顺畅。  
- [ ] 安全：敏感数据用 SecureStorage；权限文案完备。  
- [ ] i18n：ARB 生成 & 覆盖主要页面；暗黑/无障碍支持通过。

---

## 20) 练习 🏋️

1. 用 **go_router** 做三层嵌套路由（Tab → 子栈 → 详情），支持深链 `myapp://detail/42`。  
2. 用 **Riverpod** + `StateNotifier` 实现“分页列表 + 刷新/重试”，体验 `AsyncValue.guard`。  
3. 把一个 CPU 密集型任务（如图片滤镜）迁到 **Isolate**，对比帧时间分布。  
4. 接入 **Sentry**，打一条崩溃（try/catch）验证上报；再在 CI 里自动构建生产包。  
5. 用 **flutter_gen**（assets/fonts）、`ThemeData` 封装你的 **Design Tokens**，一键切主题。  

---

**小结**：Flutter 给了前端“一套代码，多端像素级一致”的超能力。把 **Widget 心智**学扎实，把 **Riverpod/路由/动画**当日常，把 **性能/工程化/可观测**做成习惯，你就能稳定产出“既顺滑又好看”的移动应用。🚀
