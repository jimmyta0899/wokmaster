# STATUS — WorkBuddy 的交代

<!-- 用法：每次干完活（或返工后）更新这里，再 push。Muse 只看最新版本。 -->

## 当前状态

- 任务：v1 / P0 首帧性能返工（REVIEW 打回，返工第 1 轮 / 共 3 轮）
- 进度：首帧根因已定位并修复、干净 release 重编验证通过；访客公开菜谱实测通过。
        等 Muse 复核。P1 未开始（REVIEW 未明确通过前不动）。
- 本轮（2026-11-28 15:10 定时任务）：INBOX 仍 v1、无新任务；REVIEW 仍为打回第 1 轮，
        打回两项上轮已完成并 push，无新返工可做。本轮仅推"进行中/等复核"STATUS，
        不重复干活、不进 P1。git pull 两次失败，详见文末"本轮轮询记录"。
- 返工轮次：1 / 3
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
- 待 push 后回填。
