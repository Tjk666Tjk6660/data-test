---
type: think
author: yuebilin
created: '2026-04-23T11:48:28+08:00'
index_state: indexed
---
已对齐做法：**先完成 index 重构，再做 AI 助手改造**，不再走"旧 schema + 适配器"的过渡路线。下面说明两件活的实施方向，以 index 重构（下称 B）的设计为主，AI 改造（下称 A）只列要点。正式细化的接口定义和实施步骤随后补齐。

## 一、整体节奏

- **第一阶段：index 重构 B**——把 `thread / discussion` 模型迁移到 `matter` + timeline-first 的新结构，依据 `AI-docs/pivot-product.md` 第六至九节。
- **第二阶段：AI 改造 A**——在新 schema 稳定后，直接基于 `quote / refer / timeline` 做 tool-calling loop，不再写兼容层。

A 的代码只写一次，B 重构完 A 就可以直接对接，符合"代码不返工"的目标。

移动端重构已由你提交的主干代码覆盖，本次开发不再规划，我会先拉主干核对一遍已合并的改动再开工。

## 二、B：index 重构的实施方向

### 1. 第一版范围

按 `pivot-product.md` 第十节的判断，第一版只优先支撑两条最小闭环：

- `planning / executing -> paused -> 恢复`
- `planning -> executing -> finished / cancelled -> reviewed`

对应的最小结构就是第八节描述的 `matter` 顶层头部 + `timeline[]`，每条 item 就是一篇文件，字段只保留 `file / created_at / creator / owner / type / summary / quote / refer / verifications / comments / status_change`。第八.4 节明确排除的字段（`goal / task / transitions / files / anchor / importance / title` 等）第一版都不进入 schema。

### 2. 关键工作项

- **schema 定义与校验**：新增 `matter.timeline[]` 的 YAML schema，给出程序侧的读写接口（`get_matter(id)` / `list_timeline(id)` / `append_timeline_item(...)` 等），访问层集中到一个模块，避免各处直接 `yaml.safe_load` 再按字段硬读。
- **数据迁移脚本**：把现有每个 `<slug>-discuss.index.yaml` 的 `discussions[].files[]` + `timeline[]` 合并成新 schema。`refs: [{type: from, path}]` 映射成 `quote`；`refs: [{type: refer, path}]` 映射成 `refer[]`。现有 `type: proposal / reply` 默认映射为新类型系统中的 `think`，具体映射规则在设计文档里给表。
- **index 写入端重构**：现在写 index 的入口在 `server/index_files.py`（`append_standalone_mention / change_thread_status / _build_reply_refs` 等），全部改为写新 schema。保留原函数名但内部改写，或新增 `matter_index.py` 做完全替代，二选一在正式设计里定。
- **API 与路由**：现有 `/api/threads/{category}/{slug}/...` 路径是否改名为 `/api/matters/{id}/...`，第一版建议保留旧路径作为别名、同时提供新路径，避免一次性冲击前端和外部调用方。
- **前端过渡**：`AI-docs/pivot-product.md` 第十三节给出了 matter 详情页的时间线视图方向。第一版可以先让后端直接返回 timeline-first 结构，前端仍按帖子列表渲染；"基于此新增 think / act / verify"的文件卡片入口（第九.1.1 节）另起一条工作项迭代。

### 3. 风险点

- 现有数据量不大，迁移脚本的复杂度主要在**类型映射**上（旧 `proposal / reply` 映射到新 `think / act / verify / result / insight` 有语义损失），建议迁移脚本保留一份原始 type 到 `legacy_type` 字段，避免不可逆。
- `_build_reply_refs` 现在只登记用户勾选的 references，历史数据里 `quote` 可能缺失。迁移时按 `reply_to` 字段补齐 quote，补不到的标成空并记录。
- Git 同步 / 持久化相关逻辑（`recovery.py`、`ai_conversations`）在本次重构中基本只读不写，但要覆盖测试。

## 三、A：基于新 schema 的 AI 改造要点

B 上线后，A 的实现会基于新 schema 直接做，主要工作项如下，正式设计文档会补细节：

1. **两个只读 tool**
   - `read_matter_index(matter_id)`：返回当前 matter 的 timeline 结构（头部 + 所有 item 的 `file / type / summary / quote / refer`）
   - `read_file(path)`：按路径读取单篇文件。**路径必须通过当前 matter 的 timeline 或其中某项的 `quote / refer` 可达**，越界直接返回错误。
2. **AI 读取边界 = 两维度**：当前目标文件 + 该 matter timeline 上所有文件 + 它们 `quote / refer` 指向的文件。无逃生舱入口，前端不再提供手工选 `reference_files`。
3. **chat 接口升级为 tool-calling loop**：SSE 事件模型扩展两类事件（`tool_call` / `tool_result`），前端消费这些事件做"AI 已读取哪些文件"的只读展示。
4. **复用点**：现有 `server/ai/context.py` 的 `build_context_from_files` 作为 `read_file` tool 的实现底座，路径校验和长度限制继续复用。
5. **持久化**：`ai_conversations` 扩展字段记录本次会话读取过的文件列表，便于重连和审计。

## 四、读取预算与频控（初版，压测后调优）

- 单次会话最多 **6 次** tool call（防卡死，超过返回错误）
- 单次 `read_file` 文件体积上限继续沿用 `MAX_CONTEXT_CHARS`
- `read_matter_index` 单独计一次，返回体按摘要字段拼装，不计入 `MAX_CONTEXT_CHARS`
- 累计读入超过 `max_context_tokens` 后截断策略：保留最新 system prompt + 最近 N 轮消息 + 最后一次 tool_result，旧的 tool_result 压缩为"曾读取：file_a, file_b..."字符串

这些数值先上线，按实际运行数据调整。

## 五、下一步

等本帖评审通过后，我会出两份正式设计文档，分别针对 B 和 A：

- **B 的设计**：新 schema 详细定义、迁移脚本流程、访问层接口、API 路由过渡方案、测试覆盖清单
- **A 的设计**：tool 接口定义、tool-calling loop 的数据流、SSE 事件协议、前端展示与交互

两份文档会放到 `AI-docs/designs/` 下并在本帖追加链接，方便评审。

@刘昱 @邓柯 请就本帖方向给意见，重点是：第一阶段 B 的范围（是否就按 `pivot-product.md` 第十节描述的两条最小闭环）、迁移策略（`legacy_type` 字段是否保留）、以及 API 路由过渡是走别名还是直接换路径。
