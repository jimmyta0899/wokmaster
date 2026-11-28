# P2-7 周计划 + 按过道排序的购物清单 —— 实现与验证

日期：2026-11-28 · 对应 INBOX v11 · 工作区：`E:\data\2026-10-15-11-35-18\wokmaster_source\web_preview`
决策依据：Jimmy 已授权一切由 Muse 决定，INBOX v11 明确「本轮不部署」→ 本轮未部署。

## 1. 复现（动手前）

- 全库检索 `meal_plan / weekly / week_plan / aisle / storeSection`：无周计划实体、
  无过道排序代码、无 storeSection→过道映射；`CartProvider` 只有同名合并计数，
  无份量换算文本，购物清单页只显示 `name × qty` 与来源菜谱。
- 基线 `flutter test` **61/61 通过**（P2-6 结束状态）。结论：功能缺失，
  不是回归；不需要新数据库表（最快路径用 SharedPreferences，游客也能排计划）。

## 2. 改了什么（一处一改，改完即验）

| 文件（源码工作区） | 作用 |
|---|---|
| `lib/data/models/weekly_plan.dart`（新增） | 纯 Dart 模型：`PlannedIngredient`（加计划时快照配料）、`PlannedMeal`（day 0=Mon…6=Sun，自带 P2-6 `ServingScale`）、`WeeklyPlan`（add/remove/setScale/clear + JSON 往返）。无 Flutter/Supabase 依赖 |
| `lib/core/utils/aisle_order.dart`（新增） | 11 个过道分组（蔬果→肉→水产→乳蛋→烘焙→冷冻→干货→亚洲/国际→调味→日用→其他）；`resolveAisle()` 按 storeSection → category → item 三级兜底、完整词优先于切词；`AisleShoppingList.build()` 同名小写归一去重、按每道菜自己的档位换算后按单位族（g/ml/个数）累加、来源去重，按 `aisle.order` 排序；未知进 Other，不猜具体货架 |
| `lib/core/services/weekly_plan_store.dart`（新增） | SharedPreferences 持久化（键 `weeklyplan.v1`），`weekStartOf()` 按周一计算，跨周空计划自动重建，坏 JSON 回退空计划 |
| `lib/providers/weekly_plan_provider.dart`（新增） | Riverpod AsyncNotifier：load/add/remove/setScale/clear/mergeIntoCart，一键把整周换算后的配料合并进现有购物车 |
| `lib/pages/plan/weekly_plan_page.dart`（新增） | 7 天卡片（每天加菜/删菜/档位切换）、菜谱搜索选择器（选中后拉详情做配料快照）、按过道分组的购物清单预览、「生成购物清单」主 CTA（强调色仅此一处，符合 60-30-10） |
| `lib/providers/cart_provider.dart`（改） | `CartEntry` 增加 `amountText`（向后兼容 JSON 默认空）；新增 `mergeShoppingLines()`，复用现有去重语义，把 P2-7 清单行并入购物车 |
| `lib/pages/shopping/shopping_list_page.dart`（改） | 有 `amountText` 时显示换算后用量文本，保留原 `× qty` 与来源 |
| `lib/core/router/app_router.dart`（改） | 新增 `/plan` 路由 |
| `lib/pages/home/home_page.dart`（改） | 首页 AppBar 增加周计划入口（不动 6 项底部 dock） |
| `test/weekly_plan_test.dart`、`test/aisle_order_test.dart`、`test/weekly_plan_store_test.dart`（新增） | 15 项单测：7 天排菜/增删/换档、季节与 JSON 往返、去重、过道顺序、档位换算联动、未知项兜底、跨周重建 |

设计取舍：档位换算复用 P2-6 同一个 `ServingScaler`，不在清单里另写一套换算；
配料在加进计划时快照，菜谱后续改动不会悄悄改变已排好的一周（同 P2-4 集合快照策略）。

## 3. 验证（实测，不是推断）

