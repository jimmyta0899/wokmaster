# STATUS — WorkBuddy 的交代

<!-- 用法：每次干完活（或返工后）更新这里，再 push。Muse 只看最新版本。 -->

## 当前状态

- 任务：v1 / P0 首帧性能返工 —— **Muse 已判 PASS（有条件），P0 四项验收全过**。
- 进度：首帧根因已消除、干净 release 重编验证通过；访客公开菜谱实测通过。
        P1 不自动开始，等 Jimmy 确认下一阶段再动；部署/真机复测须 Jimmy 点头。
- 本轮（2026-11-28 16:06 定时任务）：INBOX v1 未变，REVIEW 仍为 P0 PASS（有条件，21e24dc），
        远程 tip = 本地 HEAD（5399b0c）无新指令。无新进度，按手册只推一条"进行中/等确认"
        STATUS；不进 P1、不部署。git pull 挂起 5m44s 后终止，改用 ls-remote 核验远程状态。
        详见文末"本轮轮询记录（2026-11-28 16:06 定时任务）"。
- 返工轮次：1 / 3（P0 已通过，返工结束）
- 工作提交 hash：`9a92e28a0a97b3ad6b27a3a471ac066e7ad19e58`
  （`loop: P0 rework r1 - fix first-paint gstatic font hang, guest recipe verified`）

## 本次干了什么

REVIEW 打回的两条都做了：

**1) 首帧 <10s（先量后改）**
先分段实测，没猜：本地起服务 + headless Chrome CDP 打点，看到唯一外部运行时
请求是 `fonts.gstatic.com` 的 Roboto 回退字体。Flutter 引擎在字体清单找不到
Roboto 时硬编码去 gstatic 拉字，不受 `canvasKitBaseUrl` 影响（配
`fontFallbackBaseUrl` 实测无效，已回滚）。gstatic 在中国大陆不可达 → 线上
首帧卡到超时，这就是 20–30s 的根因。

只改一处：`pubspec.yaml` 新增自托管 `Roboto` 字族（Flutter SDK 自带字体，
Apache 2.0）。引擎发现 App 已声明 Roboto 就不再注册 gstatic URL。
`flutter clean` 全量 release 重编（95.3s）后验证：gstatic 请求 = 0，
本地 Roboto 加载 16ms，bootDone 777ms，FCP 52ms，无报错。

**2) 访客点进公开菜谱实测**
真实浏览器（未登录）打开 `#/recipe/ants-climbing-tree`（Supabase anon 取到的
真实 slug）：URL 停在菜谱页没被拦回登录，页面标题变为菜谱标题，截图可见完整
菜谱（标题/时长/份量/标签/视频/过敏原/配料），只读提示条正常，0 报错。
（从私有页 /profile 被拦后进 guest 入口会被再次拦回，属预期，非 bug。）

## 产出文件

- 源码：`wokmaster_source/web_preview/pubspec.yaml`（+ Roboto 字族）、
  `wokmaster_source/web_preview/assets/fonts/Roboto-Regular.ttf`、
  `Roboto-LICENSE.txt`
- 干净 release 产物：`wokmaster_source/web_preview/build/web/`
- 详细记录：`archive/2026-11-28_P0-rework-round1-firstpaint.md`
  （根因、分段数据、排除的错误路径、验证结论、截图位置）
- ZIP 清单：`archive/manifest.md`（本轮无新 ZIP）

## 自评（对照 PLAN.md P0 验收单 + REVIEW 打回项）

- [x] 登录页邮箱/密码框正常显示可输入；Sign in 可点；登录成功进首页（上轮通过，未回归）
- [x] 注册页三框 + Create account 可点（上轮通过，未回归）
- [x] "Keep reading without an account" 正常渲染可点（上轮通过，本轮复测仍可点）
- [x] 访客点进公开菜谱能正常看（本轮新增实测，真实浏览器截图）
- [x] 首帧 <10s 的根因已消除：本机干净构建 bootDone 777ms、gstatic 请求归零；
      线上 20–30s 需部署后真机复测（本环境不部署）
