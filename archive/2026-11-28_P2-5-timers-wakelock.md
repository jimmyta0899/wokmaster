# P2-5 做菜模式屏幕常亮 + 多个命名计时器（2026-11-28，INBOX v9）

## 1. 原始现状（动手前复现）

在源码工作区 `E:\data\2026-10-15-11-35-18\wokmaster_source\web_preview` 复现确认：

- `lib/pages/cook/cook_page.dart` 只有**一个**每步单计时器（`_armTimer` / `_timer` /
  `_remaining` / `_deadline`，墙钟计时），无屏幕常亮、无「带名字的多个计时器」。
- `pubspec.yaml` 无 wakelock 依赖；全库 grep `wakelock / wakeLock / 常亮` = 0 命中。
- grep `namedtimer / named timer / 多个计时` = 0 命中。
- 与 INBOX v9 描述一致：做菜时屏幕会黑、多个步骤计时只能靠脑子记。
- 基线：`flutter test` 全量 **35/35 通过**（改动前），Flutter 3.47.7 / Dart 3.13.5。
- 授权：Jimmy 已授权跨目录改源码、一切以 Muse 决定为准；本轮为实施级新任务，
  非 REVIEW 打回（返工轮次仍 1/3，不消耗）。

## 2. 方案取舍

沿用项目已验证的「纯 Dart 逻辑 / JS 交互分离」模式（见 P2-4 的 share_urls）：

- **纯 Dart 层（可 VM 单测）**：`core/utils/wake_lock.dart`（WakeLockPlatform 抽象 +
  WakeLockController 状态机）、`core/utils/named_timer.dart`（NamedTimer 模型）。
  不 import dart:js_interop，单测直接跑。
- **Web 平台层**：`core/services/wake_lock_web.dart` 用 `dart:js_interop` 调
  `navigator.wakeLock.request('screen')` / `.release()`，浏览器不支持 / 被拒 / 抛错
  一律静默降级，由 UI 诚实提示，绝不假装成功。
- **屏幕常亮生命周期**：进 CookPage.initState 即 acquire；dispose 时 release 恢复
  息屏；页面进入后台浏览器会自动释放，监听 `visibilitychange` 回前台重新 acquire。
- **不引入 wakelock_plus 原生依赖**：本项目是 Flutter Web 单端，直接用 Web
  Screen Wake Lock API 最轻，避免为 web 引入插件兼容层。

## 3. 改了什么（一次一处，改完即测）

### 改动 A：屏幕常亮状态机 + 单测
- 新增 `core/utils/wake_lock.dart`：`WakeLockPlatform`（supported/request/release）、
  `NoopWakeLockPlatform`、`WakeLockController`（acquire/release，异常吞掉、幂等释放）。
- 新增 `test/wake_lock_test.dart`（6 项）：不支持/成功/拒绝/抛错/幂等释放/未 acquire
  就 release。先跑通过（6/6）再接下一处。

### 改动 B：命名计时器模型 + 单测
- 新增 `core/utils/named_timer.dart`：`NamedTimer`（id/name/totalSeconds，
  墙钟 deadline，tick/pause/resume/reset/cancel/format），到时返回 true 一次
  （timeUp 状态位避免每秒重复提醒）。
- 新增 `test/named_timer_test.dart`（9 项）：到时提醒一次、后台节流按墙钟扣减、
  暂停冻结、继续从冻结剩余重算、重置、取消、多计时器独立到时、mm:ss 格式。
- 中途修一处**测试断言错误**（非代码缺陷）：resume 用例原先只 tick 到旧 deadline，
  改为验证「新 deadline 才到时」，更符合墙钟重算语义。

### 改动 C：Web 平台层 + CookPage UI
- 新增 `core/services/wake_lock_web.dart`：`BrowserWakeLockPlatform`，extension type
  封装 navigator.wakeLock，request/release 全 try/catch 降级。
- 修改 `cook_page.dart`：
  - initState 创建 controller 并 acquire、监听 visibilitychange 回前台重获、
    启动 1s 命名计时器 tick；dispose 取消 tick/监听并 release。
  - 统一提醒入口 `_announceTimer(name, kind)`：Web SpeechSynthesis 语音 + SnackBar
    双通道，命名计时器报到时名字、步骤计时器报换步提示（原 `_announceTimeUp` 复用）。
  - 步骤视图新增「Timers」入口卡，打开 `_NamedTimersSheet`：屏幕常亮开关 +
    名称/分钟添加表单 + 计时器列表（暂停/继续/重置/删除、到时绿标）。
  - 面板是独立 route，自带 1s repaint timer 刷新读数（父级 setState 不重建 bottom sheet）。

