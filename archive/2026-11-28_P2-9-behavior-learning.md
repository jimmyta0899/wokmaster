# P2-9 行为学习闭环（打卡/评分回流推荐权重与口味画像）—— 实现与验证

日期：2026-11-28 · 对应 INBOX v12 · 工作区：`E:\data\2026-10-15-11-35-18\wokmaster_source\web_preview`
决策依据：Jimmy 已授权一切由 Muse 决定，INBOX v12 明确「本轮不部署」→ 本轮未部署。
本轮开局拉取时 Muse 在轮询间隙推来 INBOX v12（P2-9）+ REVIEW P2-7 PASS；
本地原「无新指令」STATUS 提交经 `rebase --ff` 线性同步到新 tip（未 force push），
随后按 P2-9 实施。

## 1. 复现（动手前）

- 全库检索 `taste / profile / learn / signal / 画像 / 回流`：无行为学习实体、
  无口味画像、无推荐加权。`cookRecipe()` 只做 XP/积分结算；三维评分只写社区表；
  `TonightPage._buildCombos()` 按 `fetchTonight()` 的原始顺序（难度/创建时间）生成组合 ——
  做过了什么、喜欢什么，完全没有影响下一次推荐。
- 基线 `flutter test` **76/76 通过**（P2-7 结束状态）。结论：功能缺失，不是回归。
  最快路径：信号与画像存本机 SharedPreferences（游客也能看默认画像），
  不新增 Supabase 表；跨设备同步留待 Muse 另定（归档写明，不伪造上云）。

## 2. 改了什么（一处一改，改完即验）

| 文件（源码工作区） | 作用 |
|---|---|
| `lib/data/models/taste_profile.dart`（新增） | 纯 Dart：`CookSignal`（菜谱 id/slug/菜系/辣度/难度/耗时快照 + 评分 + 是否打卡 + 实测耗时 `actualMin`）、`TasteOverrides`（手动覆盖）、`TasteProfile`（画像，`affinityFor`/`effective*` 覆盖优先）；JSON 往返 |
| `lib/core/utils/taste_learning.dart`（新增） | `buildProfile(signals)`：评分 1–5 归一到 -1..1（Quick Log 无评分按 +0.5 弱正向），菜系取均值、辣度/难度/耗时按好感度加权平均，耗时优先用实测；`tasteScore()`：菜系强信号（±1.2）+ 辣度/难度/耗时距离惩罚；`rankRecipes()`：稳定排序（同分保持默认顺序），中性画像原样返回 |
| `lib/core/services/taste_profile_store.dart`（新增） | SharedPreferences 持久化（键 `wm.tasteprofile.v1`），信号 + 覆盖一起存取避免漂移，FIFO 上限 200 条，坏 JSON/存储不可用吞异常不崩主流程 |
| `lib/providers/taste_profile_provider.dart`（新增） | Riverpod StateNotifier：启动时 load，`record()` 沉淀信号，`setOverride()`/`setCuisineOverride()`/`clearOverrides()`/`clearAll()`；`recordCookSignal()` 助手吞异常保证打卡主流程不受影响；`tonightRankingProvider` 派生排序器 |
| `lib/pages/profile/taste_profile_page.dart`（新增） | 画像页：已学自 N 次做菜说明、菜系亲和度条 + 手动三档（−/·/+）、辣度/难度/耗时滑杆手动覆盖（可清空回到自动值）、恢复自动画像；颜色只取 Tokens，单一 CTA 符合 60-30-10 |
| `lib/pages/recipe/log_cook_sheet.dart`（改） | `showLogCookSheet` 增 `Recipe?` 参数；Quick Log 打卡成功后沉淀信号（无评分→弱正向） |
| `lib/pages/recipe/recipe_detail_page.dart`（改） | 「I made this」调用处传入 `recipe: r` 提供元数据快照 |
| `lib/pages/cook/cook_page.dart`（改） | 加 `_enteredAt` 计时；打卡成功后沉淀信号：Rate & Review 带综合分 + 实测耗时，Quick Log 无评分 |
| `lib/pages/tonight/tonight_page.dart`（改） | 结果列表按 `tasteProfileProvider` 画像加权排序（watch，画像异步加载完成后自动重排）后再生成组合，因此「每晚 3/12 个组合」顺序随画像变化 |
| `lib/core/router/app_router.dart`（改） | 新增 `/taste-profile` 路由 |
| `lib/pages/profile/profile_page.dart`（改） | 成长区新增「Taste profile / 口味画像」入口 |
| `test/taste_learning_test.dart`、`test/taste_profile_store_test.dart`（新增） | 15 项单测：中性/画像计算、正负菜系、Quick Log 弱正向、同菜系均值、实测耗时优先、手动覆盖优先、中性默认排序、高分升/低分沉、信号反转→顺序反转、同分稳定、不减候选、存储 round-trip/坏 JSON/清空 |

设计取舍：加权是「排序微调」不是过滤——所有菜仍在列表，只调整顺序，避免信息茧房；
菜系快照在打卡那一刻固化，菜谱后来改名/改辣度也不改写历史信号（同 P2-7/P2-4 快照策略）。

