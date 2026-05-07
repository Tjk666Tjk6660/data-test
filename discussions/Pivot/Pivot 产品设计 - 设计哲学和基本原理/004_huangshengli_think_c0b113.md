---
type: think
author: huangshengli
created: '2026-04-22T17:22:54+08:00'
index_state: indexed
---
本期目标：以 matter 为主线打通 plan → execution → acceptance 闭环，review 暂不实现。

## 流程拓扑

```
[plan] 已实现基础能力
  proposal / reply 数据流 ─ AI 整体回顾 ─ 拍板
       │  发布 goal（阶段跃迁锚点）
       ▼
[execution]
  goal ─ AI 拆 task ─ task 推进（含子讨论回挂原 task）
       │  所有 task 完成（阶段跃迁锚点）
       ▼
[acceptance]
  goal + task 产出 ─ 验收清单 ─ 验收记录
       │  验收通过（阶段跃迁锚点）
       ▼
[review]  本期不实现
```

## 待澄清问题

### Q1. matter 创建规则和存储形态

matter 是实体，**创建 plan 时隐式创建 matter**，不需要额外动作。

matter 元数据统一放 SQLite，不进 Git YAML：

```
matter_id / title / category / owner / current_stage
goal_ref / plan_index / execute_index / acceptance_index
cross_matter_refs
```

好处：**monitor AI 扫描全局 matter 矩阵时直接查数据库**，比扫整仓 YAML 轻量。

### Q2. 引入 `matters/` 顶层目录 + 事项解释层物理落地

主张目录拓扑：

```
matters/
  <matter_id>/
    plan/
      001_<author>_proposal_<hash>.md
      002_<author>_reply_<hash>.md
      plan.index.yaml              ← plan 阶段 index
    goal_<hash>.md                  ← plan 产物，平级于阶段目录
    execute/
      tasks/
        t_01_<hash>.md
      execute.index.yaml            ← execute 阶段 index
    acceptance/
      records/
        ...
      acceptance.index.yaml         ← acceptance 阶段 index
```

matter 级元数据走 SQLite（见 Q1），不在目录里留 matter.yaml。

原理 4 "index 是事项解释层"精神保留，物理落地为：

- **事项解释层是概念层，不是单一文件**
- 物理形态：**SQLite matter 行 + plan/execute/acceptance 三个阶段 YAML index** 四者协同
- 各阶段 index schema 不同（plan 描述讨论数据流、execute 描述 task、acceptance 描述验收）
- 分型让原理 7（程序做确定性筛选）更自然——按任务读对应阶段，不用读全量
- 代价：AI 读"matter 完整视图"需 3~4 次 fetch，但绝大多数 AI 任务只关心单阶段

如果坚持"一个事项一个 YAML index"，就要接受单文件随推进膨胀——这是真正的路线分歧，希望重点拍一下。

### Q3. 状态规范——4 阶段 + 阶段内细粒度状态

matter 顶层阶段（4 阶段）和阶段内部状态是两个层级，当前 5 状态机语义需要分层。

主张：matter 顶层 `current_stage` 走 4 阶段；现有 5 状态降级为阶段内细粒度 status：

| current_stage | 阶段内 status |
|---|---|
| plan | open / concluded / closed / pending |
| execution | doing / blocked / done |
| acceptance | pending / passed / failed |
| review | —（本期不实现）|

阶段跃迁独立记录，不和阶段内 status 转移混淆。

### Q4. result（预留）是否就是 goal

vision.md 已预留 `result`（plan 正式收口文档），你文中叫 `goal`，两者语义是否一致？

建议统一为 goal，不能两套名词并存。

### Q5. index 的 `importance` 字段谁决策

`importance` 字段由 AI 自由写、程序控制、还是人工决定？

主张 **AI 提供候选 + 程序写入 + 人工可覆盖**，守住原理 6 "schema 归程序、AI 供候选"。

---

以上五点确认后，进 execution / acceptance 的数据结构细节和 P0/P1 开工顺序。
