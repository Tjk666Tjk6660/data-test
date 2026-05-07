---
type: think
author: huangshengli
created: '2026-04-23T01:50:51+08:00'
index_state: indexed
---
任何结构变更都应建立在对现状的理解之上。下面按这条思路展开：先说明当前设计与现有 pivot 项目的继承和调整关系，再给出新设计的完整规格和迁移路径。

---

**与现有项目的继承和调整**

现有 index 在 discuss 模块层面是自洽的——字段够用，没有需要打补丁的漏洞。真正需要承接的是**覆盖范围**：本 thread 从 001 起讨论的是 `plan / execution / acceptance / review` 四阶段完整生命周期，而现有 `<slug>-discuss.index.yaml` 从文件后缀起就标记了 "discuss"，只覆盖 plan 阶段的讨论语义；execution / acceptance / review 三阶段尚无数据载体——这和 `vision.md §6` 列出的"关键缺口"（project / task / knowledge 模块未实现）一致。

因此本 thread 实际在设计的，是**为尚未覆盖的三阶段建立数据模型，同时把 plan 阶段的现有实现并入同一套 matter 抽象**。新模型不是修补现有字段，而是在现状之上的承接和扩展。

`vision.md §5` 定义的"多人讨论、单人收口、RESULT 正式发布"治理机制，在新模型里被吸收为"一人拍板切 stage 同时指定 baseline_ref"——人工确认门保留，形式从"发布 RESULT 文档"变成"stage 切换动作 + baseline 锚点"。

**直接继承沿用的部分**

- 原子写机制（`_atomic_write_yaml` 完整复用）
- 两阶段事务标记（`un-indexed → indexed` 翻转 + 启动扫描恢复 `workspace.recover()`）
- slug 生成和 unique hash 逻辑（`sanitize_slug` / `generate_unique_hash`）
- `<cat>/<slug>` 目录层级结构
- post frontmatter 基本字段（`type / author / created / index_state`）
- refs 关系类型（`from` / `refer`；未来的 `blocked_by` 预留位置不变）
- index 作为单一事实源、程序独占写入（AI 只读）的原则
- 一条 timeline 承载所有事件的结构（`{time, event, file, mention?}`）

**需要调整的部分**

- `discussions/` 目录改名为 `plan/`，新增 `execution/ acceptance/ review/ matter/` 平级顶层目录
- 5 状态机（`open/concluded/closed/pending/produced`）整体废除，改为 `current_stage` + stage 级 `lifecycle` 两个维度
- index schema 从 `discussions[0].files[]` 扁平结构改为 `stages.*.files[]` 按阶段分桶，新增 `current_stage / stage_transitions / baseline` 字段
- post type 从 `proposal/reply/result` 改为 `think/act/verify/insight`
- 文件命名后缀从 `-discuss.index.yaml` 改为 `-matter.index.yaml`
- RESULT 机制废除，收口由 stage 切换承担
- API 路径和参数结构重构（`/api/threads/*` → `/api/matters/*`）

**重合度估算**

- 底层基础设施（原子写 / 事务 / 恢复 / hash / slug）约 **90% 直接沿用**
- 业务层（状态机 / index schema / publish / threads）约 **50% 可复用**（主要是模式和分层，具体函数需重写）
- API 和前端改造面约 **50–80%**

**总体与现有 pivot 项目的代码与设计重合度约 40–50%**——属于"在现有地基上加盖楼层"，不是"推倒重建"。

---

**新设计的完整规格**

**目录布局**

```
<data_root>/
├─ plan/           ← 原 discussions/ 改名
│  └─ <cat>/<slug>/001_author_think_hash.md ...
├─ execution/      ← 新增，和 plan/ 平级
│  └─ <cat>/<slug>/...
├─ acceptance/     ← 新增
│  └─ <cat>/<slug>/...
├─ review/         ← 本期目录预留，不写入
└─ matter/         ← 原 index/ 改名
   └─ <slug>-matter.index.yaml
```

`<slug>` 沿用现有 `sanitize_slug` 逻辑，作为 matter 的唯一标识；`<cat>` 保留 category 维度。

**post 类型值**：`think / act / verify / insight`（继承 006）。

**index.yaml schema 样板**

