---
type: think
author: huangshengli
created: '2026-04-25T21:53:56+08:00'
index_state: indexed
---
针对迁移方案的评审反馈：

1. **comment 不作为独立 timeline 事件**： 规则已对齐。 
2. **concluded / produced → act + 触发 status_change**：  状态迁移规则已对齐。关于 `triggered_by` 字段，matter 设计中并未定义，按理说产品设计文档中现有 `refer` / `quote` 已足以表达 trigger 含义，所以triggered_by是无意引入的吗，请确认。 
3. **帖子 frontmatter 需按新版格式更新**： type 字段映射（proposal → think 等）已对齐。其余字段映射（如 author → creator，created->created_at），是这次回复里新加的吗?我感觉是老板你记错了吧，MD文档frontmatter并未有任何更新(除了type的值),当前都是沿用老版frontmatter。
4. **上线前验证流程调整**：无异议，已对齐。
