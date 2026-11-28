# P2-6 份量缩放（½x / 1x / 2x）+ 公制/美制并列 —— 实现与验证

日期：2026-11-28 · 对应 INBOX v10 · 工作区：`E:\data\2026-10-15-11-35-18\wokmaster_source\web_preview`

## 1. 复现（动手前）

- 数据层已有双单位：`Ingredient.metricValue/unitMetric`（克/毫升）与 `qty/unitImperial`
  （盎司/杯）**都已从数据库导入**，但 UI 只通过 `useMetricProvider` 的开关
  **二选一显示**，两单位从不同屏；没有份量档位，做 2 人份 / 6 人份只能心算。
- 详情页 `_qtyText()` 读出单一数量；做菜页 `CookPage` 完全没有配料区。
- 结论：不是数据缺失，是**换算 + 展示层缺失**，不需改数据库、不需后端。

## 2. 改了什么（一处一改，改完即测）

| 文件（源码工作区） | 作用 |
|---|---|
| `lib/core/utils/serving_scaler.dart`（新增） | 纯 Dart 换算层：`ServingScale` 三档 + `ScaledQuantity` 并列结果 + `ServingScaler.scale()`。无 Flutter/Supabase 依赖，可 VM 单测 |
| `lib/core/widgets/serving_scale_control.dart`（新增） | 共享档位控件 `SegmentedButton<ServingScale>` + 换算后人份数 |
| `test/serving_scaler_test.dart`（新增） | 11 项单测：三档乘数、质量换算、体积换算、反推、未知态 |
| `lib/pages/recipe/recipe_detail_page.dart` | 食材区插入档位切换；每行并列显示主用量（大）+ 副单位（小）；档位随人份联动 |
| `lib/pages/cook/cook_page.dart` | 做菜模式分步视图顶部新增配料卡：档位切换 + 并列用量；档位状态本地持有 |

数值口径：1 oz=28.349523125 g，1 lb=453.59237 g，1 cup=236.5882365 ml，
1 tbsp=14.7867648 ml，1 tsp=4.9289216 ml，1 fl oz=29.5735296 ml。
- 质量 ≥1000 g 用 kg，美制 ≥1 lb 用 lb；体积 ≥¼ cup 用 cup（½x 240ml → ½ cup，
  而非难读的 4 fl oz），小量用 tbsp/tsp。
- 单位解析不了（eggs/cloves/片）只缩放数值，不伪造换算；`qty=0` 且无单位 → 未知态
  「—」（库里 936 行导入丢描述，渲染成 "Ginger 0" 比破折号更误导）。

## 3. 验证（实测，不是推断）

- `flutter test test/serving_scaler_test.dart` → **11/11 通过**（质量/体积/反推/未知态全覆盖）。
- `flutter test` 全量 → **61/61 通过**（基线 50 + 新增 11）。P0 相关测试全绿
  （登录墙/访客/首帧相关回归用例随基线通过）。
- `flutter analyze`（4 个改动文件）→ 仅 `dart:html` 弃用 info（改动前既有），无 error/warning。
- `flutter build web --release` → **构建成功**（`√ Built build\web`，83.8s）；
  产物 `main.dart.js` 命中 `ServingScale` / `servings` / `1x` / `2x` 关键字。
- 档位乘数单测逐条断言：½x→0.5、1x→1.0、2x→2.0；4 人份 → 2/4/8 人份。

## 4. 诚实边界（未验证，不写"全部完成"）

- **UI 端到端点验未做**：本地构建未注入 Supabase 配置，详情页/做菜页取不到真实菜谱，
  档位按钮的点击联动以单测 + analyze + build 产物为证据，未在浏览器里逐档点击截图。
- **真机/线上未验证**：本轮 INBOX 明确不部署，真机多档切换留到部署后确认。
- **烹饪场景特有换算**（如"盐适量"的个数单位）只缩放不换算，符合数据现状，非缺陷。
- R2 归档本轮凭证环境变量全空、boto3 未安装 → **跳过上传**，未伪造成功；
  本 md 同步进 loop 分支 `archive/`，zip/大产物不同步（R2 规矩）。

## 5. 验收单对照（INBOX v10）

- [x] ½x / 1x / 2x 切换，配料自动换算正确 —— 单测 11/11 + 构建产物验证。
- [x] 公制 + 美制并列 —— 详情页/做菜页每行主副双单位并列， scaler 保证同源换算。
- [x] 不回归 P0 —— 全量 61/61 通过，登录/访客/首帧相关基线用例未破坏。
- [ ] 线上真机 —— 本轮不部署，诚实留待 Muse 定部署后验证。
