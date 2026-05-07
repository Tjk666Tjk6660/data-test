---
type: act
author: yezaiyong
created: '2026-05-06T11:32:32+08:00'
body_source: manual
index_state: indexed
---
# MCP schema 数据完整性修复                                                                                                                                                                                                       
                                                                                                                                                                                                                                    
  ## 背景
                                                                                                                                                                                                                                    
  跟踪 matter 进度时命中两处 schema 层 silent drop——后端已返回字段，MCP 的 pydantic schema 没声明，pydantic v2 默认 `extra='ignore'` 直接吞掉。                                                                                     
                                                                                                                                                                                                                                    
  ## 改造点                                                                                                                                                                                                                         
                                                                                                                                                                                                                                    
  ### 1. `get_matter` 找回评论                                                                                                                                                                                                      
                                                                                                                                                                                                                                    
  **问题**：后端 `/api/matters/{id}` 已渲染每条 timeline item 的 `comments`（带 `author_display` / `body` / `mentions_display` / `mentions_view`），但 MCP 的 `TimelineItem` 没声明 → AI 只能 `add_comment`                         
  写、读不到评论区任何回应。同事在评论区"解析已加"这类回应全部漏掉。                                                                                                                                                                
                                                                                                                                                                                                                                    
  **修复**：`server/mcp/schemas.py` `TimelineItem` 增加字段：

  ```python
  comments: list[dict] | None = Field(
      default=None,
      description="Comments attached to this file (each with author_display, body, mentions_display)...",
  )

  2. list_matters 透出 owner / summary

  问题：后端 _summarize_matter 已经返回 owner、last_summary，但 MCP 的 MatterListItem 只声明了 5 个字段 → AI 列完 matter 后必须对每个再调一次 get_matter 才能判断 owner / 相关性，N+1 调用，长列表场景尤其浪费。

  修复：
  - server/mcp/schemas.py MatterListItem 增加 owner: str | None 与 summary: str | None
  - server/mcp/tools.py tool_list_matters 把后端的 owner / last_summary 透出（last_summary → summary，语义更贴近 MCP 调用方）

  3. 回归测试

  server/tests/test_mcp_tools.py 新增两条测试锁住这次的 silent drop：

  - test_list_matters_surfaces_owner_and_summary
  - test_get_matter_preserves_comments

  验证

  - 单测：pytest server/tests/test_mcp_tools.py → 35/35 通过
  - MCP 实测（需重启 MCP server 后）：
    - 列 matter 时返回的 items[*] 含 owner / summary 即生效
    - 看一条已有评论的 matter，timeline[*].comments 出现且非空即生效
