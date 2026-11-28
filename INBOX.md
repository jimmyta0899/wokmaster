# INBOX — Muse 给 WorkBuddy 的最新指令（v8，2026-11-28）

## 背景

P2-3 已通过（REVIEW.md：PASS，有条件）。P2 按优先级继续，缺失清单见
archive/2026-11-28_P1-audit.md 第三节。部署仍等 Jimmy 直接授权，不卡 P2。
Jimmy 已授权跨目录改源码（INBOX v5 原话），本次继续有效。
Jimmy 决定：按实现目标的最快路径走，不再逐项问他。

## 任务：P2-4 —— 菜谱集/文件夹分享

现在只能分享单条（菜谱/饭局/社区帖），没有 folder/collection/菜谱集实体。
分享基建已完备，加一层集合实体即可复用。把它做出来。

## 做什么

1. 加 collections 表（集合实体：名称、描述、菜谱列表、owner）。
2. 加集合页（查看/编辑一个集合里的菜谱）。
3. ShareService 加 collectionUrl，让集合能像单条一样分享出去。
4. 自测：建集合、加菜谱、分享链接能打开并看到集合内容；
   回归登录/访客/首帧不受影响。
5. 实现和验证方法写进 archive/2026-11-28_P2-4-collections.md。

## 验收

- 能建集合、往里加菜谱、分享集合链接并正常打开。
- 不回归 P0（登录/访客/首帧）。

## 不要做的事

- 不要部署、不要动 gh-pages
- 不要动 PLAN.md / REVIEW.md
- 不要 force push
- 不要做 P2-8（一键下单，需 Jimmy 定合作方）

## 完成后

STATUS.md 写清楚：改了哪些文件、分享链接实测结果。
Muse 每 10 分钟看分支，REVIEW 会写进 REVIEW.md。
