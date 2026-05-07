---
type: think
author: liuyu
created: '2026-04-23T15:04:26+08:00'
index_state: indexed
---
# 011 架构理解 + 几点问题

读完 011 三遍。先用四张图把我的理解画出来，确认我们在同一个语义上，然后讲我看到的几个问题。如果第一部分理解有偏差，下面的问题就以修正后的理解为准。

## 一、先确认理解（四张图）

### 1.1 静态视角——核心对象长什么样

```
                      一个 matter
                      (事项 = 顶层对象)
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
         ▼                  ▼                  ▼
   current_stage       4 个 stage 桶         timeline
   单一值              plan / execution /     事件流水账
   "主推进位置"         acceptance / review
                            │
                            │  每个 stage 内部:
                            │
                            ├──► lifecycle  (null/active/paused/completed/archived)
                            │    本阶段的活性
                            │
                            ├──► baseline   (指向某篇 think 文档)
                            │    本阶段的基线是哪篇
                            │
                            └──► files []
                                 本阶段的文档清单
                                      │
                                      ▼
                         ┌─────────────────────────────┐
                         │  一篇 MD 文档                 │
                         │  frontmatter:                │
                         │    type: think/act/verify/   │
                         │          insight             │
                         │    author / refs …           │
                         └─────────────────────────────┘
```

三个维度正交：**stage（事项状态）×  lifecycle（阶段活性）×  type（文档用途）**。

### 1.2 物理落地——文件具体住在哪

```
<data_root>/
├── plan/                              ← 所有 matter 的 plan 文档集中
│   └── <category>/<slug>/<NNN>_<author>_think_<hash>.md
├── execution/                         ← execution 文档集中
│   └── <category>/<slug>/...
├── acceptance/                        ← acceptance 文档集中
│   └── <category>/<slug>/...
├── review/                            ← 本期留白
└── matter/                            ← 索引集中
    └── <slug>-matter.index.yaml       ← SSoT
```

**关键**：同一 matter 的文件**分散在三个顶层目录下**，靠相同 slug 串起来。事项全貌只能通过 `matter/<slug>-matter.index.yaml` 拼出来。

### 1.3 动态视角——一个 matter 的一生

以本帖为例假设走完全程：

```
  [T0] dengke 发 001
       系统隐式创建 matter:
         matter_id = "..."
         current_stage = plan
         stages.plan.lifecycle = active
         stages.plan.files = [001]

  [T1..T10] 10 人次追加 reply (002-011)
       每次 stages.plan.files 新增, timeline 追加
       current_stage / lifecycle 不变

  [T11] 某人"推进到 execution"        ◄── 阶段跃迁（一人拍板）
       stage_transitions 追加 {from:plan, to:execution, baseline_ref:<某篇 think>}
       current_stage = execution
       stages.plan.lifecycle = completed
       stages.execution.lifecycle = active
       stages.execution.baseline = <那篇 think>

  [T12..Tn] execution 阶段文档追加
       新 act 文档 (改代码记录) 写到 execution/<slug>/ 下
       ★ 如果中途方案要重新讨论:
          把 plan.lifecycle 重新置为 active
          current_stage 仍是 execution (不回退)
          新 think 文档写到 plan/<slug>/ 下

  [Tm] 推进到 acceptance                ◄── 阶段跃迁 #2
  [Tp] 验收通过                          ◄── 阶段跃迁 #3
  review 本期不实现
```

### 1.4 双维度状态模型（最关键的一张）

```
                      ━━━━  current_stage (单向前进, 唯一)  ━━━━►

                          plan        execution       acceptance      review
                      ┌──────────┬────────────────┬────────────────┬──────────┐
  stages.plan         │          │                │                │          │
     .lifecycle       │  active  │ 可重新 active   │ 可重新 active   │    -    │
                      │          │(补方案讨论)     │(验收异常调方案) │          │
                      ├──────────┼────────────────┼────────────────┼──────────┤
  stages.execution    │   null   │    active      │ 可重新 active   │    -    │
     .lifecycle       │          │   (主推进)     │  (验收后补救)   │          │
                      ├──────────┼────────────────┼────────────────┼──────────┤
  stages.acceptance   │   null   │     null       │    active      │    -    │
     .lifecycle       │          │                │                │          │
                      ├──────────┼────────────────┼────────────────┼──────────┤
  stages.review       │   null   │     null       │    null        │  active  │
     .lifecycle       │          │                │                │(本期不做)│
                      └──────────┴────────────────┴────────────────┴──────────┘
```

横轴是**事项的主推进指针**（一次只在一格，只往右走），纵轴是**每个阶段自己的活性**（可多个同时 active）。这个设计真正贴合现实——"执行中需要重开方案讨论"是单状态机表达不了的空隙。

### 三维度汇总

| 维度 | 回答什么问题 | 可选值 | 数量约束 |
|---|---|---|---|
| current_stage | 事项主推进到哪儿？ | plan/execution/acceptance/review | **一个 matter 只有一个** |
| stages.*.lifecycle | 每个阶段自己的活性？ | null/active/paused/completed/archived | **每阶段各自一个，可并行 active** |
| 文档 type | 某篇文档是啥用途？ | think/act/verify/insight | **每篇文档一个** |

---

理解大致是这样。**如果偏差请指出，以下问题以这个理解为基础。**

## 二、我看到的问题（按紧迫度分三级）

### A 级——011 拍板前应该补齐的 schema 级缺口

#### A-1. execution 阶段的 schema 是空壳

011 里 `stages.execution.files = []` 和 plan 长一模一样。但两者本质不同构：