- [ ] 线上真机首帧 <10s：**未验证**（部署后 Jimmy 复核，本轮红线不部署）

## 卡住的地方 / 挂起

- 首帧线上数字：根因（gstatic 不可达）已确定性消除，但本机测不到线上网络，
  真实秒数待部署后复测。不写"已完成"。
- 节流模拟（200KB/s）下测量 harness 未稳定捕获 bootDone（stale Chrome 干扰），
  故不声称节流下的具体秒数。
- 少数中文字符豆腐块（NotoSansSC 子集覆盖不全）：既有问题，建议单列，不在本轮范围。
- R2 归档：本环境 `R2_*` 凭证全空、boto3 未装，**跳过，未上传**（不伪造成功）；
  凭证就绪后可补。
- 真机验证：需部署后 Jimmy 手机复核。

## 上线状态

**未部署、未上线。** gh-pages 主站未动，等 Muse 复核 + Jimmy 点头。

## 提交记录

- `9a92e28a0a97b3ad6b27a3a471ac066e7ad19e58` — P0 返工第 1 轮全部产出
  （STATUS + 归档记录 + zip 清单），已 push 到 origin/loop 并验证 tip 一致。
- `.workbuddy/` 为本地运行时目录，未纳入提交。

## 本轮轮询记录（2026-11-28 15:10 定时任务）

### 判断
- INBOX.md 版本 v1，与 STATUS 上次处理版本一致 → 无新任务。
- REVIEW.md 结论仍是"打回（返工第 1 轮 / 共 3 轮）"，打回两项（首帧首屏、访客实测）
  上轮已完成并 push（hash 9a92e28 / 2258826），本轮无新返工意见。
- REVIEW 未明确"通过"，按 INBOX v1 不进 P1。
- 按手册第 5 章，无新进度也推一条进行中 STATUS。

### 本地 git 状态（纯本地命令，未触发网络）
- HEAD = `2258826`（loop: backfill commit hash 9a92e28 into STATUS (P0 rework r1)）
- origin/loop 本地缓存 = `2258826`，与 HEAD 一致
- `git status --short` = `?? .workbuddy/`（仅本地运行时目录），工作区干净
- 当前分支 = loop

### 卡住的地方 / 报错原文（照抄，未改写）
`git pull` 连续两次失败，无法确认远程 loop 分支是否有 Muse 新推的 REVIEW/INBOX：

第 1 次（`git pull`，耗时约 3m56s）：
```
error: RPC failed; curl 56 schannel: server closed abruptly (missing close_notify)
error: 5113 bytes of body are still expected
fetch-pack: unexpected disconnect while reading sideband packet
fatal: early EOF
fatal: fetch-pack: invalid index-pack output
```

第 2 次（`git -c http.postBuffer=524288000 pull --no-rebase`，卡住 8m40s 无任何输出，
已手动终止）：
```
（无输出，进程挂起，达到"同一假设试两次没用就停下"阈值后终止）
```

### 试过什么
1. 普通 `git pull` —— RPC failed / early EOF，失败。
2. 加大 `http.postBuffer` 到 512MB + `--no-rebase` 再 pull —— 挂起无输出，终止。

### 需要什么
- 远程 loop 分支网络通道不稳定（HTTPS + token，schannel 报错）。本轮 STATUS 的 push
  若同样失败，将在下方记录原文。请 Jimmy/Muse 检查网络或远程可达性；本环境无法确认
  远程是否已有新的复核结论。
- 未部署、未合并 gh-pages、未上线，红线守住。

### 本轮 push 验证结果
- push 成功：`2258826..c488e0d  loop -> loop`
- `git log origin/loop -1` = `c488e0d loop: status - no new task, P0 rework r1 waiting Muse review; record pull network failure`
- HEAD = origin/loop = `c488e0d41280ed511c9e198be03c980392baff9a`（tip 一致）
- `git status` 干净：仅 `?? .workbuddy/`（本地运行时目录，手册允许的例外）
- 本轮提交 hash：`c488e0d41280ed511c9e198be03c980392baff9a`

