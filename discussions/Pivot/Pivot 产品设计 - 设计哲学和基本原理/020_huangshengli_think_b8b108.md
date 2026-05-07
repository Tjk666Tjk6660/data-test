---
type: think
author: huangshengli
created: '2026-04-25T14:41:38+08:00'
index_state: indexed
---
简洁回复，第2点目前支持，第3点又前端控制，主要说下第1点，index增加字段来支持，前端及 `drafts` 模块对此不感知，纯由后端通过 index 写入派生逻辑处理：
   - **字段落位**：反写设计落实到 `verifications_received` 字段，且仅挂在 `act` 名下。
   - **验证者归属**：`verified_by` 字段直接取自 `verify.owner`。
   - **写入机制**：反写操作会在创建 `verify` 的同一次**原子写入**中完成，确保 `act` 上的该字段为 **append-only**，保证数据一致性。
   **结构示例参考：**
   ```yaml
   - file: discussions/Pivot/auth-redesign/004_huangshengli_act_e7980c.md
     created_at: '2026-04-24T14:12:08+08:00'
     creator: huangshengli
     owner: dengke
     type: act
     summary: 子任务 B：cookie 持久化
     verifications_received:
       - verify_file: discussions/Pivot/auth-redesign/005_huangshengli_verify_920bf2.md
         verified_at: '2026-04-24T14:12:16+08:00'
         verified_by: huangshengli
         judgement: failed
         comment: cookie 跨域场景未覆盖，需返工
       - verify_file: discussions/Pivot/auth-redesign/008_huangshengli_verify_7a3b1c.md
         verified_at: '2026-04-24T16:00:00+08:00'
         verified_by: huangshengli
         judgement: passed
         comment: cookie 跨域场景已补齐，通过
   ```
