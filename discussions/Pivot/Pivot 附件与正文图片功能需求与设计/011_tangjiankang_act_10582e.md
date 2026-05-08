---
type: act
author: tangjiankang
created: '2026-05-08T14:03:01+08:00'
body_source: manual
index_state: indexed
---
# Pivot Matter Index Schema(权威源)

> ⚠ **本文是 matter index 数据结构的单一权威源**(single source of truth)。
>
> - 任何 AI 应用读 index 给 LLM 看时,**应注入本文档作为字段语义说明**
>   (公司日报 / 个人日报 / AI 助手 chat / MCP tools / 未来新 AI 应用 都从这里读)
> - matter index 字段集若有变动(`server/matter_index.py` 的
>   `_ITEM_KEY_ORDER` / `_INVALIDATION_EVENT_KEY_ORDER` / `_OWNER_CHANGE_KEY_ORDER` /
>   `_MATTER_KEY_ORDER` 等常量),**必须同次 commit 更新本文档**;否则程序化测试
>   `server/tests/test_index_schema_doc.py` 会 fail
> - 本文档只描述"数据结构层"(字段集 + 字段语义 + 字段间关系);
>   "在某个具体场景下如何使用这些字段"由各 AI 应用自己的 prompt 写业务规则


![9094f1af-a30b-4fe7-8d0e-fbef93753e7b](attachment://att_6Oy9zs90tkx7)