## 本轮轮询记录（2026-11-28 15:26 定时任务）

### 判断
- INBOX.md 版本 v1，与 STATUS 上次处理版本一致 → 无新任务。
- REVIEW.md 结论仍是"打回（返工第 1 轮 / 共 3 轮）"，打回两项（首帧首屏、访客实测）
  上轮已完成并 push（hash 9a92e28），本轮无新返工意见、无新增打回。
- REVIEW 未明确"通过"，按 INBOX v1 不进 P1。
- 无新进度：按手册第 5 章，本轮只推一条"进行中/等复核"STATUS，不重复干活、不空提交。

### 本轮做了什么
- 无代码改动。源码工作区 `wokmaster_source/web_preview` 本轮未触碰（无新任务需动手）。
- R2 归档：本环境 `R2_ACCOUNT_ID / R2_ACCESS_KEY_ID / R2_SECRET_ACCESS_KEY /
  R2_BUCKET / R2_ENDPOINT` 全空，boto3 未装 → **跳过，未上传**（不伪造成功）。
  本轮无新产出 md/zip，archive/ 内容不变，manifest.md 不变。
- 未部署、未合并 gh-pages、未上线，红线守住。

### 卡住的地方 / 报错原文（照抄，未改写）
`git pull` 本轮再次两次失败，无法确认远程 loop 分支是否有 Muse 新推的复核结论：

第 1 次（`git pull`，耗时约 2m41s）：
```
error: RPC failed; curl 56 schannel: server closed abruptly (missing close_notify)
error: 12138 bytes of body are still expected
fetch-pack: unexpected disconnect while reading sideband packet
fatal: early EOF
fatal: fetch-pack: invalid index-pack output
```

第 2 次（`git pull`，挂起约 8m37s 无任何输出，按"同一假设试两次没用就停下"终止）：
```
（无输出，进程挂起，达到阈值后终止）
```

### 试过什么
1. 普通 `git pull` —— RPC failed / early EOF，失败。
2. 再试一次普通 `git pull` —— 挂起无输出，终止。

### 本地 git 状态（纯本地命令）
- HEAD = `2f6bd76`（loop: backfill push verification result and hash c488e0d into STATUS）
- origin/loop 本地缓存 = `2f6bd76`，与 HEAD 一致
- `git status --short` = `?? .workbuddy/`（仅本地运行时目录），工作区干净
- 当前分支 = loop

### 需要什么
- 远程 loop 分支 HTTPS 通道不稳定（schannel 报错，本轮连续两轮出现）。请 Muse/Jimmy
  检查远程可达性；本环境无法确认远程是否已有新的复核结论。
- 未部署、未合并 gh-pages、未上线，红线守住。

### 本轮 push 验证结果
- push 成功：`2f6bd76..16b846c  loop -> loop`
- `git log origin/loop -1` = `16b846c loop: status - no new task, P0 rework r1 awaiting Muse review; record 2nd consecutive pull network failure`
- HEAD = origin/loop = `16b846c71804f41cf94d639b369e0dad5a437219`（tip 一致）
- `git status` 干净：仅 `?? .workbuddy/`（本地运行时目录，手册允许的例外）
- 本轮工作提交 hash：`16b846c71804f41cf94d639b369e0dad5a437219`
- 注：本条验证结果回填后会产生一个 backfill 提交，tip 会前移一格；backfill 提交只是
  记录验证结果，无新增产出或任务进度。

## 本轮轮询记录（2026-11-28 15:45 定时任务）

### 判断
- INBOX.md 版本 v1，与 STATUS 上次处理版本一致 → 无新任务。
- REVIEW.md 结论仍是"打回（返工第 1 轮 / 共 3 轮）"，打回两项（首帧首屏、访客实测）
  上轮已完成并 push（hash 9a92e28），本轮无新返工意见。
- REVIEW 未明确"通过"，按 INBOX v1 不进 P1。
- 无新进度：按手册第 5 章，本轮只推一条"进行中/等复核"STATUS，不重复干活、不空提交。

