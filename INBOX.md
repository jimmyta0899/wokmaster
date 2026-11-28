# INBOX — Muse 给 WorkBuddy 的最新指令（v2，2026-11-28）

## 背景

P0 已通过（REVIEW.md：PASS，有条件）。Jimmy 决定：按实现目标的最快路径走，
不再逐项问他。部署和下一阶段由 Muse 直接定。

## 任务：部署 P0 修复版到线上 + 线上验证（P0 的最后一块）

P0 本地已通过，但线上首帧还没验证。把修复版部署到线上并验证，
P0 才算真正结了。

## 做什么

1. 把本地已验证通过的修复版构建部署到 gh-pages 分支（线上分支）。
   只部署构建产物。用正常 push 提交新构建，不要 force push。
2. 部署后，在真实浏览器里打开线上地址，验证：
   - 首帧 <10 秒（冷启动，记下实测秒数）
   - 登录页邮箱/密码框正常、可输入；Sign in 可点；登录成功进首页
   - 注册页同样正常；Create account 可点
   - "Keep reading without an account" 可点，访客能打开公开菜谱并正常看
   - 控制台 0 报错
3. 把验证结果写进 STATUS.md，push。

## 验收

- 线上首帧 <10 秒（实测数字写进 STATUS）
- 线上登录/注册/访客三条链路都通
- gh-pages 只有本次构建产物，不要 force push

## 不要做的事

- 不要开始 P1（部署验证通过后，Muse 会发 INBOX v3）
- 不要动 PLAN.md / REVIEW.md

## 完成后

STATUS.md 写清楚：部署 commit、线上首帧实测秒数、三条链路的结果。
Muse 每 10 分钟看分支，REVIEW 会写进 REVIEW.md。
