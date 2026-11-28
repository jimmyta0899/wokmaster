# P2-4 菜谱集 / 文件夹分享实施与验证记录（2026-11-28，INBOX v8）

## 1. 原始现状（动手前复现）

在源码工作区 `E:\data\2026-10-15-11-35-18\wokmaster_source\web_preview` 复现确认：

- `ShareService` 只有三种单条 URL：`recipeUrl / eventUrl / communityPostUrl`
  （`share_service.dart:41-45`）；`ShareKind` 只有 recipe/cookLog/adjusted/event。
- 路由 `app_router.dart` 无 `/collection`；`isPublicLanding` 白名单只有
  `/recipe/ /event/ /community/post/`。
- 全库搜 collection / folder / 菜谱集 = 0 命中（唯一 folder 是图片上传目录）。
- 结论：与 INBOX v8 描述一致，分享基建完备但缺一层集合实体。

## 2. 方案取舍（为什么不直接写 Supabase 表）

数据层现状：repository 全部依赖 `SupabaseService.client`，而本地构建未注入
`SUPABASE_URL / SUPABASE_ANON_KEY`（`isConfigured == false`）。沿用 P2-3 已验证的模式：

- **本地可跑通层（本轮交付）**：纯 Dart 模型 + SharedPreferences 存储，
  单测 + web 构建 + 真实浏览器都能验证「建集合 → 加菜谱 → 分享链接打开看到内容」。
- **跨设备共享（本轮无法验证，写 DDL 供后续）**：真正给别人看需要后端表 + anon
  SELECT。本环境无 Supabase 凭证，不伪造"线上已验证"。DDL 见第 5 节。

分享链接是 hash 路由 `#/collection/<id>`，落点按 id 从本机读；同一浏览器内
打开链接即可看到集合内容。这也符合 REVIEW 对 P2-3 的诚实口径。

## 3. 改了什么（一次一处，改完即测）

### 改动 A：集合实体 + 本地存储（核心）
- 新增 `lib/data/models/recipe_collection.dart`：`RecipeCollection`（id / name /
  description / ownerId / createdAt / updatedAt / items）与 `CollectionItem`
  （slug / title / titleZh / imageUrl 的**快照**）。fromJson / toJson / copyWith，
  菜谱条目存快照而非 join，故无后端时链接照样能显示菜谱清单。
- 新增 `lib/core/services/collection_store.dart`：SharedPreferences CRUD
  （listAll / getById / create / rename / addRecipe / removeRecipe / delete），
  key = `wm.collections.v1`；重复 slug 不重复加；读写吞异常，不崩主流程。
- 新增 `test/collection_store_test.dart`（8 项），先跑通过再接下一处。

### 改动 B：分享接线
- 新增 `lib/core/services/share_urls.dart`：把纯 URL 拼接从 ShareService 抽出，
  不含 `dart:js_interop`，可在 VM 单测（原因见第 4 节）。
- `ShareService`：四种 URL 方法委托给 ShareUrls，公开 API 与行为不变；
  新增 `collectionUrl(id)` → `${appRoot}#/collection/$id`。
- `ShareSheet`：`ShareKind.collection` + `ShareTarget.collection(...)` 工厂 +
  `_headerFor` 分支（"Share this cookbook / 分享这个菜谱集"），复用七渠道面板。
- 新增 `test/collection_share_test.dart`（3 项）。

### 改动 C：集合页 + 路由 + 入口
- 新增 `lib/pages/collection/collections_page.dart`（`/collections`）：列表、
  新建、空态、进详情。
- 新增 `lib/pages/collection/collection_page.dart`（`/collection/:id`）：查看
  名称/描述/菜谱清单、点进菜谱、移除菜谱、编辑名称描述、分享、删除；
  链接不存在时给诚实提示，不伪造内容。
- 新增 `lib/pages/collection/add_to_collection_sheet.dart`：从菜谱详情页把
  当前菜存进已有/新建集合（支持粘贴整条链接或 slug）。
- `app_router.dart`：加两条路由；`/collection/` 进 `isPublicLanding` 白名单
  （分享落地游客可读，不被注册页挡门）。