### 远程状态（ls-remote 轻量核验成功）
- `git ls-remote origin refs/heads/loop` 返回 `b66f4d17e246a9b3c8c9f4cb16e9637025595926`
  = 本地 HEAD = origin/loop 缓存 → 远程 loop 分支无新提交，Muse 未推新的 REVIEW/INBOX。
- 虽然 `git pull` 仍失败（schannel 断连，详见下方），轻量 ref 查询证明远程内容与本地一致，
  本轮判断建立在最新远程状态上，不存在"看不到新指令"的盲区。

### 本轮做了什么
- 无代码改动。源码工作区 `wokmaster_source/web_preview` 本轮未触碰（无新任务需动手）。
- R2 归档：脚本位于 `E:\data\2026-10-15-11-35-18\holia_import\upload_to_r2.py`，凭证只从环境变量
  读取（不回显不落盘）。本环境 `R2_ACCOUNT_ID / R2_ACCESS_KEY_ID / R2_SECRET_ACCESS_KEY /
  R2_BUCKET` 全空，且 boto3 未安装 → **跳过，未上传**（不伪造成功）。本轮无新产出 md/zip，
  archive/ 内容与 manifest.md 不变。
- 未部署、未合并 gh-pages、未上线，红线守住。

### 卡住的地方 / 报错原文（照抄，未改写）
`git pull`（耗时约 2m47s）：
```
error: RPC failed; curl 56 schannel: server closed abruptly (missing close_notify)
error: 4082 bytes of body are still expected
fetch-pack: unexpected disconnect while reading sideband packet
fatal: early EOF
fatal: fetch-pack: invalid index-pack output
```
注：这是连续第 3 轮 pull 遇同样 schannel 错误；但 `git ls-remote`（4s）正常返回，说明远程可读，
只是 pull 走 fetch-pack 传包时断连。按"同一假设试两次没用就停下"，本轮 pull 只试一次，
改用 ls-remote 核验远程状态，不再重试 pull。

### 试过什么
1. `git pull` —— RPC failed / early EOF（schannel），失败（本轮 1 次）。
2. `git ls-remote origin refs/heads/loop` —— 成功，确认远程 tip 无变化。

### 需要什么
- 远程 loop 分支 HTTPS fetch-pack 通道不稳定（schannel 报错，连续第 3 轮）。push 通道此前正常。
  请 Muse/Jimmy 知悉：pull 失败不影响本轮判断（ls-remote 已核验），但若后续 Muse 推了新 REVIEW，
  WorkBuddy 可能因 pull 失败读不到正文；届时请 Jimmy 检查网络或远程可达性。
- 未部署、未合并 gh-pages、未上线，红线守住。

### 本轮 push 验证结果
- push 成功：`b66f4d1..febc54d  loop -> loop`
- `git log origin/loop -1` = `febc54d loop: status - no new task, P0 rework r1 awaiting Muse review; remote confirmed via ls-remote (pull schannel fails 3rd round)`
- HEAD = origin/loop = `febc54d4ca4009c5a79b17ee28e3ebe14c186339`（tip 一致）
- `git status` 干净：仅 `?? .workbuddy/`（本地运行时目录，手册允许的例外）
- 本轮提交 hash：`febc54d4ca4009c5a79b17ee28e3ebe14c186339`
- 注：本条验证结果回填后会产生一个 backfill 提交，tip 会前移一格；backfill 提交只是
  记录验证结果，无新增产出或任务进度。

## 本轮轮询记录（2026-11-28 15:51 定时任务）

### 判断（本轮有状态变化：P0 由打回转 PASS）
- 开局读到的 INBOX.md 版本 v1，与 STATUS 上次处理版本一致 → 无新任务版本。
- 开局 REVIEW.md 是"打回（返工第 1 轮）"，打回两项（首帧首屏、访客实测）已于 9a92e28
  完成并 push；本轮准备只推一条"等复核"STATUS。
- **推 STATUS 时被拒（non-fast-forward）→ 发现 Muse 期间推来新 REVIEW（21e24dc）：
  P0 PASS（有条件）。** 首帧根因消除、访客实测通过、登录/注册未回归，P0 四项全过。
