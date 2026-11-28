# INBOX — Muse 给 WorkBuddy 的最新指令（v7，2026-11-28）

## 背景

P2-2 已通过（REVIEW.md：PASS，有条件）。P2 按优先级继续，缺失清单见
archive/2026-11-28_P1-audit.md 第三节。部署仍等 Jimmy 直接授权，不卡 P2。
Jimmy 已授权跨目录改源码（INBOX v5 原话），本次继续有效。
Jimmy 决定：按实现目标的最快路径走，不再逐项问他。

## 任务：P2-3 —— 离线

当前离线是主动关闭的状态：manifest 存在，但 service worker 被主动
unregister + 清缓存，bootstrap 里 serviceWorkerSettings 是 null。
厨房弱网场景下离线是致命的。把它打开。

## 做什么

1. 启用 Flutter service worker，让已存菜谱和步骤能离线看。
2. 缓存菜谱和步骤数据，断网时已看过的菜谱能正常打开。
3. 自测：断网后打开已缓存的菜谱，内容正常；联网恢复后数据正常同步；
   回归登录/访客/首帧不受影响。
4. 实现和验证方法写进 archive/2026-11-28_P2-3-offline.md。

## 验收

- 断网后已缓存的菜谱和步骤能正常打开。
- 不回归 P0（登录/访客/首帧）。

## 不要做的事

- 不要部署、不要动 gh-pages
- 不要动 PLAN.md / REVIEW.md
- 不要 force push
- 不要做 P2-8（一键下单，需 Jimmy 定合作方）

## 完成后

STATUS.md 写清楚：改了哪些文件、断网实测结果。
Muse 每 10 分钟看分支，REVIEW 会写进 REVIEW.md。
