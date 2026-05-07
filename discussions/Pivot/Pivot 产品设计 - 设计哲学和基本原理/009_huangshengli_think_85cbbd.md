---
type: think
author: huangshengli
created: '2026-04-22T18:31:52+08:00'
index_state: indexed
---
按照 007 的收敛方向：matter 主记录就是 index，不做 SQLite 双存；状态 4 个（`plan/execution/acceptance/review`）和文档类型 4 个（`think/act/verify/insight`）正交；不单独设计 goal，用 index 锚点指定一篇 `think` 文档作为"当前执行基线"，执行中方向调整由新 think + index 更新基线。

以下是按这个方向的具体落地样本，看是否可按此先行实现。

## 目录结构 + 文件命名

一个 matter 一个顶层子目录，内部按 4 状态分子目录。文件命名 `<NNN>_<author>_<type>_<hash>.md`，编号按 matter 内**全局顺序**递增（跨阶段连续），`type` 取 `think/act/verify/insight`。

```
matters/
  <matter_id>/
    plan/
      001_<author>_think_<hash>.md
      002_<author>_think_<hash>.md
    execution/
      003_<author>_act_<hash>.md
      004_<author>_verify_<hash>.md
    acceptance/
      005_<author>_verify_<hash>.md
    review/
      006_<author>_insight_<hash>.md
    index.yaml
```

## index 结构（单一，严守解释层）

```yaml
matter_id: <hash>
title: ...
category: ...
owner: ...
current_stage: plan              # plan | execution | acceptance | review
created: ...
last_updated: ...

stage_transitions:
  - from: null
    to: plan
    at: ...
    by: ...
  # 进入 execution 时追加：
  # - from: plan
  #   to: execution
  #   at: ...
  #   by: ...
  #   baseline_ref: plan/<某篇 think 文件>     ← 被锚定为执行基线

current_baseline: null
# 进入 execution 后形如：
# current_baseline:
#   ref: plan/<某篇 think 文件>
#   set_at: ...
#   set_by: ...

stages:
  plan:
    files:
      - path: plan/001_<author>_think_<hash>.md
        type: think
        summary: ""
        importance: null
        refs: []
      - path: plan/002_<author>_think_<hash>.md
        type: think
        summary: ""
        importance: null
        refs:
          - type: from
            path: plan/001_<author>_think_<hash>.md
  execution: { files: [] }
  acceptance: { files: [] }
  review: { files: [] }

timeline: []                      # matter 级高阶事件
cross_matter_refs: []
ai_commentary: []
```

原则：`stages.*.files[]` 只存引用和 metadata，不嵌内容；`stage_transitions` 和 `timeline` 分开，前者是跃迁锚点，后者是高阶事件流。

## 新建规则（API 层强制归属）

- `POST /api/matters` → 创建新 matter（默认行为，全局新建入口）
- `POST /api/matters/<id>/posts` → 在已有 matter 下追加文档（归原 matter）
- `POST /api/matters?spawned_from=<id>` → 从 matter 派生独立新事项（新 matter + 跨引用）

默认发起讨论 = 新 matter；属于已有 matter 的由 API 路径明示，不让系统推断。

## 其他保留项

- `importance` 字段采用"AI 候选 + 程序写入 + 人工可覆盖"
- execution 单向不回 plan（继承 006）
- 原 `proposal/reply` 位置关系通过 `refs.from` 表达，不另设字段
- 本期只实现 `plan → execution → acceptance` 闭环，`review` 留白

---

是否可按此方案先行落地？如有分歧请直接指出。
