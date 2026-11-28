# P2-1 灶具 / 锅具适配 —— 实现与验证记录（2026-11-28）

> 来源：INBOX v5（Muse，含 Jimmy 16:58 CST 原话"好，授权它跨目录改源码"）。
> 该授权仅放行"跨目录改源码"一项；部署 / gh-pages / 上线仍属红线，未动。
> 源码工作区：`E:\data\2026-10-15-11-35-18\wokmaster_source\web_preview`（非 git 仓库，无源码 commit hash）。

## 1. 先复现，再修

- 现状：`UserProfile` 模型已有 `stoveType` / `cookware` 字段并从 JSON 解析，
  但问卷只有 8 题、不采集设备；`_finish()` 里明确注释"炉灶/锅具不再收集"；
  全仓搜索 `stove_type|cookware` 只命中模型文件，无任何写入或消费方。
- 复现方式：先写 `test/device_heat_hint_test.dart`（8 条断言），首次运行**编译失败**
  （`Undefined name 'kStoveOptions' / 'DeviceHeatHint'`）→ 确认功能确实不存在，不是"以为没有"。
- 没有改菜谱原文：设备提示只在展示层追加，不改 `recipe_steps` 数据。

## 2. 改了哪些文件（源码工作区 web_preview）

新增：
- `lib/core/utils/device_heat_hint.dart` —— 纯 Dart 映射层：灶火 3 档 × 火候 3 档、
  锅具 5 类 → 中英提示；`kStoveOptions` / `kCookwareOptions` 固定存库值；
  游客（未选设备）返回 null，不改原文案。
- `lib/core/widgets/device_hint_card.dart` —— 详情页与 cook mode 共用的提示条。
- `test/device_heat_hint_test.dart` —— 8 条映射单测（含"换灶火/换锅具提示不同"）。
- `test/device_hint_card_test.dart` —— 2 条 widget 测试（提示条真实渲染、换设备文案变化）。

修改：
- `lib/pages/onboarding/onboarding_page.dart` —— 问卷 8 题 → **10 题**（Q9 灶火单选、
  Q10 锅具多选）；草稿缓存/恢复/清理新增 2 个 key；`_canContinue` 覆盖新题；
  落本机 `stove_type` / `cookware`；登录后 upsert `users.stove_type` / `users.cookware`
  与 `questionnaire` 内嵌字段（version 升 `v2.1-10q`）；提交后立刻写入 provider。
  兼容兜底：老库若无这两列，去掉设备字段重试主 upsert，不把问卷主流程卡死
  （设备偏好仍落本机，提示照常生效）。
- `lib/providers/recipe_providers.dart` —— 新增 `stoveTypeProvider` /
  `cookwareProvider` / `deviceHintProvider`（详情页与 cook mode 单一数据源）。
- `lib/main.dart` —— 冷启动从 SharedPreferences 恢复灶火/锅具，刷新页面提示不丢。
- `lib/pages/recipe/recipe_detail_page.dart` —— 步骤区顶部显示"已按设备调整"标签，
  每步原文之后追加设备提示条。
- `lib/pages/cook/cook_page.dart` —— 当前步骤的 tip / 常见错误之后追加同一提示条。
- `build/web/` —— 重新 release 构建产物（未部署、未上传）。

## 3. 怎么验证的 / 证据

### 3.1 单元测试（可重复）
`flutter test`：10/10 通过（8 映射 + 2 widget）。关键输出：
```
00:00 +3: 换灶火：同一道步骤的提示必须不同（实测证据）
00:00 +4: 换锅具：炒锅 vs 不粘锅的提示必须不同（实测证据）
00:00 +5: 电灶：提示包含预热时间（不计入菜谱时间）
00:00 +6: 小火炖煮 + 汤锅：命中低火与盖盖提示
00:01 +10: All tests passed!
```

### 3.2 火候提示变化实测（同一句 "Stir-fry over high heat"）
- 未选设备 → `null`（原文不动）。
- 燃气 → `Gas: turn to the highest flame; heat the pan until it just smokes.`
  / `燃气：开到最大火，锅烧到微微冒烟再下锅。`