### analyze 修正
- `html.document.hidden` 是 `bool?`，`!x` 编译报错 → 改 `== false`。修正后
  `flutter analyze lib` **无 error**（10 条均为改动前既有 deprecated/unused）。

## 4. 验证结果（改完本地跑通，未部署）

- `flutter test` 全量：**50/50 通过**（基线 35 + 新增 15）。
- `flutter analyze lib`：无 error；既有 10 条 info/warning 未新增。
- `flutter build web --release`：成功（`√ Built build\web`，编译 88.9s）。
- build 产物含 P2-5 代码标志：`wakeLock` / `Kitchen timers` / `Screen stays on` /
  `keep screen on`（grep main.dart.js 命中）。
- 真实 Chrome headless（本地 release，CDP 打点）：
  - **首帧 bootDone 4834ms < 10s**，consoleErrors=[]、exceptions=[] → P0 首帧未回归。
  - 访客：`#/recipe/ants-climbing-tree` 落到 `/onboarding`（本地模式无登录墙），
    `redirectedToLogin=false` → 未被登录墙拦截。
  - **屏幕常亮实测通过**：页面加载前注入 wakeLock spy，导航到 `/cook/ants-climbing-tree`，
    记录到 `navigator.wakeLock.request('screen')` 调用 1 次（`WAKELOCK_REQUESTED_ON_COOK_ENTRY`，
    证据 `.mobai-scratch/p25-wakelock-result.json`）。主动探针证明 spy 有效、原生 API
    可用（inOp=true、reqType=function、isSecureContext=true）。
  - 排查备忘：GoRouter 路由是 `/cook/:slug`（**无** `/wokmaster` 前缀），测试 URL
    必须用 `http://127.0.0.1:<port>/cook/<slug>`；带 `/wokmaster/` 前缀会
    `GoException: no routes for location`。

## 5. 诚实边界（未验证，不伪造）

- **命名计时器 UI 端到端（设/提醒的真实页面操作）未验证**：本地 build 未注入
  `SUPABASE_URL / SUPABASE_ANON_KEY`，`recipeRepositoryProvider` 创建即抛
  「Supabase 未配置」，CookPage 步骤模式（`_buildStepMode`）不渲染，计时器入口
  无从点起。曾尝试注入离线缓存 + 假 dart-define 重建，前者因 provider 在 catch
  之前抛错无效，后者会触发登录门需额外 mock session——按「卡住即停」不再烧积分。
  证据为 9 项命名计时器单测（逻辑闭环）+ analyze/build（UI 语法与接线）+ build 产物
  文案命中。**不写「UI 端到端已完成」**。
- 屏幕常亮的**真机持续亮屏**未验证：本轮验证的是 API 被正确调用；真机物理亮屏
  需部署后在手机上确认。
- 语音提醒在 headless 环境无音频输出，验证的是调用路径（SpeechSynthesis 存在于
  Chrome），实际发声需真机。
- 少数中文标注豆腐块（NotoSansSC 子集覆盖不全）：P0 阶段既有，未改字体子集。
- R2 归档：`E:\data\2026-10-15-11-35-18\holia_import\upload_to_r2.py` 存在，凭证只从
  环境变量读（不回显不落盘）。本环境 5 项 `R2_*` 全空、boto3 未装 → **跳过，未上传**
  （不伪造成功）。
- ZIP：本轮无新 ZIP 产出，`archive/manifest.md` 不变。
- 未部署、未动 gh-pages（INBOX v9 明确本轮不部署）。

## 6. 产出文件

- 源码：`lib/core/utils/wake_lock.dart`、`lib/core/utils/named_timer.dart`、
  `lib/core/services/wake_lock_web.dart`、`lib/pages/cook/cook_page.dart`（改动）。
- 单测：`test/wake_lock_test.dart`、`test/named_timer_test.dart`。
- 浏览器证据：`.mobai-scratch/p25-wakelock-result.json`、`p25-smoke-result.json`、
  `p25-cook-page.png`、`p25-timers-step.png`（未同步二进制进 loop 分支）。
- 归档对话：本文件。
