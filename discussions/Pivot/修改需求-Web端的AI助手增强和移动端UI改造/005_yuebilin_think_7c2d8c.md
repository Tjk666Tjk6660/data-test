---
type: think
author: yuebilin
created: '2026-04-23T12:45:42+08:00'
index_state: indexed
---
方向已对齐：**先完成 index 重构（B），再做 AI 助手改造（A）**，两件活都在本轮一起交付，不走"旧 schema + 适配器"的过渡路线。

正式设计文档已写完，放在仓库 `AI-docs/designs/` 下：

- **B（index 重构）**：`AI-docs/designs/2026-04-23-index-refactor-design.md`
- **A（AI 助手改造）**：`AI-docs/designs/2026-04-23-ai-autonomous-context-design.md`

为评审方便，下面给出两份文档的要点速览和待评审的决策点。正文细节（字段定义、迁移映射表、接口、实施步骤、测试、风险）请见两份文档本体。

## 一、总体节奏

- **阶段一：B 重构**——把 `thread / discussion` 模型迁移到 `matter` + timeline-first 新 schema（依据 `AI-docs/pivot-product.md` 第六至九、十节）
- **阶段二：A 改造**——在新 schema 上直接实现 tool-calling loop 的自主读取

移动端重构已在主干代码里完成，本次不再规划，我会拉主干核对已合并的改动。

## 二、B 的关键要点

- **新 schema**：一个 matter 一个 index 文件，头部保留 matter 快照（id / title / current_status / created_at / updated_at），下面是唯一的 timeline，每条 item 对应一篇文件，含 `quote / refer / type / summary / comments / status_change / verifications / outcome` 等字段
- **状态迁移**：覆盖五态 → 六态全部映射（`open → planning`、`concluded → planning`、`closed → cancelled`、`produced → finished`、`pending → planning`），`concluded` 与 `produced` 迁移后进入人工审查流程，不一刀切
- **引用关系**：旧 `refs:[{type:from,path}]` → 新 `quote`（单值）；旧 `refs:[{type:refer,path}]` → 新 `refer`（数组）
- **前端完整交付**：matter 详情页改为时间线视图 + 文件卡片"基于此新增 think/act/verify"入口 + `verify` 与 `result` 专属展示 + "生成 Result" 页面级入口；同步修讨论列表、侧栏、状态徽章
- **迁移与兼容**：旧 API 路由保留为别名、数据库 schema 不动（`favorites / read_state / inbox` 仍按 `(category, slug)` 键），只做 YAML index 文件层面的迁移
- **特殊文件**：`SUMMARY*.md / RESULT*.md` 不入 timeline，保留为 legacy artifact，是否转写为 `insight / result` 由单独脚本处理
- **前端测试栈**：当前项目没有 vitest / @testing-library，本次一并安装

## 三、A 的关键要点

- **两个只读 tool**：`read_matter_index(matter_id)` 返回 timeline 概览，`read_file(path)` 读单篇全文
- **AI 读取边界**（严格两维度）：目标文件 + 当前 matter timeline 上所有文件 + 它们 `quote / refer` 指向的文件；越界由服务端硬性拒绝
- **chat 接口升级为 tool-calling loop**：SSE 事件扩展 `tool_call / tool_result` 两类（tool_result 内容不推前端，只显示"AI 读了哪些文件"）
- **下线手工入口**：`FileTreeBrowser.tsx`、`/api/ai/files` handler、`fetchAIFiles` 一起删；`reply_target` / `reference_files` 的请求字段在灰度期兼容后移除
- **预算初值**：单次会话最多 6 次 tool call;单文件上限沿用 `MAX_CONTEXT_CHARS`；超额截断策略保留 system + 最近 N 轮消息 + 最后一次 tool_result

## 四、待评审拍板的决策点

B 设计文档（评审关注点在文档第十节）：

1. frontmatter 旧 type（`proposal / reply / comment`）是否全部映射为 `think`，还是按内容细分
2. 新旧 API 路由并存窗口（30 天后拆 vs 无期限保留）
3. 迁移脚本部署时是否需要停机窗口
4. matter 详情页顶部"生成 Result"按钮的位置与文案取舍
5. 回帖按钮文案"回复"过渡保留多久

A 设计文档（评审关注点在文档第十节）：

1. 预算 6 次 tool call 是否合适（vs 10 次）
2. `tool_result` 内容不推前端会不会让 AI 阅读过程不够透明
3. `reply_target / reference_files` 旧请求字段兼容期多长
4. `[[GENERATE_REPLY_DRAFT]]` tag 是否借本次改造升级为 tool

## 五、下一步

请评审两份文档。有改动意见直接在 Pivot 讨论里回帖，我按反馈更新后再进入实施（M1 开始：B 的 `server/matter_index.py` + 单测）。

@刘昱 @邓柯 辛苦过一下。