- plan 是**讨论的线性流**（think 文档按时间一篇接一篇）
- execution 是**任务的图**（task 有父子、有 `blocked_by` 依赖、有子讨论回挂、有分配）

plan 的扁平 `files[]` 装不下 task 树。上线后必然要改 schema——要么做第二次迁移，要么让 schema 变畸形（`files[]` 里塞 parent_id / blocked_by / assignee）。

**建议**：011 拍板前至少给一版 execution.files 的草稿 schema——可以不完整，但要体现"这里会长成图结构"的预留空间。

#### A-2. matter 之间的关系完全缺位

`cross_matter_refs` 在 009 一笔带过，011 不谈。但现实会大量出现：

- **母子**：一个大 matter 拆出多个子 matter
- **阻塞**：matter A 的 execution 等 matter B 的 acceptance（vision §5 已预留 `blocked_by`）
- **派生**：讨论中发现另一事项，开新 matter 但保留来源
- **合并**：两个独立讨论归并为一个

这些不是实现细节，是 **schema 级决定**。一上线就会遇到——"这事从哪儿分出来的？""在等谁？"。vision.md 预留的 `blocked_by` 011 没衔接。

**建议**：index.yaml 里显式预留 `relations: []`，本期 UI 可以不实现，字段要先有。

#### A-3. baseline 没有历史版本

`stages.execution.baseline` 是单字段，意味着**只记当前**。但执行中方向会调整——新 think 成为新 baseline，**旧 baseline 从 index 里消失**。

这会让 Pivot 丢掉一个重要能力：回答"当初为什么定这个方向？什么时候改的？为什么改？"——**这是复盘的核心原材料**。

**建议**：改成 `baseline_history: [{ref, set_at, set_by, reason}]`，最后一条是当前。额外成本极低，信息保全价值大。

### B 级——可以上线后补，但现在不想会积技术债

#### B-1. 双维度状态模型的 UX 没人想过

`current_stage × lifecycle × type` 对后端模型优雅，对用户是**认知负担**：

- "这 matter 在 execution，但 plan.lifecycle 又 active 了，我该去哪儿看？"
- 同一 slug 文件散在三个顶目录，用户要先查 index
- 列表页如何同时展示 current_stage + 多个并行 active lifecycle？UI 语言没设计

011 是纯后端设计。**前端/产品同步出一版 UX 草稿**，验证这套抽象能不能被人理解和使用。如果最后 UI 只能呈现 current_stage、lifecycle 在界面上消失，那 lifecycle 就变成了只有程序和 AI 用的内部概念——这也是一种选择，但要显式决定。

#### B-2. 阶段跃迁的回滚和权限模型缺失

"一人拍板推进"——谁有资格拍板？没有 role 设计。
拍错了（选错 baseline_ref、推进早了）怎么撤回？
撤回时 `stage_transitions` 追加"回退"还是软删前一条？两种是完全不同的审计模型。

现有 5 状态机里 `concluded → open` 的 reopen（带原因）是先例。新模型里这个机制丢了，需要补一个"跃迁回退"的显式设计。

#### B-3. lifecycle 状态机角落未定义

011 画了 `null → active → paused → archived` 和"current_stage 前进离开 → completed"，但**没定义 `completed → active`**。

现实：matter 到 acceptance 发现方案问题要重讨论，plan.lifecycle 此时已是 completed。能不能重激活？如果能，和 paused→active 的"恢复"一样语义吗？

**建议**：显式补 `completed → active` 的"重激活"动作，用独立事件名（如 `reactivated_from_completed`）和 `resumed_from_paused` 区分。

### C 级——产品/架构战略层面

#### C-1. review 本期不实现 = Pivot 差异化延后

Pivot 对 Jira 的护城河在 vision.md §4 的"超越 GTD 部分"——review / evolve / 知识沉淀。**011 本期明确 review 留白**。

后果：第一版上线时，Pivot 能力曲线和 Jira / 飞书项目高度重合。**差异化要等下一期**，市场叙事难讲。

这是产品决策，不是技术决策。但架构拍板会锁定节奏。

**一个折中思路**：本期不做完整 review，但**做"最简 review"——纯 insight 文档沉淀**，让 `review/` 目录不是空的。叙事上说得通："Pivot 从第一天就在沉淀知识。"

## 三、一页纸总结

| 级别 | 问题 | 建议 |
|---|---|---|
| A-1 | execution schema 空壳 | 给一版草稿 schema，体现图结构 |
| A-2 | matter 关系缺位 | index 预留 `relations: []` |
| A-3 | baseline 无历史 | 改成 `baseline_history: [...]` |
| B-1 | UX 没设计 | 前端同步出草稿 |
| B-2 | 跃迁回滚/权限缺失 | 补"跃迁回退"事件 + role 设计 |
| B-3 | lifecycle 状态机未闭合 | 补 completed→active 语义 |
| C-1 | review 延后 = 差异化延后 | 做最简 insight 沉淀打底 |

## 四、建议

1. **A 级三条在 011 修订版里补**——不然会遇到第二次迁移，代价比第一次贵得多
2. **B 级边做边补**，但跃迁回滚至少要有一个最简方案不能空着
3. **C 级是和团队/老板对齐的事**，不是工程决策，但会影响架构拍板的取舍

整体对 011 架构的**基本方向认同**——特别是双维度状态模型在"执行中重开讨论"这个空隙上的创新、SSoT 收敛到 index.yaml 的妥协、迁移路径的责任感，都是扎实的工作。以上几点是希望在拍板前再打磨一轮。