- 新 REVIEW 明确：P1 不自动开始，等 Jimmy 确认下一阶段；部署/真机复测须 Jimmy 点头。
  故本轮不进 P1、不部署，只更新 STATUS 反映 PASS，等 Jimmy/Muse 下一步指令。

### 远程状态（pull 失败，ls-remote + 浅 fetch 取回新结论）
- 初次 `git ls-remote origin refs/heads/loop` = `89ce1f4...` = 本地 HEAD → 此时无新内容。
- 提交 STATUS 后 `git push` 被拒：`! [rejected] loop -> loop (fetch first)` /
  `Updates were rejected because the remote contains work that you do not have locally`。
- 复查 `git ls-remote` = `21e24dc1b10df4e0199191e0adf5cf0ff1ca4b3d`（远程已前移）。
- `git fetch --depth=3 origin loop` + `git rebase origin/loop` 成功取回并适用 Muse 新 REVIEW。
- 未 force push；本地未推的 STATUS 提交通过 rebase 叠到 Muse 提交之上，线性历史。

### 本轮做了什么
- 无代码改动。源码工作区 `wokmaster_source/web_preview` 本轮未触碰（P0 已通过，P1 待 Jimmy 点头）。
- 据 Muse 新 REVIEW 更新 STATUS：P0 PASS、P1 待确认、返工结束。
- R2 归档：脚本 `holia_import/upload_to_r2.py` 存在，但 `R2_ACCOUNT_ID /
  R2_ACCESS_KEY_ID / R2_SECRET_ACCESS_KEY / R2_BUCKET / R2_ENDPOINT` 全空，且 boto3
  未安装 → **跳过，未上传**（不伪造成功）。凭证就绪后补传。
- 无新对话 md / ZIP 产出，archive/ 与 manifest.md 内容不变（不空提交）。
- 未部署、未合并 gh-pages、未上线，红线守住。

### 卡住的地方 / 报错原文（照抄，未改写）
`git pull` 本轮只试一次，挂起约 4m46s 无任何输出，按"同一假设试两次没用就停下"及超时
阈值手动终止：
```
（无输出，进程挂起；stdout 仅到 "---PULL---" 标记，stderr 为空，达到阈值后终止）
```
注：连续第 4 轮 pull 异常（前三轮为 schannel RPC failed / early EOF，本轮为挂起无输出）。
`git ls-remote`（轻量 ref 查询）正常返回，远程可读；仅 pull 的 fetch-pack 传包通道不稳定。

### 试过什么
1. `git pull` —— 挂起约 4m46s 无输出，终止（本轮 1 次，不重试）。
2. `git ls-remote origin refs/heads/loop` —— 初次成功（tip 89ce1f4），push 被拒后复查
   成功（tip 21e24dc，发现新 REVIEW）。
3. `git fetch --depth=3 origin loop` + `git rebase origin/loop` —— 成功取回 Muse
   新 REVIEW 并把本地未推的 STATUS 提交线性叠上去。
4. 新 REVIEW 提供备用办法：pull 失败时可直接 raw.githubusercontent.com 读分支文件。

### 需要什么
- **Jimmy 确认是否进入 P1**（新 REVIEW：P1 不自动开始，等 Jimmy 确认下一阶段）。
- **部署 / 真机首帧复测须 Jimmy 点头**（P0 通过不等于可上线；线上 <10s 尚未真机复测）。
- 远程 loop 分支 HTTPS fetch-pack 通道不稳定（连续第 4 轮 pull 异常）。本轮靠浅 fetch
  成功取回新内容，未阻塞；后续若 fetch 也失败，按新 REVIEW 用 raw.githubusercontent.com
  读分支文件，不要干等。
- R2 凭证与 boto3 需 Jimmy 补齐，届时补传归档。

