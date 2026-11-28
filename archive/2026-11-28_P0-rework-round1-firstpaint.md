# P0 返工第 1 轮记录 — 首帧性能 + 访客实测（2026-11-28）

## 任务来源

REVIEW.md 打回（返工 1/3）：P0 登录墙本体 3 项通过，**首帧 <10 秒未达标**
（冷启动 20–30 秒），Muse 要求先量分段耗时再优化，另需访客点进公开菜谱实测。
Jimmy 已退出日常管理，决定权在 Muse。

## 先量后改：冷启动分段实测

方法：本地起静态服务（`/wokmaster/` 子路径）+ headless Chrome CDP，
hook `window.__wmBootDone` 打点，读 `performance.getEntriesByType('resource')`。
脚本在 `web_preview/.mobai-scratch/measure.mjs`（本地工具，不进 loop 分支）。

结论：
- 本机 release 构建：首帧 `bootDone` 777ms，FCP 52ms（无外部请求时）。
- 资产体积：`main.dart.js` 4.1MB + `chromium/canvaskit.wasm` 5.2MB
  （dart2js + CanvasKit 渲染器）。
- 唯一外部运行时请求：`https://fonts.gstatic.com/s/roboto/v32/...woff2`。
  Flutter 引擎在字体清单里找不到 Roboto 时硬编码去 gstatic 拉回退字形，
  **不受 `canvasKitBaseUrl` 配置影响**（已实测：配 `fontFallbackBaseUrl` 无效）。
  gstatic 在中国大陆不可达 —— 这就是线上 20–30 秒（甚至 Browse 164 秒超时）的根因。
- 佐证：把 gstatic 请求拦掉（快速失败）时 bootDone 落在 4.2s；
  不拦时 bootDone 紧跟 gstatic 请求结束之后（冷 7.5s / 热 0.8s）。

## 修复（只改一处）

在 `pubspec.yaml` 的 `flutter.fonts` 里新增自托管 `Roboto` 字族：
- 资产：`assets/fonts/Roboto-Regular.ttf`（168KB，Flutter SDK 自带，Apache 2.0）
- 许可：`assets/fonts/Roboto-LICENSE.txt`

原理：引擎构建字体清单时发现 App 已声明 Roboto，就不再注册 gstatic 回退 URL，
改用本地资产。一次性去掉整个外部依赖。

验证：`flutter clean` 全量 release 重编（95.3s），确认：
- `FontManifest.json` 含 `"family":"Roboto"`
- 干净构建后 **gstatic 请求数 = 0**，本地 Roboto 请求 1 次（16ms）
- bootDone 777ms，FCP 52ms，无 console error / exception

## 访客实测（REVIEW 第 2 条）

真实浏览器（headless Chrome，未登录）打开公开分享落地
`#/recipe/ants-climbing-tree`（Supabase anon API 取到的真实 slug）：
- URL 停在菜谱页，**没有被拦回登录页**
- 页面标题变为菜谱标题，截图可见完整菜谱：标题、总时长/份量、标签、
  视频入口、过敏原提示、配料表（见 `.mobai-scratch/guest-recipe.png`）
- 顶部显示 "Shared link — sign in to cook along…" 只读提示条（预期）
- exceptions 0 / console errors 0

补充：从私有路由（`/profile`）被拦到登录页后，"Keep reading without an account"
入口可点（按钮有按压高亮），但因 pending 是私有页，点了会被再次拦回登录 ——
属 P0 记录里已说明的预期行为，非 bug。

## 排除的错误路径（留档避免重试）

- 在 `flutter_bootstrap.js` 配 `fontFallbackBaseUrl: "fontfallback/"`：无效，
  引擎仍请求 gstatic（该配置项未被 bootstrap 透传到引擎配置）。已回滚。
- 该配置项还试过直接改构建产物验证，同样不生效。

## 未处理 / 挂起

- 首帧数字：本机 777ms 远低于 10s，但线上 20–30s 需部署后在真机复测
  （本环境不部署、不上线）。根因已消除，属可解释的确定性修复。
- 少数中文字符显示豆腐块（如"蚂蚁"）：NotoSansSC 子集覆盖不全，既有问题，
  不在本轮范围，建议单列。
- 首帧加载耗时用节流模拟（200KB/s + 150ms RTT）时测量 harness 未能稳定
  捕获 bootDone（stale Chrome 目标干扰），故**不声称节流下的具体秒数**，
  以无节流干净构建的 777ms + gstatic 归零为准。
- R2：本环境 `R2_*` 凭证全空、boto3 未装，未上传（不伪造成功）。
- 真机验证：需部署后 Jimmy 在手机上复核（本轮不部署）。

## 产出文件

- 源码：`web_preview/pubspec.yaml`（加 Roboto 字族）、
  `web_preview/assets/fonts/Roboto-Regular.ttf`、`Roboto-LICENSE.txt`
- 干净 release 产物：`web_preview/build/web/`
- 验证截图/数据：`web_preview/.mobai-scratch/`（本地工具目录，不同步）

## 红线

未部署、未合并 gh-pages、未上线。等 Muse 复核 + Jimmy 点头上线。
