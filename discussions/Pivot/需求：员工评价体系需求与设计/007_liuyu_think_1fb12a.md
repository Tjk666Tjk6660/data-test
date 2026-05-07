---
type: think
author: liuyu
created: '2026-05-06T14:56:36+08:00'
index_state: indexed
---
# 关于评分体系落地策略的补充建议

整体思路认同，Phase 1 + Phase 2 全部完成后基本能覆盖 003/004/005 的
全部规则，阶段顺序不构成正确性问题。

仅两点补充。

## 一、Phase 2 任务清单里的三条小遗漏

逐条对照需求走下来，Phase 2 的几个 task 需要补一下：

### 1. attribution_basis enum 缺 verify_outcome

§2.5 列了 4 种：file_creator / explicit_mention / at_target / owner_change_reason。

但 005 §4 表里的"隐式评价（动作）"—— verify failed → 被 verify 文件
作者的交付质量 —— 这个归因依据在 enum 里没有对应值。建议补一项
`verify_outcome`。

### 2. "长期 @ 不回"的隐式评价没编入 prompt 规则

005 §4 表里第二条隐式评价是"长期被 @ 不回复 → 协作贡献负向"。Phase 2
任务清单里没明确把这条写进 prompt 系统提示。建议 §2.4 worker 多
subject 任务里把这条规则一起编入。

### 3. AI 派生层与用户原始输入的边界没显式声明

dengke 003 §4 强调：用户原始 annotation 永远只是自然语言；sentiment、
strength、dimension、score_delta、confidence 都是 AI 派生解释，"AI 推断
出的 rating 或 score delta 不能冒充用户原始输入"。

当前实现实际上就是这么做的（commenter_weights 是 admin 配的、不是用户
填的；rating 由 AI 生成），但 Phase 2 §2.5 没把这条边界写成显式约束。
建议在 attribution_basis 旁边加一个 schema 层规则，明确 evidence 表里
哪些列是"原始输入"哪些列是"派生解释"，避免后续误用。

## 二、annotation 数据模型拆分建议剥离出去做

Phase 2 §2.2（comments → mention + annotation 拆分）和 §2.6（前端
annotation 创建入口）本质上是 Pivot 整体数据模型的演进，不属于评分模块
的责任范围。这件事其实由另一条独立工程线在同时推进。

如果评分模块自己重新设计这套数据模型，会出现：

- 重复造轮子：matter_index、publish API、前端创建入口都已经在另一条线
  规划
- 设计冲突风险：两条线各自决策的拆分细节可能不一致，后期收敛代价高

建议从本 matter 里把 mention/annotation 数据模型设计的部分剥离出去，新
起一条 matter 专门推进 annotation 数据模型；本 matter 聚焦在"评分系统
如何消费 annotation"上。

这样：

- annotation matter 关注：拆分边界 / 创建入口 / 数据迁移 / 兼容策略
- 本 matter 关注：归因规则 / 多 subject 评分 / 派生评分层 / admin 视图

两条线各自推进、互不阻塞，在评分系统 v0.4 上线时再做接入对齐。评分模块的
v0.4 范围由此瘦身：不做数据模型拆分、不做 publish_annotation API、不做
前端 annotation 创建入口；改为 annotation 模型的消费方。

## 总结

- Phase 1 + Phase 2 全部完成后基本满足新需求，方向认同
- 遗漏的三条小细节请在 Phase 2 设计稿里一起补
- annotation 数据模型拆分建议剥离到独立 matter 推进
