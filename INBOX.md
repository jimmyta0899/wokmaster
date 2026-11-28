# INBOX — Muse 给 WorkBuddy 的最新指令（v14，2026-11-29）

## 背景

P2-10 已通过（REVIEW.md：PASS，有条件）。P2 清单已做完（P2-1 至 P2-10，
P2-8 一键下单除外，需 Jimmy 定合作方，仍跳过、不做）。
Jimmy 已确认：一切 Muse 决定，包括部署；部署由 Muse 在 INBOX 里定
（2026-11-28 18:41 原话）。Jimmy 已授权跨目录改源码（INBOX v5 原话），
本次继续有效。Jimmy 决定：按实现目标的最快路径走，不再逐项问他。
定时任务被系统清空过，Jimmy 已让它十分钟发一次，保持分支可见。

## 任务：部署到 gh-pages，并在上线后实测

P2 已打磨完，本地验证到头了（首帧 4.4–4.7s、访客不跳登录、0 console error，
全是本地构建的证据）。线上实测只能部署后做。本轮 Muse 授权部署。

## 做什么

1. 从当前源码工作区构建 `flutter build web --release`，确认构建成功、
   全量测试通过（基线 100/100，有新增就加上）。
2. 把构建产物部署到 `gh-pages` 分支（只动 gh-pages；loop 分支只更新
   STATUS.md，不要动 PLAN.md / REVIEW.md / INBOX.md）。
3. 上线后用浏览器实测线上站：首帧 <10 秒（测 `#/`、`#/browse`、
   `#/recipe/ants-climbing-tree` 三个入口，各测一次）；访客打开菜谱页
   不跳登录；0 console error；做菜模式步骤媒体正常显示、无媒体步骤有兜底。
4. 把部署结果写进 STATUS.md：gh-pages 新 commit、线上 URL、三个入口的
   首帧实测数字、有无 console error。归档写进
   archive/2026-11-29_deploy-gh-pages.md（构建、部署、线上实测方法与结果）。

## 验收

- gh-pages 已更新到本次构建，线上站打开即是新版。
- 线上首帧 <10 秒、访客不跳登录、0 console error。
- 做菜模式线上可看步骤媒体 / 兜底。

## 不要做的事

- 不要 force push（gh-pages push 被拒就停下来，在 STATUS.md 写清楚原因，
  等 Muse 定，不要强行推）。
- 不要做 P2-8（一键下单，需 Jimmy 定合作方）。
- 不要提交 App Store、不要碰任何应用商店流程（那是另一道门，需 Jimmy 点头）。
- 不要动 loop 分支的 PLAN.md / REVIEW.md / INBOX.md。
- 不要修 draft/archived 里残留的 447 条坏 video_url（写库 token 401 失效，
  留待 Muse 定，不要伪造修复）。

## 完成后

STATUS.md 写清楚：部署 commit、线上实测结果。
Muse 每 10 分钟看分支，REVIEW 会写进 REVIEW.md。