### 本轮 push 验证结果
- push 成功：`21e24dc..3833e4d  loop -> loop`
- `git log origin/loop -1` = `3833e4d loop: status - P0 PASS per Muse r1 review (21e24dc); P1 awaits Jimmy, no deploy; log pull hang + fetch/rebase recovery`
- HEAD = origin/loop = `3833e4dad0c234d715e45bda10efb7cc18cb2a87`（tip 一致）
- `git status` 干净：仅 `?? .workbuddy/`（本地运行时目录，手册允许的例外）
- 本轮工作提交 hash：`3833e4dad0c234d715e45bda10efb7cc18cb2a87`
- 注：本条验证结果回填后会产生一个 backfill 提交，tip 会前移一格；backfill 提交只是
  记录验证结果，无新增产出或任务进度。

## 本轮轮询记录（2026-11-28 16:06 定时任务）

### 判断
- INBOX.md 版本 v1，与 STATUS 上次处理版本一致 → 无新任务。
- REVIEW.md 结论仍为 P0 PASS（有条件，commit 21e24dc）：P0 四项验收全过。
  明确写着"P1 不自动开始，等 Jimmy 确认下一阶段再动""部署/真机复测须 Jimmy 点头"。
  → 本轮不进 P1、不部署。
- 无新进度：按手册第 5 章，本轮只推一条"进行中/等确认"STATUS，不重复干活、不空提交。
- 权限模式自查：本轮所有 git 命令均直接执行，未出现等待确认的弹窗/挂起，判断为 auto 模式正常。

### 远程状态（ls-remote 轻量核验成功）
- `git ls-remote origin refs/heads/loop` =
  `5399b0c2968d115e5f11d34733b07c8c7b319481` = 本地 HEAD = origin/loop 缓存
  → 远程 loop 分支无新提交，Muse 未推新的 REVIEW/INBOX。
- 本轮判断建立在最新远程状态上，不存在"看不到新指令"的盲区。

### 本轮做了什么
- 无代码改动。源码工作区 `wokmaster_source/web_preview` 存在但未触碰
  （P0 已通过，P1 待 Jimmy 点头，无新任务需动手）。
- R2 归档：脚本 `E:\data\2026-10-15-11-35-18\holia_import\upload_to_r2.py` 存在，
  凭证只从环境变量读取（不回显不落盘）。本环境 `R2_ACCOUNT_ID / R2_ACCESS_KEY_ID /
  R2_SECRET_ACCESS_KEY / R2_BUCKET / R2_ENDPOINT` 全空，且 boto3 未安装
  （`ModuleNotFoundError: No module named 'boto3'`）→ **跳过，未上传**（不伪造成功）。
- 本轮无新对话 md / ZIP 产出：`archive/` 现有两份 md 与 `manifest.md` 内容不变，不做空同步。
- 未部署、未合并 gh-pages、未上线，红线守住。

### 卡住的地方 / 报错原文（照抄，未改写）
`git pull` 本轮挂起约 5m44s、无任何输出，按手册"同一假设试两次没用就停下"及超时阈值
终止，未重试：
```
（无输出，进程挂起；stdout / stderr 均为空，达到阈值后终止）
```
注：连续第 5 轮 pull 异常（前四轮为 schannel RPC failed / early EOF 或挂起）。
`git ls-remote` 正常返回，远程可读；仅 pull 的 fetch-pack 传包通道不稳定。
REVIEW 已授权 pull 失败时改用轻量核验，本轮照此执行。

### 试过什么
1. `git pull` —— 挂起约 5m44s 无输出，终止（本轮 1 次，不重试）。
2. `git ls-remote origin refs/heads/loop` —— 成功，确认远程 tip 无变化。

### 需要什么
- **Jimmy 确认是否进入 P1**（REVIEW 条件：P1 不自动开始）。
- **部署 / 真机首帧复测须 Jimmy 点头**（P0 通过不等于可上线，线上 <10s 未真机复测）。
- R2 凭证与 boto3 需 Jimmy 补齐，届时补传归档。
- 远程 loop 分支 HTTPS fetch-pack 通道不稳定（连续第 5 轮 pull 异常），push 通道此前正常。

### 本轮 push 验证结果
- 待 push 后回填。


