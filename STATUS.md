# STATUS — WorkBuddy 的交代

<!-- 用法：每次干完活（或返工后）更新这里，再 push。Muse 只看最新版本。 -->

## 当前状态

- 任务：v1 / P0 登录墙修复（已按 Jimmy 真机确认「出不来」启动并完成）
- 进度：P0 已完成，待 Muse 复核；P1 未开始
- 返工轮次：0 / 3

## 本次干了什么

定位并修复登录墙根因 `No MaterialLocalizations found`：shadcn_ui 0.61.2
的 `ShadApp.router` 底座是 WidgetsApp，Flutter 3.47 下不注入 Material 本地化，
登录页的 Material 原生 TextField 一 build 就抛断言，整个表单变灰色 ErrorWidget
（连锁导致按钮灰、注册/访客入口点了没反应）。

修复：`lib/app.dart` builder 里显式包一层 `Localizations`（Material + Widgets
delegate 兜底）；`_patchTheme` 钉死 FilledButton 前景/背景。两棵代码树
（web_preview / flutter-demo）同步，diff 一致。

按红线做了 `flutter clean` 全量 release 重编（避免增量编译产物不更新顽疾），
在真实浏览器上逐项验证 P0 登录链路。

## 产出文件

- 源码：`wokmaster_source/web_preview/lib/app.dart`、`wokmaster_source/flutter-demo/lib/app.dart`
- 干净 release 产物：`wokmaster_source/web_preview/build/web/`
- 详细记录：`archive/2026-11-28_P0-login-wall-fix.md`（含根因、修复、排除的错误路径、验证结论）

## 自评（对照 PLAN.md P0 验收单）

- [x] 登录页邮箱/密码框正常显示可输入；Sign in 砖红底白字可点；登录成功进首页
- [x] 注册页同样正常（邮箱/密码/邀请码三框 + Create account 可点）
- [x] "Keep reading without an account / 先不登录继续看" 正常渲染可点；
      公开分享落地路由免登录白名单在代码里已确认
- [ ] 首帧 <10 秒：**未达标**，release 冷启动约 20–30 秒，既有性能债，
      不在本次登录墙修复范围，建议单独立项

结论：P0 登录墙本体 3/3 通过，性能那条挂起。可进 P1。

## 卡住的地方

- 首帧性能（<10s）未做，需 Muse 决定是否单列任务。
- R2 归档：本环境 R2_* 凭证为空、boto3 未装，本次也无新归档包，未上传；
  凭证就绪后可补。不影响 P0 验收。

## 上线状态

**未部署、未上线。** gh-pages 主站未动，等 Jimmy 点头。
