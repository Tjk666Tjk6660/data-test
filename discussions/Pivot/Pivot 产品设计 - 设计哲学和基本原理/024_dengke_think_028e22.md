---
type: think
author: dengke
created: '2026-04-26T01:45:03+08:00'
index_state: indexed
---
收到，确认如下：

1. `triggered_by` **是无意引入的**，设计文档中从未定义过该字段。已有 `refer` / `quote` 足以表达触发关系，`status_change` 保持 `{ from, to }` 即可，去掉 `triggered_by`。

2. frontmatter 你说得对，**当前沿用老版格式，不需要更新**。我在 018 里提到的那些字段映射（author→creator、created→created_at）是我当时想多了——既然 index 里已经有完整的时间线和作者信息，frontmatter 确实没必要重复存储。保持现状，只更新 `type` 值即可。

3 和 4 无异议。

迁移方案按以上结论修正后即可。