```yaml
matter_id: <slug>
title: "事项标题示例"
category: Pivot
owner: huangshengli
created: '2026-04-21T22:31:13+08:00'
last_updated: '2026-04-22T18:31:52+08:00'

# 主推进阶段指示器。独立字段，不从 stages 推导。
# current_stage 只由 stage_transitions（前进动作）修改；
# stages.<?>.lifecycle 描述每个 stage 自己的活性，允许并行 active。
# 典型场景：current_stage=execution 时，plan.lifecycle 可同时=active
# （执行中发现需重新讨论方案，plan 被重新激活补 think 文档，
#  但主推进仍在 execution）。
current_stage: execution

stage_transitions:
  - from: null
    to: plan
    at: '2026-04-21T22:31:13+08:00'
    by: dengke
  - from: plan
    to: execution
    at: '2026-04-22T18:31:52+08:00'
    by: huangshengli
    baseline_ref: plan/<cat>/<slug>/009_huangshengli_think_85cbbd.md

stages:
  plan:
    lifecycle: active           # 可和 current_stage 并行 active
    baseline: null              # plan 是源头，无上游 baseline
    files:
      - path: plan/<cat>/<slug>/001_dengke_think_e8a253.md
        summary: ""
        importance: null
        refs: []
      - path: plan/<cat>/<slug>/002_liuyu_think_fbeca6.md
        summary: ""
        importance: null
        refs:
          - type: from
            path: plan/<cat>/<slug>/001_dengke_think_e8a253.md
  execution:
    lifecycle: active           # 主推进阶段
    baseline:                   # 本 stage 自己的 baseline，不被后续 stage 冲掉
      ref: plan/<cat>/<slug>/009_huangshengli_think_85cbbd.md
      set_at: '2026-04-22T18:31:52+08:00'
      set_by: huangshengli
    files: []
  acceptance:
    lifecycle: null
    baseline: null              # 进入 acceptance 时写入
    files: []
  review:
    lifecycle: null
    baseline: null
    files: []

timeline:
  # 沿用现有 _timeline_entry 结构 {time, event, file, mention?}
  - time: '2026-04-21T22:31:13+08:00'
    event: dengke created thread
    file: plan/Pivot/Pivot 产品设计 - 设计哲学和基本原理/001_dengke_think_e8a253.md
  - time: '2026-04-22T14:07:30+08:00'
    event: liuyu replied
    file: plan/Pivot/Pivot 产品设计 - 设计哲学和基本原理/002_liuyu_think_fbeca6.md
  - time: '2026-04-22T17:22:54+08:00'
    event: huangshengli mentioned
    file: plan/Pivot/Pivot 产品设计 - 设计哲学和基本原理/004_huangshengli_think_c0b113.md
    mention:
      users:
        - user: dengke
          open_id: ou_xxx
      comments: 请关注这一条
  - time: '2026-04-22T18:31:52+08:00'
    event: huangshengli advanced stage plan -> execution
    file: plan/Pivot/Pivot 产品设计 - 设计哲学和基本原理/009_huangshengli_think_85cbbd.md
  - time: '2026-04-23T10:00:00+08:00'
    event: liuyu reactivated plan
    file: plan/Pivot/Pivot 产品设计 - 设计哲学和基本原理/011_liuyu_think_abc123.md
```

**current_stage 流转图**

```
  ┌──────┐  一人拍板 + baseline   ┌───────────┐  验收通过  ┌────────────┐  完成  ┌────────┐
  │ plan │ ──────────────────────►│ execution │ ──────────►│ acceptance │ ──────►│ review │
  └──────┘                         └───────────┘            └────────────┘        └────────┘
                                                                              [本期 review 不实现]
```

推进时：前一 stage 的 lifecycle 从 `active` 自动置 `completed`；后一 stage 的 lifecycle 从 `null` 置 `active`。

**stage 内部 lifecycle 流转**（每 stage 独立，允许并行 active）

```
  null → active ──搁置──► paused ──关闭──► archived
                   ▲        │
                   └──恢复──┘
                   │
                  （当 current_stage 前进离开该 stage 时）
                   ▼
                completed
```

