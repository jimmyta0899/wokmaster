# P0 登录墙修复记录 — 2026-11-28

## 任务来源
INBOX v1 / PLAN.md P0。Jimmy 真机确认「登录框出不来」= 真 bug，P0 启动。

## 根因（精确到异常）
`No MaterialLocalizations found`。

shadcn_ui 0.61.2 的 `ShadApp.router` 底座是 **WidgetsApp.router**（不是
MaterialApp.router），在 Flutter 3.47 下不注入 `MaterialLocalizations`。
登录页用的是 **Material 原生 TextField**，build 时需要该本地化，缺失直接抛
断言 → 整个表单子树被 Flutter 替换成灰色 ErrorWidget。

连锁现象全部由此解释：邮箱/密码框空白、Sign in 禁用、注册入口与访客入口
点了没反应（都在同一个崩溃子树里）。

## 修复（两处，两棵代码树同步）
文件：`lib/app.dart`（`web_preview/` 与 `flutter-demo/` 两棵树内容已 diff 一致）。

1. 在 `ShadApp.router` 的 `builder` 里，给路由 child 包一层显式 `Localizations`：
   ```dart
   child = Localizations(
     locale: const Locale('en', 'US'),
     delegates: const <LocalizationsDelegate<dynamic>>[
       DefaultMaterialLocalizations.delegate,
       DefaultWidgetsLocalizations.delegate,
     ],
     child: child,
   );
   ```
   独立于 shadcn 的 delegate 组装，兜底注入。

2. `_patchTheme` 里显式钉死 FilledButton 前景/背景（含禁用态）：
   `backgroundColor: Tokens.brand`（砖红）、`foregroundColor: Colors.white`、
   `disabledBackgroundColor: Tokens.paperSunken`、`disabledForegroundColor: Tokens.inkFaint`。
   解决 WidgetsApp 默认墨色文字压在砖红按钮上不可读的问题。

## 诊断过程中排除的错误路径（留档避免重试）
- 只在 `ShadApp.router` 传 `localizationsDelegates` / `supportedLocales`：不生效
  （shadcn 的 effectiveLocalizationsDelegates 组装顺序问题）。
- 用 shadcn 官方 `materialThemeBuilder` 入口：编译失败——它要的是 fork 包
  `material_ui` 的 ThemeData，和 `flutter/material.dart` 的不是同一个类型，
  无法直接传 `_patchTheme(ThemeData)`。已回滚。
- Flutter 增量编译有「源码改了但产物没变」顽疾，已按红线做 `flutter clean`
  全量 release 重编（85.5s）验证。

## 验证（真实浏览器截图，非静态模拟）
产物：`flutter clean` 后的干净 release 构建（`build/web`，main.dart.js 4.23MB），
本地以 `/wokmaster/` 子路径服务（http://127.0.0.1:8912/wokmaster/）。

- [x] 登录页邮箱/密码框正常显示、可输入；填表后 Sign in 变砖红底白字可点；
      用测试账号登录成功进首页
- [x] 注册页切换正常：邮箱/密码/邀请码三框 + Create account 按钮，可点
- [x] 分享拦截场景（访问 /profile）：顶部分享原因提示条 + 底部
      "Keep reading without an account / 先不登录，继续看" 均正常渲染、可点
      （/profile 本身是私人页，点后被再次拦回登录页属预期；
       公开分享落地 /recipe/ /event/ /community/post/ 的免登录白名单在代码里已确认）
- [ ] 首帧 <10 秒：未处理。release 版本地冷启动约 20–30 秒，属既有性能债，
      不在本次登录墙修复范围。

## 产出文件
- `wokmaster_source/web_preview/lib/app.dart`（已改）
- `wokmaster_source/flutter-demo/lib/app.dart`（已同步，与上面 diff 一致）
- 干净 release 产物：`wokmaster_source/web_preview/build/web/`

## 归档说明
- 本次无新 ZIP、无全量对话重新导出，故 `archive/manifest.md` 不动。
- R2 上传：本环境 `R2_*` 凭证环境变量为空、boto3 未安装，且归档脚本固定
  传 `outputs/wokmaster_archive` 那批（本次无新增），故未上传。凭证就绪后可补。

## 上线状态
**未部署、未上线。** 主站 gh-pages 未动，等 Jimmy 点头。
