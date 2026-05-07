---
type: think
author: huangshengli
created: '2026-04-23T15:43:45+08:00'
index_state: indexed
---
基于这轮对齐，把当前产品设计文档（`pivot-product.md` + `pivot-interface.md`）落到执行侧的实施路径整理如下，供评审。

## 总体口径

- `discussions/<category>/<slug>/` 磁盘结构继续沿用，文件名与 git 历史不动。
- 老 INDEX 不做兼容；采用**上线前一次性数据迁移**。
- 接口按 `pivot-interface.md` "Matter API（设计草案）" 实现，不再另起字段集。
- Threads 旧接口开发期保留，UI 切到 Matter 之后再谈下线节奏。
- AI-monitor 独立设计，本路径只负责按阶段打好埋点和预留 UI 槽位。

## 实施阶段

### Phase 1 · 后端状态与 `matter_index` 模块

落地 6 态状态机、5 类文档类型、按 §八 示例的 INDEX 读写模块。状态 × 类型允许矩阵与触发文件约束落在 writer 层，保留 `write_session` 锁与原子 tmp+rename。无用户可见变化。

### Phase 2 · Matter API（后端）

按 `pivot-interface.md` 草案实现：

- `GET /api/matters`、`GET /api/matters/{matter_id}`
- `POST /api/matters`
- `POST /api/matters/{matter_id}/files`（含 think / act / verify / result / insight 类型专属体）
- `POST /api/matters/{matter_id}/result`

服务端最小校验按草案"最小服务端校验建议"一段。Threads 旧接口保留不改。

### Phase 3 · Matter 详情页（前端主导）

前端把 `ThreadDetailPane` 切为 `MatterDetailPane`，右栏换成时间轴 + 文件卡片。文件卡片提供"新增 think / act / verify"三入口，自动把当前文件填到新文件的 `quote`；页面级"生成 Result"入口带收口确认弹窗。`StatusControl` / `StatusBadge` 按 6 态配置化重写；`quote / refer / verifications` 在卡片上显式可视化。此阶段 UI 上可完整驱动最小闭环 A（paused 进出）与 B（planning → executing → finished / cancelled）。

### Phase 4 · creator/owner + verify/insight/reviewed 完善

writer 严格校验 creator / owner；`act.owner` 允许非 creator；`verify.verifications[].target` 必须指向本 matter 内已存在的 `act`；`reviewed` 只能由 `insight` 触发且 matter 必须处于 finished / cancelled。前端补 owner 选择器、verify 判断编辑器（多选 act + 逐条 judgement/comment）、insight 表单可勾"同时推进到 reviewed"。

### Phase 5 · AI 面板改造 + AI-monitor 接入

`AIPane` 从 thread 回复助手转为 matter 上下文助手：`replyTarget → quoteTarget`，`referenceFiles → refer[]`，草稿生成支持 5 类文档。后端 AI context builder 切到压缩版 timeline。AI-monitor 正式通过 Phase 2 的事件流与 Phase 3 的 UI 槽位接入，不绕过 writer。

## 横跨matter的不变量

- 磁盘是单一事实源，SQLite 不镜像 matter 状态
- INDEX 写走原子 tmp+rename + `write_session` 锁
- committer = `team-pivot-web`，author = 用户 pinyin
- 状态机约束落在 writer 层，不仅 API 层
- AI 永远只产候选，落盘由程序决定（§六）

## AI 在产品模型里的参与面

产品文档已经明确 AI 在 matter 演进中的职责，AI-monitor 的埋点需同时覆盖下面 8 类：

1. 上下文理解（§二.3、§六）
2. 摘要候选（§六、§十二.6）
3. 关系判断候选 — quote / refer（§六）
4. 状态变化候选 — status_change（§六）
5. 验证评价候选 — verifications[].comment（§九.4 + §六）
6. 巡视 — verify → result 严谨性、长期无推进的 matter 巡检（§六、§九.4）
7. 关键节点提醒与推动（vision §四.2）
8. 闭环后的 insight 候选与复盘沉淀（§六、vision §四.2）

原则（§六）：AI 永远只产候选，schema 控制权属于程序。

## AI-monitor 参与节奏