stage 级 lifecycle 单看会觉得有些冗余（每个 stage 都带一个独立活性字段），但这个冗余是有意保留的——它让"execution 推进过程中重新打开 plan 补充方案讨论"这类跨 stage 并发成为可表达的场景；如果只用一个全局 current_stage，这种并发就没有字段可承载。

---

**代码改动对照清单**（按源码模块，用于快速核对）

| 模块 | 现有职责 | 改动量 | 关键变更 |
|---|---|---|---|
| `posts.py` | post 读写 / 两阶段事务 | ~5% | 仅扩展 `VALID_POST_TYPES` 允许值 |
| `index_files.py::_atomic_write_yaml` | 原子写 | 0% | 完全复用 |
| `recovery.py::scan_un_indexed` | 扫残留 | 0% | 完全复用 |
| `threads.py::generate_unique_hash / sanitize_slug` | hash 和 slug | 0% | 完全复用 |
| `status_machine.py` | 5 状态机 | ~90% | 重写为 `current_stage` 单向机 + stage 级 lifecycle 机 |
| `index_files.py` 其他 | index 写入 | ~70% | create / append 函数按 matter + stage 分发 |
| `publish.py` | 发布路径 | ~50% | 按 stage + type 分发（或统一 `publish_post(type=...)`） |
| `threads.py` 其他 | 扫目录 | ~60% | 扫三层 `<stage>/<cat>/<slug>/`，返回 `MatterMeta / MatterDetail` |
| `recovery.py::_repair_post` | 崩溃恢复 | ~40% | 路径解析改三层 |
| `api/discussions.py` | API | ~50% | URL 从 `/api/threads/*` 改为 `/api/matters/*` 系列 |
| 前端 `StatusBadge / StatusControl` | UI | ~80% | 重构为 stage 显示 + stage 跃迁按钮 + 每 stage 独立 lifecycle 开关 |
| 前端左栏 | 按 category 聚合 | ~20% | 保留 category 维度；tab/filter 层按 `current_stage` 过滤 |
| SQLite | `thread_key` 字段 | ~5% | 字段 rename 为 `matter_key`；内容不变（仍是 slug） |

---

如果权衡下来迁移代价和重构代价不合算，也可以直接新起项目，按上述"新设计的完整规格"从零实现，跳过下面的迁移路径。无论走哪条路，新设计在设计原则和思想上与现有 pivot 完全一致——变化只在数据模型的覆盖范围，底层原则不变。

---

**附录：迁移路径关键步骤**（实现时再展开细节）

1. **前置扫描**：确认仓库里不存在 `status == produced` 的 thread，也不存在 `type == result` 的 post；不存在则跳过对应处理，存在则脚本中断，等待人工介入。
2. **目录改名**：`discussions/` → `plan/`；`index/` → `matter/`；新建空目录 `execution/ acceptance/ review/`（占位）。
3. **post 改名 + frontmatter 改写**：`plan/**/<NNN>_<author>_proposal_<hash>.md` 和 `plan/**/<NNN>_<author>_reply_<hash>.md` 改 `_think_` 后缀；frontmatter `type: proposal / reply → think`。
4. **index schema 转换**：每个 `matter/<slug>-discuss.index.yaml`：
   - 文件名改为 `<slug>-matter.index.yaml`
   - 添加 `matter_id / current_stage: plan / stage_transitions（单条 null → plan）`
   - `discussions[0].files[]` 重构为 `stages.plan.files[]`；其他三个 stages 置空
   - `discussions[0].status` 按对应规则写入 `stages.plan.lifecycle`（`open/concluded → active`、`pending → paused`、`closed → archived`）
   - 所有 path 中 `discussions/` 替换为 `plan/`；post 文件名同步更新
5. **SQLite 字段重命名**：`drafts / read_state / favorites / ai_conversations` 的 `thread_key` 字段 rename 为 `matter_key`；key 值不变。
6. **原子部署**：迁移脚本执行 + 新代码部署同一 PR / 同一天，避免新旧版本并存期；保留 `migration-baseline` git tag 作为回滚锚点。
7. **校验**：启动服务打 `/api/matters` 非 500；扫描所有 index schema 合法；对比迁移前后 matter 数量一致。

每步的具体脚本和边界 case 实现时再计划。
