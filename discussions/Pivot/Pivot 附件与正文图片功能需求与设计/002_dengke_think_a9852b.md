---
type: think
author: dengke
created: '2026-05-06T23:00:17+08:00'
body_source: ai
index_state: indexed
---
针对第十章 AI 集成部分，建议将方案调整为**"元数据透传 + Agent 工具自决"**，核心思路是明确附件的首要价值是**人与人之间的协作**，AI 只是辅助阅读者，不应强制介入。

具体落地建议如下：

**1. 彻底简化 §10.2（移除后端能力判断）**
- **不再维护** `supports_vision` 开关和 known-model 白名单，避免模型更新带来的维护负担和第三方接口兼容性风险。
- **永远不主动发送** 原图或解析后的全文，不再在 prompt 中注入复杂的 `image_url` 或大段文本。

**2. 仅注入附件元数据**
- 每次 timeline item 传给 AI 时，只附带附件清单：`[id, format, size, description]`。
- 其中 **`description`（用户填写的内容说明）** 是 AI 判断"值不值得读"的唯一依据。建议前端上传时强引导或轻量必填。

**3. 依靠 MCP 工具让 AI 自决**
- 提供 `read_attachment(id)` 工具，Prompt 中提示 AI 根据自身上下文余量、模型能力（是否 vision/解析文档）及元数据相关性，**自行决定是否调用**。
- 纯文本模型看到 `format: image/*` 通常会跳过，仅依赖 description 回答；多模态模型认为关键时会主动调取原图。

**4. 收益**
此方案将 v1 的复杂度从"后端规则引擎"降维到"元数据透传 + 工具自决"，既避免了 token 爆炸和解析崩溃的坑，又完美契合 LLM Agent 的工具调用范式，开发成本最低且扩展性最好。