| 阶段 | 阶段主要工作 | AI-monitor 参与形态 |
|---|---|---|
| P1 | 后端词表与 `matter_index` 读写模块 | 不参与 |
| P2 | 实现 Matter API，打通服务端最小闭环 | 埋点：事件流（create_matter / 追加 file / 追加 comment / status_change / apply_result）+ `matter_index` 只读快照 API |
| P3 | 前端 Matter 详情页切时间线视图 | UI 槽位占位：文件卡片"AI 摘要候选""AI 建议"；页面顶部"AI 观察 / 巡视""AI 下一步建议" |
| P4 | creator/owner、verify/insight/reviewed 闭环完善 | 候选字段 `ai_candidate_*` 元数据（summary / quote / refer / verifications.comment / status_change）+ reviewed 终态事件 |
| P5 | AI 面板改造，AI-monitor 正式接入 | 正式接入：消费事件流，输出上述 8 类职责，通过 P3 槽位渲染，不绕过 writer |

## 落点

- Phase 1–2 后端侧可独立推进，不影响现有 UI。
- Phase 3 是用户可见切换点，配合上线前迁移一起发布。
- Phase 4–5 分别闭合"责任字段 + verify/reviewed"与"AI 调度者"两条线。
- AI-monitor 自身的设计与实现独立立项，本路径只负责到埋点就位。

## 附录 · 代码改动清单

### 后端

**新增**

| 文件 | 作用 | 阶段 |
|---|---|---|
| `server/matter_status.py` | 6 态状态机 + 触发文件类型约束 | P1 |
| `server/doc_types.py` | 5 类文档枚举 + status×type 允许矩阵 | P1 |
| `server/matter_index.py` | timeline-first INDEX 读写（§八 示例） | P1 |
| `server/api/matters.py` | Matter API 路由 | P2 |
| `scripts/migrate_index_schema.py` | 上线前一次性迁移脚本 | 部署期 |

**修改**

| 文件 | 作用 | 阶段 |
|---|---|---|
| `server/app.py` | 挂载 matters 路由 | P2 |
| `server/publish.py` | 接入新 writer，逐步替换旧写路径 | P2 |
| `server/workspace.py` / `server/recovery.py` | 确认恢复顺序与新 writer 相容 | P2 |
| `server/threads.py` | 适配/最终下线 | P2–P5 |
| `server/ai/context.py` | 切投喂压缩版 timeline | P5 |
| `server/ai/prompts.py` | 改为 matter 语境系统提示词 | P5 |
| `server/api/ai.py` | chat 接口 context 语义调整 | P5 |

**下线**

- `server/status_machine.py`
- `server/index_files.py`
- 旧 publish / thread 写路径残留

### 前端

**新增**

| 文件 | 作用 | 阶段 |
|---|---|---|
| `web/src/pages/MatterDetailPane.tsx` | 替代 ThreadDetailPane | P3 |
| `web/src/components/TimelineView.tsx` | 时间轴 + 文件流主体 | P3 |
| `web/src/components/FileCard.tsx` | 文件卡片（type/summary/quote/refer/verifications/comments + 三入口） | P3 |
| `web/src/components/ResultConfirmDialog.tsx` | 收口确认弹窗 | P3 |
| `web/src/components/AISuggestionSlot.tsx` | AI 槽位（P3 占位 → P5 实渲染） | P3/P5 |
| `web/src/components/VerifyEditor.tsx` | verify 多选 + 逐条 judgement/comment | P4 |
| `web/src/components/OwnerPicker.tsx` | owner 选择器 | P4 |

**重写**

| 文件 | 作用 | 阶段 |
|---|---|---|
| `web/src/components/StatusControl.tsx` | 按 6 态配置化 | P3 |
| `web/src/components/StatusBadge.tsx` | 6 态徽章 | P3 |
| `web/src/components/AIPane.tsx` | quoteTarget + refer[]，支持 5 类草稿生成 | P5 |
| `web/src/pages/NewThread.tsx` → `NewMatter.tsx` | 创建 matter 首项（think/act） | P3 |

**修改**

| 文件 | 作用 | 阶段 |
|---|---|---|
| `web/src/App.tsx` | 路由与类型改名 | P3 |
| `web/src/pages/Dashboard.tsx` | Outlet context 改名 | P3 |
| `web/src/components/ThreadListPane.tsx` | 读 matter 字段 | P3 |
| `web/src/api.ts` | 新增 matter API fetch wrappers | P2/P3 |

### 数据迁移（上线前运行一次）

- 脚本：`scripts/migrate_index_schema.py`
- 操作：读工作区全部 `index/*.index.yaml`，按 §八 示例重写；所有存量 discussion 归为 `current_status: planning` 的 matter；文件项 `creator / owner / type / summary` 用最小合理默认
- 输出：单次 git commit，committer 维持 `team-pivot-web`
- 结果：系统只认新 shape，老 reader 与兼容分支全部下线

### 文档同步

- `AI-docs/pivot-interface.md` — 每阶段落接口时同步更新
- `AI-docs/pivot-memo.md` — Phase 5 完成后重写架构小节