- `profile_page.dart`：加「我的菜谱集」入口。
- `recipe_detail_page.dart`：分享按钮旁加「存进菜谱集」图标按钮。

## 4. 踩坑与修复（真实过程）

- **VM 单测编译失败**：`share_service.dart` 顶层 `import 'dart:js_interop'`，
  `JSObject / jsify` 只在 web 编译环境存在，VM `flutter test` import 它即报
  `'JSObject' isn't a type` / `The method 'jsify' isn't defined`，测试加载失败。
  **修复**：抽出纯 Dart 的 `share_urls.dart` 承载 URL 计算，ShareService 委托，
  测试只 import 纯 Dart 层。生产 web 行为不变（重新 build 验证通过）。
- **列表页 FutureBuilder 类型**：初写 `List<dynamic>` 与 `listAll()` 的
  `List<RecipeCollection>` 不匹配，改强类型并导入模型后消除。
- 未 force push、未改 PLAN/REVIEW、未部署。

## 5. Supabase `collections` 表 DDL（供跨设备分享，本轮未执行）

本环境无 Supabase 凭证，下面 DDL 只作记录，未在任何库执行，不声称线上可用：

```sql
create table if not exists public.collections (
  id uuid primary key default gen_random_uuid(),
  owner_id uuid not null references auth.users(id) on delete cascade,
  name text not null,
  description text not null default '',
  items jsonb not null default '[]'::jsonb,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
alter table public.collections enable row level security;
-- 分享落地：任何人可读集合；写操作仅 owner。
create policy "collections public read" on public.collections
  for select using (true);
create policy "collections owner write" on public.collections
  for all using (auth.uid() = owner_id) with check (auth.uid() = owner_id);
```

对应 Dart 层：`CollectionStore` 可加一个 Supabase 实现（按 id 读 / 按 owner 列
出），页面只依赖 store 接口，改动面小。本轮不接，避免无凭证的空实现。

## 6. 验证（改完本地跑通，未部署）

- `flutter test`：**35/35 通过**（既有 24 + 存储 8 + 分享 3），
  `All tests passed!`。
- `flutter analyze lib`：**10 issues，全为改动前既有**（dart:html deprecated、
  unused field、anonKey deprecated），**无 error、无新增**。
- `flutter build web --release`：**成功**（`√ Built build\web`，编译 86.9s）。
- 真实浏览器（headless Chrome + CDP，本地 release HTTP 服务，非部署）：见第 7 节。
- P0 回归：登录/访客/首帧代码本轮未触碰；首帧打点见第 7 节。

## 7. 浏览器端到端实测（CDP，本地 release）

脚本：`.mobai-scratch/p24-e2e.mjs`（未同步二进制进 loop 分支）。
截图：`p24-collection-detail.png`、`p24-collections-list.png`（同上）。
结果：见 STATUS.md 本轮记录（首帧 ms、路由落地、localStorage 原始值、异常数）。

## 8. 对照 INBOX v8 验收单

- [x] 能建集合：create CRUD + UI「New cookbook」入口，8 项单测覆盖。
- [x] 往里加菜谱：addRecipe（去重、快照）+ 详情页「存进菜谱集」+ 集合页「加菜谱」。
- [x] 分享集合链接并正常打开：`collectionUrl` + 分享面板 + `/collection/:id`
  白名单 + 浏览器 deep-link 实测（第 7 节）。
- [x] 不回归 P0（登录/访客/首帧）：auth/router 门逻辑未动，首帧打点见第 7 节。
- [ ] 跨设备 / 线上真机：**未验证**（本环境无 Supabase 凭证、不部署；DDL 见第 5 节）。

## 9. 诚实边界

- 集合数据目前存本机，跨设备分享需 Supabase（DDL 已备，未执行）。
- 同一浏览器内 deep-link 分享链接可看到内容；换设备打开同一 id 会显示
  "This cookbook link is not available on this device"，不伪造内容。
- Q9/Q10 之外的中文标注豆腐块为既有字体子集问题，非本轮范围。
