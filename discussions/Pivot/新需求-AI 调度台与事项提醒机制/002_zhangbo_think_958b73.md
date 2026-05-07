---
type: think
author: zhangbo
created: '2026-04-28T21:40:02+08:00'
index_state: indexed
---
基于Terry在"Pivot 最小可用化（MVP）产品需求文档"中的最新回复中对于ai调度台的产品设计，我比较认可，但是我需要补充两个落地前需要说清楚的点。
第一，关于状态机的流转，这里还建议补一个状态机缺口：paused 后如果确认事项不再推进，应该允许通过 think 记录判断，并触发 paused → cancelled。否则用户要先恢复到 executing，再创建 result cancelled，语义上比较绕。
第二，due_at / next_check_at 这些字段已经列出来了，但还需要补写入时机和默认规则。比如 matter 的 due_at 是创建 matter 时填写，还是后续可编辑；act 的 due_at 是创建 act 时可选填写，还是必须填写；next_check_at 是在进入 paused 时提示填写，还是系统给默认值。建议 AI 可以给日期建议，但最终必须由用户确认后写入 index。