- `flutter test test/weekly_plan_test.dart test/aisle_order_test.dart` → **12/12 通过**。
- `flutter test` 全量 → **76/76 通过**（基线 61 + 新增 15），P0 相关基线用例全绿。
- `flutter analyze` 全量 → 仅既有 `dart:html` 弃用 info（10 项，改动前存在），无新增 error/warning。
- `flutter build web --release` → **成功**；产物命中 `weeklyplan.v1` / `Generate shopping list` /
  `Asian & International` 标志；`flutter_service_worker.js` 存在（P2-3 能力未被破坏）。
- **浏览器实测（CDP + Chrome headless，release 产物）**：
  - 首帧 boot 完成 **4.0–4.4s**，< 10s；无 console error、无 uncaught exception。
  - 访客路径 `/recipe/ants-climbing-tree` 未被重定向到登录页（`redirectedToLogin: false`）。
  - `/plan` 实际渲染截图确认：标题「Weekly plan / 每周计划」、Mon–Sun 七张卡片、
    每卡「Add meal / 加菜」、底部「Shopping list preview (by aisle) / 购物清单预览（按过道）」
    与「Generate shopping list / 生成购物清单」CTA；空计划时 CTA 禁用并有「Add a meal to see the list」提示。
  - 截图：`.mobai-scratch/p27-plan.png`、`.mobai-scratch/p27-plan-bottom.png`（源码工作区，不同步进 loop）。
- 冒烟脚本踩坑与修正（已解决，非产品缺陷）：Flutter web 用 path URL 策略，`#/plan` 不会命中路由；
  首次以 `/wokmaster/` 子路径托管，GoRouter 看到 `/wokmaster/plan` 报 Page Not Found；
  改为根路径托管 + `/plan` 后正常。Flutter canvas 渲染使 `document.body.innerText` 为空，
  文案校验以截图为准，不把 DOM 文本为空误判成页面空白。

## 4. 诚实边界（未验证，不写「全部完成」）

- **加菜交互未逐点击验证**：浏览器实测验证了页面的排菜网格、过道预览区与 CTA 渲染；
  本地构建无 Supabase 配置，菜谱搜索结果为空，因此「搜索→选菜→配料快照→生成清单」的
  完整端到端点击链未走通，该部分证据为 15 项单测 + analyze + build 产物。
- **跨设备同步未做**：本周计划存本机 SharedPreferences，游客可用；登录用户跨设备同步需
  Supabase `meal_plan` 表，INBOX v11 允许最快路径，本轮未新增表，留待 Muse 决定。
- **超市过道映射为启发式**：数据里没有真实 storeSection 枚举样本，按关键词三级兜底，
  映射不到进 Other；真实货架顺序需接入门店数据后校准，不伪造精确货架号。
- **真机/线上未验证**：INBOX v11 明确本轮不部署，未部署、未合并 gh-pages、未上线。
- **R2**：`holia_import/upload_to_r2.py` 存在，但 `R2_ACCOUNT_ID / R2_ACCESS_KEY_ID /
  R2_SECRET_ACCESS_KEY / CF_ACCOUNT_ID / CF_R2_*` 环境变量全为 EMPTY → **跳过上传，
  未伪造成功**；凭证未回显、未落盘。本归档 md 同步进 loop `archive/`；无 ZIP 产出，
  `manifest.md` 无需更新。

## 5. 验收单对照（INBOX v11）

- [x] 一周 7 天能排菜谱，购物清单合并去重正确，按过道顺序排列 —— 单测覆盖
      同项合并/来源去重/11 分区顺序，页面截图确认 7 天网格与过道预览。
- [x] 清单用量与份量缩放一致 —— 每道菜用自身 `ServingScale` 走 P2-6 同一个
      `ServingScaler`，单测断言 ½x/1x/2x 累加进清单；页面显示换算后用量文本。
- [x] 不回归 P0 —— 全量 76/76 通过，首帧 4.0–4.4s，访客路径不跳登录，无 console error。
- [ ] 端到端点击「搜索→选菜→生成清单」全链 —— 本地无 Supabase 菜谱数据，未验证（见 §4）。
- [ ] 真机/线上 —— 本轮不部署，诚实留待 Muse 定部署后验证。

文件 sha256（前 8 位）：weekly_plan=`9a6ebc43`，aisle_order=`6b3c3cf0`，
weekly_plan_store=`568d54e5`，weekly_plan_provider=`f4460f0c`，weekly_plan_page=`6531908f`。
