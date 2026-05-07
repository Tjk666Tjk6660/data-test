---
type: think
author: xiongjianping
created: '2026-05-06T11:48:28+08:00'
index_state: indexed
---
刚发完上面的内容，立刻命中第三个相关问题——同一主题，但在 **input 侧**。

**问题 3：create_matter 的 `owner` 字段被静默丢弃**

实测：调用 `create_matter` 时传 `owner="yezaiyong"`，工具返回 `ok: true`，无 error/warning，response 也不回显实际 owner 值。但 web 端打开 matter，owner 显示为创建者（xiongjianping），需要手动改成 yezaiyong。

无法从 response 区分以下三种根因：

1. MCP 层 silently 忽略 `owner` 字段，未传给后端
2. `owner` 字段无法 resolve pinyin——schema 只声明 `string`，未说明接受形式；不像 `mentions` 字段会显式解析 pinyin/display/open_id 并在失败时 422
3. 后端策略禁止创建时设 `owner != creator`，silent fallback 到创建者

**主题对照**：

- 问题 1、2：**output 通道**丢字段
- 问题 3：**input 通道**吃参数
- 共同特征：**失败静默**，AI 调完无法察觉，必须靠人肉到 web 端验证才发现

**最小修复建议**：

- 短期：`create_matter` response 回显实际 owner 值，让调用方能 verify
- 中期：`owner` 字段和 `mentions` 一样支持 pinyin/display/open_id 解析，无法 resolve 时 422，与现有协议保持一致
