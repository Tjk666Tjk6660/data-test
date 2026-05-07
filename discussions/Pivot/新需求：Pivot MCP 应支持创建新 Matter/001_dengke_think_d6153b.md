---
type: think
author: dengke
created: '2026-04-28T13:04:25+08:00'
index_state: indexed
---
# 新需求：Pivot MCP 应支持创建新 Matter

## 背景

目前 Pivot MCP 已经可以读取 Matter 列表、解析 Matter URL、读取 Matter 元数据和文件内容，也可以向已有 Matter 追加 timeline 文件。

但在实际使用外部 AI 终端协作时，还有一个关键动作无法通过 MCP 完成：创建新的 Matter。

这会导致一个断点：AI 可以帮助用户讨论和整理一个新需求，但当用户确认内容后，AI 不能直接通过 MCP 发布为新的 Matter，只能让用户手动复制粘贴，或者绕开 MCP 走 HTTP API。

## 目标

为 Pivot MCP 增加创建新 Matter 的工具能力，让外部 AI 终端可以在用户确认后，通过 MCP 直接创建新的 Matter。

这个能力应覆盖“新建 Matter 第一篇文档”的基础闭环：

- 指定 category；
- 指定 Matter 标题；
- 指定第一篇 timeline 文件类型；
- 写入正文；
- 写入 summary；
- 创建 Matter index；
- 返回新 Matter 的 URL 和关键信息。

## 需求描述

### 1. 增加 create_matter 工具

Pivot MCP 应新增一个工具，例如：

```text
create_matter
```

建议入参包括：

```json
{
  "category": "Pivot",
  "title": "新需求：为 Matter 增加 Owner 责任人机制",
  "type": "think",
  "summary": "为每个 Matter 增加 owner 字段，并在列表、详情页和 timeline 中展示与记录 owner 变更。",
  "body": "Markdown 正文",
  "owner": null
}
```

其中：

- `category` 必填；
- `title` 必填；
- `type` 默认可为 `think`；
- `summary` 建议必填；
- `body` 必填；
- `owner` 可选，如果为空则由服务端按默认规则处理，例如 owner 默认为创建者。

### 2. create_matter 应复用现有 Matter API 写入逻辑

MCP 不应重新实现一套 Matter 创建逻辑。

create_matter 应作为薄适配层，底层复用现有 authenticated Matter API，由服务端完成：

- 创建 Matter 目录；
- 创建 index；
- 创建第一篇 timeline 文件；
- 写入结构化字段；
- 处理 creator / owner / created_at / updated_at；
- 返回标准 Matter 信息。

这样可以保持 Web、HTTP API、MCP 三个入口的数据模型一致。

### 3. 创建前必须由用户确认

由于 create_matter 会产生新的正式 Matter，AI 客户端在调用 MCP 工具前，应先把拟发布内容展示给用户确认。

MCP 工具本身可以不负责交互，但工具说明中应明确：

- AI 应先展示 title、category、summary、body；
- 用户确认后再调用 create_matter；
- 不应在未确认的情况下自动创建 Matter。

这与当前 create_file 工具的写入协议保持一致。

### 4. 返回结果应包含可打开的 Matter URL

create_matter 成功后，应返回：

```json
{
  "matter_id": "...",
  "category": "Pivot",
  "title": "...",
  "url": "https://pivot.enclaws.com/m/...",
  "first_file": "discussions/Pivot/.../001_xxx_think_xxx.md",
  "summary_for_ai": "已创建 Matter：..."
}
```

其中 `summary_for_ai` 应适合 AI 原样转述给用户。

### 5. 错误提示应清晰

常见错误应给出明确原因，例如：

- category 不存在；
- title 为空；
- Matter id 冲突；
- 用户未登录或 token 无效；
- body 为空；
- 当前用户没有创建权限。

错误不应只返回底层异常字符串。

## 验收标准

- MCP 工具列表中出现 `create_matter`；
- 外部 AI 终端可以通过 MCP 创建 category 为 `Pivot` 的新 Matter；
- 创建时可以指定 title、summary、body 和第一篇文件 type；
- 未指定 owner 时，服务端按默认规则设置 owner；
- 创建成功后，Matter 列表中出现新 Matter；
- 新 Matter 的详情页可以正常打开；
- 新 Matter index 和第一篇 timeline 文件均正确生成；
- MCP 返回新 Matter URL、matter_id、first_file 和 summary_for_ai；
- AI 客户端可以把 summary_for_ai 原样转述给用户；
- 常见错误返回清晰提示，而不是底层异常字符串。
