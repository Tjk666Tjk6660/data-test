---
type: think
author: yuebilin
created: '2026-04-23T10:48:40+08:00'
index_state: indexed
---
收到，先把需求分析和几个需要确认的点列出来，方向对齐后再出正式设计文档和实施计划。

## 一、对现状的核对

看了一下现有代码，需求里涉及的能力不是全部从零做，有几件已经到位：

- **"点击 AI 回复自动带入当前帖子"已经实现**。前端 `AIPane` 通过 `pendingReplyTarget` 机制会把当前帖注入 `replyTarget`，我本地也体验过。这条不需要额外开发。
- **index 文件本身存在且持续维护**。`server/index_files.py` 在建 thread、回帖、@人、改状态时都会写 `{slug}-discuss.index.yaml`，格式包含 `files[].summary/refs` 和 `timeline`。
- **受控文件读取的底子在**。`server/ai/context.py` 的 `build_context_from_files` 已经限定路径必须是 `category/slug/filename.md`、只从 `discussions_dir` 下读。

所以这次要真正动的是三块：**AI 读取链路**（从"一次性拼好上下文"改成"对话中按需取"）、**前端主流程去配置化**、**移动端 AI 面板重做**。

## 二、核心架构问题：AI 怎么"自主"读 index

需求文档里多次出现"AI 再自主读取 index / 相关文件"这类表述，这是整个方案最关键也是最容易被误读的一点。想先把机制讲清楚，避免到时候实现不对齐。

"自主"在工程上**只能通过 tool use（工具调用）实现**。具体是：

1. 服务端在 system prompt / `tools` 字段里，把可用工具告诉模型（例如 `read_thread_index(category, slug)`、`read_file(path)`），并写明调用原则（先答用户、信息不足再调、不要一次调太多等）。
2. 模型返回的不是文本，而是结构化的 `tool_calls`。
3. 服务端执行 tool，把结果塞回对话再请求一轮，直到模型不再调工具、开始输出正文。

这意味着现在的 `/api/ai/threads/{category}/{slug}/chat`（单轮 SSE、pre-baked system prompt）要升级为 **tool-calling loop**。这是架构级改动，**不是前端小改也不是 prompt 调优**。如果不做这一层，"AI 自主"就只能退化成"服务端强制把 index 附进去"，那其实是另一种方案，而不是需求文档所描述的那种。

**边界由服务端硬性把住，不靠模型自觉**。例如 `read_file` 实现里校验路径三段式、校验命中白名单（同 thread 或当前 thread index 的 `refs[]`），越界直接返回错误。模型看到错误会自己放弃，这才是"不让 AI 自由扫仓库"的真正落地方式。

## 三、待确认的问题

下面几条是做设计之前必须先对齐的，否则中途返工概率很大：

### 1. tool use 升级是否算在本次需求内

这是最关键的一票。

- 如果**算**：要改 `/api/ai/.../chat` 接口、新增 index/file 读取工具、前后端调整消息与事件模型、补测试。
- 如果**不算**：退化方案是服务端在每次请求时自动把当前 thread 的 index 摘要附进 system prompt，不做真正的 tool use。这种做法容易实现，但"AI 按需继续读相关文件"那一步做不到。

倾向认为**需求文档真正想要的是前者**，请确认。

### 2. 跨 thread 引用的边界（重要）

现状：前端允许任意挑最多 4 个 reference file，**跨 thread 完全自由**，不要求 index 里有登记。

需求文档的措辞是"支持 **index 指向的** 跨 thread 文件"——这相对于旧行为是**收窄**，会出现这样一类场景做不到：

> 用户在 A thread 写回复，想引用 B thread 的某帖，但 A 的 index `refs` 里没登记那篇。旧流程可以手工挑，新流程 AI 读不到。

可选做法：

- **A. 严格按文档**：AI 只能读 index `refs` 白名单内的跨 thread 文件。功能收窄。前提是 refs 完整度要够，而目前 `_build_reply_refs` 只登记回帖时用户主动勾选的 references，没勾就不会出现在 refs 里。
- **B. 保留逃生舱**：默认 AI 自主 + refs 白名单；同时前端保留一个**次要**入口，允许用户为**本次会话**临时追加 1–2 个任意文件进白名单，不改 index，只对这次对话有效。

我的倾向是 **B**。理由：需求原文要解决的是"用户不再被迫手工组装上下文"，不是"用户不允许指定文件"。B 在满足需求的同时保留兜底能力，风险更小。

请决定采 A 还是 B，或给出第三种意见。

### 3. 读取预算 / 频控

为防止模型在 tool-loop 里卡住或读得太多，需要定死几个数字：

- 一轮对话最多几次 tool call（建议 ≤ 6）
- 单次 `read_file` 的文件体积上限（沿用 `MAX_CONTEXT_CHARS` 或单独设）
- 多次累计读入后的 context 截断策略

这些值希望由你和刘昱拍一下，不然我只能按自己经验设。

## 四、初步设计方向（待上面确认后细化）

在上面几点敲定前不写正式设计文档，先给一个骨架让你判断方向对不对：

**后端**
- 新增只读 API：`GET /api/ai/index/{category}/{slug}`、`GET /api/ai/file/{category}/{slug}/{filename}`，访问控制复用 `discussions_dir` 路径校验 + 白名单判定。
- `chat` 接口改为 tool-calling loop，SSE 事件类型扩展（新增 `tool_call`、`tool_result`，前端可展示"AI 读取了哪些文件"）。
- `ChatRequest.reference_files` 字段行为调整（按问题 2 的结论决定是保留只读入口还是彻底下线）。
- 现有 `build_context_from_files` 保留作为 tool 的实现复用点。

**Web 前端**
- 移除手工选 `reply_target` / `reference_files` 的主流程 UI（`FileTreeBrowser` 的相关调用点）。
- 新增"AI 已读取文件"只读展示区（消费后端的 tool_call/tool_result 事件）。
- 按问题 2 结论决定是否保留逃生舱入口。

**移动端**
- 需求原文对移动端只给了方向（"更清晰""更适合小屏""路径更短"），没有具体形态。我先搞一个出来看看
## 五、风险

`index_files.py`、持久化、Git 同步这几块是高风险区。本次改动主要读不写，可以降低接触面。但 tool-loop 会影响 SSE 流和对话持久化（`ai_conversations`），测试要覆盖多轮 tool call 下的中断、重连、草稿生成等路径。

上面问题回复后，我出正式设计文档，附接口定义和数据流图，再进入实施。

@刘昱 麻烦看一下上面三个问题，尤其问题 1（tool use 是否算需求内）和问题 2（跨 thread 引用边界），这两条直接决定工作量和交互形态。