## 3. 验证（实测，不是推断）

- 分层单测：`flutter test test/taste_learning_test.dart test/taste_profile_store_test.dart` → **15/15 通过**。
- 全量：`flutter test` → **91/91 通过**（基线 76 + 新增 15），P0 基线用例全绿。
- `flutter analyze` 全量 → 10 issues 全为改动前既有（`dart:html` 弃用 info、
  `anonKey` 弃用 info、`porcelain_frame`/`user_profile_page` 的 unused-field warning），
  新增文件 0 error / 0 warning。
- `flutter build web --release` → **成功**（83.4s）；产物 `main.dart.js` 命中
  `wm.tasteprofile.v1`；SW 文件存在（P2-3 能力未破坏）。
- **浏览器实测（CDP + Chrome headless，release 产物）**：
  - 首帧 boot **4.4–4.6s** < 10s；访客路径 `/recipe/ants-climbing-tree` 未跳登录。
  - `/taste-profile` 新用户中性画像渲染截图：标题、What WokMaster learned、
    Fine-tune 三个滑杆、恢复按钮齐全；该页单独访问 **0 console error / 0 exception**。
  - 运行时种子验证：localStorage 写入 2 条信号（川菜 5 分 / 粤菜 1 分）后重载，
    截图确认画像页显示「Based on 2 cooks」、川菜亲和条满、粤菜亲和条浅、
    辣度/难度/耗时滑杆移到学到的档位并出现恢复图标 —— store→provider→UI 全链通过，0 报错。
  - 截图：`.mobai-scratch/p29-taste-profile.png`（中性）、`p29-taste-seeded.png`（有信号）、
    `p29-tonight.png`（Tonight 页）、结果 JSON `p29-*-result.json`（源码工作区，不同步进 loop）。
- 踩坑（已解决，非产品缺陷）：`document.body.innerText` 在 Flutter canvas 下为空，
  文案以截图为准；`/tonight` 在本地无 Supabase 构建下访问会触发
  `Supabase 未配置` console error（既有行为，与 P2-9 无关，画像页本身 0 报错）。

## 4. 诚实边界（未验证，不写「全部完成」）

- **打卡→信号沉淀的端到端点击未验证**：浏览器实测验证了种子信号经 store→provider→UI
  的读取链与画像页渲染；真实「Quick Log / Rate & Review 点击 → cookRecipe 成功 →
  record() 落库」链在本地无 Supabase（`cookRecipe` 抛 AUTH_REQUIRED）走不通，
  该部分证据为代码接线 + 15 项单测 + analyze + build 产物。
- **Tonight 顺序变化的页面级验证未做**：本地无 Supabase 菜谱数据，`fetchTonight` 返回空/报错，
  无法在浏览器里对比「画像前后组合顺序」；排序正确性由 `rankRecipes` 单测
  （高分川菜升/低分粤菜沉、信号反转→顺序反转）证明，页面接线由 analyze + build 证明。
- **跨设备同步未做**：信号存本机 SharedPreferences，游客可用；登录用户跨设备同步需
  Supabase 表，INBOX v12 允许最快路径，本轮未新增表，留待 Muse 定。
- **真机/线上未验证**：INBOX v12 明确本轮不部署，未部署、未合并 gh-pages、未上线。
- **R2**：`holia_import/upload_to_r2.py`（工作区根）存在，但 `R2_*` / `CF_R2_*` 环境变量
  全 EMPTY → **跳过上传，未伪造成功**；凭证未回显、未落盘。本归档 md 同步进 loop
  `archive/`；无 ZIP 产出，`manifest.md` 无需更新。
- **耗时实测口径**：`actualMin` 用「进入做菜模式到提交」的墙钟分钟数（含中途离开），
  是粗略值；快速打卡无实测耗时，回退用菜谱标称耗时。非精确厨房计时。

## 5. 验收单对照（INBOX v12）

- [x] 打卡/评分能沉淀为口味画像，画像可查看/可改 —— 模型 + store + provider + 画像页；
      Quick Log/Rate & Review 两处成功打卡接 `recordCookSignal()`；画像页菜系三档 + 三个滑杆可改。
- [x] Tonight 推荐顺序随画像变化（有数据时加权，无数据时默认） —— `rankRecipes` 稳定加权排序，
      中性画像原样返回；单测覆盖升/沉/反转/同分稳定/不减候选；Tonight 页接 watch 自动重排。
- [x] 不回归 P0 —— 全量 91/91 通过，首帧 4.4–4.6s < 10s，访客不跳登录，画像页 0 console error。
- [ ] 端到端点击「真实打卡→信号落库」与「画像前后 Tonight 顺序对比」 —— 本地无 Supabase，
      未走通（见 §4），证据为单测 + 接线 + build；诚实留白。
- [ ] 真机/线上 —— 本轮不部署，留待 Muse 定部署后验证。

文件 sha256（前 8 位）：taste_profile=`8bf25b94`，taste_learning=`d985557f`，
taste_profile_store=`73d5cfe7`，taste_profile_provider=`ba19e6dd`，taste_profile_page=`baff626b`。
