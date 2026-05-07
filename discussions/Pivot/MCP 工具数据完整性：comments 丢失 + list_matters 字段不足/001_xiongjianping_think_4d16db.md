---
type: think
author: xiongjianping
created: '2026-05-06T11:18:14+08:00'
index_state: indexed
---
# MCP 工具数据完整性：comments 丢失 + list_matters 字段不足

最近用 MCP 跟踪 matter 进度，命中两处 schema 层的数据完整性问题，根因相同：**后端已经返回，MCP 的 pydantic schema 没声明，被序列化层 filter 掉。**

**问题 1：get_matter 丢失 comments**

后端 `/api/matters/{id}` 已返回每条 timeline item 的 `comments` 数组（含 `author_display` / `body` / `mentions_display`），但 `server/mcp/schemas.py` 的 `TimelineItem` 没声明 `comments`，pydantic 序列化时直接丢弃。Agent 只能 `add_comment` 写、读不到任何评论——同事在评论区"解析已加"这类回应全部漏掉。

最小修复：`TimelineItem` 加一行 `comments: list[dict] | None = None`。

**问题 2：list_matters 字段不足**

当前 `list_matters` 不返回 `owner` 和 `summary`，AI 列完后必须对每个 matter 再调一次 `get_matter` 才能判断 owner / 相关性。N+1 次调用，浪费 context 和延迟，长列表场景尤其明显。

最小修复：list 接口 schema 增加 `owner` 和 `summary`（后端若已返回则仅补 schema；若未返回则同步补查询字段）。