- 电磁炉 → `Induction: set 1800-2000 W (level 8-9); check 10-15 s early...`
  / `电磁炉：功率 1800–2000W（8–9 档）；升温快，建议提前 10–15 秒检查，别照搬计时。`
- 电灶 → 含 `preheat 3-5 minutes`（预热不计入菜谱时间）。
- 小火 + 汤锅 → `电磁炉：小火档，功率 600–800W（2–3 档），可用定时。 汤锅：盖盖炖煮可缩短约三分之一时间。`
- 炒锅 vs 不粘锅 → `先热锅再下油，大火快翻更出锅气` vs `全程中小火，避免空烧和金属铲`。
- 详情页 / cook mode 共用 `DeviceHintCard`，widget 测试确认文案渲染且随设备变化。

### 3.3 静态检查与构建
- `flutter analyze`：10 issues，全部为改动前就存在的 deprecated/unused 提示，**无 error**。
- `flutter build web --release`：成功，编译 81.1s，`√ Built build\web`。
  （只有 wasm-dry-run 的 `dart:html` 提示，既有问题。）

### 3.4 真实浏览器（本地 release build，fresh profile）
- 首帧：FCP **3492 ms**（冷启动），二次 36 ms；`bootPresent=false`；
  `exceptions=0`、`consoleErrors=0`。< 10s，P0 首帧口径未回归。
- 问卷走查：`Step 1 of 10` 起，逐步点选到第 10 题，0 异常。
  - Q9 截图：`web_preview/.mobai-scratch/p2-q9-stove.png`
    （"My stove is..." + Gas flame / Electric coil / Induction）
  - Q10 截图：`web_preview/.mobai-scratch/p2-q10-final.png`
    （"I usually cook with..." + Wok / Skillet / Saucepan / Stockpot / Non-stick）
  - 原始记录：`p2-smoke-result.json`、`p2-onboard-walk.json`（同目录）。

## 4. 验收对照（INBOX v4/v5 P2-1）

- [x] 问卷能采集灶火/锅具并存入档案 —— Q9/Q10 真实渲染；写 SharedPreferences +
      登录态 upsert `users.stove_type` / `users.cookware`；冷启动恢复。
- [x] 步骤火候提示按设备变化，有实测证据 —— 8 条单测 + 2 条 widget 测试 +
      上面的真实提示文案；详情页与 cook mode 接同一 `deviceHintProvider`。
- [x] 不回归 P0（登录/访客/首帧）—— 未触碰 auth/router/guest 代码；
      本地 release 首帧 FCP 3492ms、0 异常；原 P0 四项仍由 Muse PASS 结论覆盖。
- [ ] 线上真机复核：**未做**（部署属红线，须 Jimmy 点头）。

## 5. 局限 / 诚实记录

- 本地构建未注入 Supabase `dart-define`，因此**没有**用线上真实菜谱数据端到端截图
  详情页 / cook mode 的设备提示；这两处的证据是 widget 测试 + 代码接线，不是线上实测。
  数据库端 `users.stove_type` / `cookware` 是否已有列也无法在本地验证：
  已加兼容兜底（无列时去掉设备字段重试，不阻塞问卷）。
- Q9/Q10 中文标注里仍有少量豆腐块（NotoSansSC 子集覆盖不全），是 P0 阶段已记录的
  既有字体问题；本次新增中文标注会命中同一问题，未改字体子集（需 DB 凭证 + fontTools）。
- R2 归档：脚本 `E:\data\2026-10-15-11-35-18\holia_import\upload_to_r2.py` 存在，
  凭证只从环境变量读取（不回显不落盘）；本环境 `R2_*` 全空且 boto3 未装
  （`ModuleNotFoundError: No module named 'boto3'`）→ **跳过，未上传**（不伪造成功）。
- ZIP：本轮无新 ZIP，`archive/manifest.md` 不变；本 md 按"对话 md 全量同步"进 loop 分支。
- 未部署、未合并 gh-pages、未上线、未改 PLAN/REVIEW、未 force push。
