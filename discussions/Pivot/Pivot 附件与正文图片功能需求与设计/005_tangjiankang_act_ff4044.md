---
type: act
author: tangjiankang
created: '2026-05-08T10:38:12+08:00'
body_source: manual
index_state: indexed
---
## 附件与正文图片 — 落地实施 Plan

> 基于 Pivot matter「Pivot 附件与正文图片功能需求与设计」timeline 004（zhangbo 完整方案最新版，已合并 dengke 反馈）整理为可执行落地计划。
> 设计原文：[var/git/data-test/discussions/pivot/Pivot 附件与正文图片功能需求与设计/004_zhangbo_think_8b00a5.md](../../var/git/data-test/discussions/pivot/Pivot%20附件与正文图片功能需求与设计/004_zhangbo_think_8b00a5.md)

---

## 更新记录

- **2026-05-08 v1**：首版落地计划，按"后端基础 → publish 绑定/渲染 → 前端上传 → AI 集成"四阶段切分
  - 与设计稿 §8 / §12 的差异对齐：当前 [server/publish.py](../../server/publish.py) 已经拆为 `publish_matter_create / append / mention / annotation / owner_change / event`，没有 `publish_matter_comment`。bind 钩子挂载点改为：**`create / append / mention / annotation`** 这 4 个写 body 的函数；`owner_change.reason` 通常不放附件，v1 不挂；`event` 不写 md 文件，跳过
  - [web/src/components/matter/AnnotationDialog.tsx](../../web/src/components/matter/AnnotationDialog.tsx) 已存在，annotation body v1 同样**不开**上传 UI（与 mention 一致），但 bind 钩子保留以支持手粘 `attachment://`

---

## 总体策略

**四阶段交付，每阶段独立可验证。**

- Phase 1 落 SQLite 表 + 后端 IO + 3 条 API 路由：先把"附件能上传 / 下载 / 删除"做通，纯后端可测
- Phase 2 接 publish bind 钩子 + 渲染层：让发布后的附件能在 FileCard 正常显示（读路径打通）
- Phase 3 前端上传组件：用户能从编辑器上传 / 拖拽 / 粘贴（写路径打通）
- Phase 4 AI 集成：`_render_item` 透传 `attachments[]` 元数据 + 新增 `read_attachment` MCP 工具

设计哲学（设计稿 §10）：**后端不主动注入正文 / 原图**，只透传元数据。AI 自己看 `description` 决定要不要展开，多模态模型按需调 `read_attachment` 拿 base64。后端零兼容性维护，不区分 `supports_vision`。

**约束（v1 不做）**：评论附图、对象存储、OCR、版本管理、缩略图、病毒扫描。详见设计稿 §11。

---

## 改动清单速查

**新建**

- [server/attachments.py](../../server/attachments.py) — `AttachmentRepo` + 本地 IO
- [server/attachments_parsing.py](../../server/attachments_parsing.py) — pdf / docx / xlsx / pptx / txt / md → 纯文本提取
- [server/api/attachments.py](../../server/api/attachments.py) — 3 条路由
- [web/src/components/matter/AttachmentUploader.tsx](../../web/src/components/matter/AttachmentUploader.tsx) — 上传组件

**修改**

- [server/db.py](../../server/db.py) — `SCHEMA` 末尾加 `attachments` 表
- [server/app.py](../../server/app.py) — 注册路由
- [server/publish.py](../../server/publish.py) — `create / append / mention / annotation` 4 个函数末尾加 bind 钩子（`write_session` 退出之后）
- [server/api/matters.py](../../server/api/matters.py) `_render_item` — 注入 `attachments[]`
- [server/api/ai.py](../../server/api/ai.py) — 复用 `_render_item` 即可，几乎零改动
- [server/mcp/tools.py](../../server/mcp/tools.py) + [server/mcp/instructions.py](../../server/mcp/instructions.py) — 新增 `read_attachment(id)` 工具
- [web/src/api.ts](../../web/src/api.ts) — `Attachment` 类型 + `TimelineFileItem.attachments?` + helpers
- [web/src/components/matter/CreateFileDialog.tsx](../../web/src/components/matter/CreateFileDialog.tsx) — `BodyMarkdownEditor` 包 uploader
- [web/src/pages/NewMatter.tsx](../../web/src/pages/NewMatter.tsx) — Textarea 包 uploader
- [web/src/components/matter/FileCard.tsx](../../web/src/components/matter/FileCard.tsx) — `markdownComponents` 加 img/a，body 下加附件列表

**不动**

- [server/workspace.py](../../server/workspace.py)、[server/posts.py](../../server/posts.py)、[server/matter_index.py](../../server/matter_index.py)
- [web/src/pages/MatterDetailPane.tsx](../../web/src/pages/MatterDetailPane.tsx)
- 任何 Git / `write_session` 流程

---

## Phase 1：后端基础设施（SQLite + 本地 IO + 3 条 API）

> **目标**：附件能上传到磁盘、能鉴权下载、草稿态能删；纯后端可验。
> **约束**：不接 publish.py、不改 `_render_item`、前端零感知。

### Task 1.1：SQLite schema

**File:** [server/db.py](../../server/db.py) `SCHEMA`

- [ ] 末尾追加 `attachments` 表，字段对齐设计稿 §5.1（`id / matter_id / file_path / filename / content_type / size / storage_key / uploaded_by / usage / description / created_at`）
- [ ] `file_path` 草稿态可空、发布后绑定
- [ ] 加 `(matter_id, filename)` UNIQUE 索引兜底设计稿 §4 同名约束 → 应用层捕 `IntegrityError` 转 409
- [ ] 老库下次启动 `db.init` 自动建表，无迁移脚本

### Task 1.2：AttachmentRepo + 本地 IO

**File:** [server/attachments.py](../../server/attachments.py)

实现路线：

- 数据类 + Repo（`create / get / list_by_matter / list_by_file / bind_to_file / delete_draft`）
- 本地 IO 抽出 `write_blob / open_blob / unlink_blob`，`storage_key` 作为抽象层（设计稿 §6 预留对象存储切换）
- 根目录：`workspace.data_dir / "attachments" / <matter_id>`，参考 [server/config.py:46](../../server/config.py#L46)；天然在 Git 仓库目录之外，无需改 `.gitignore`
- description 兜底：`create()` 内若空字符串则用去扩展名后的 filename（设计稿 §10.3）

### Task 1.3：白名单 + 限制校验

**File:** [server/api/attachments.py](../../server/api/attachments.py)

- 白名单：`content_type` + 扩展名双重校验（设计稿 §4 三类）
- 限制阈值集中到 [server/settings.py](../../server/settings.py)：单文件 2 MB、单帖 3 个、`ATTACHMENT_TEXT_TOKEN_LIMIT=8000`
- HTTP 状态码与设计稿 §4 对齐：单文件超限 **413**、数量超限 **422**、同名 **409**、白名单拒绝 **415**

### Task 1.4：3 条 API 路由

**File:** [server/api/attachments.py](../../server/api/attachments.py)、[server/app.py](../../server/app.py) 注册

- `POST /api/matters/{matter_id}/attachments`：multipart 接收 `file / usage / description?` → 返回 `{id, ..., markdown_ref}`（图片走 `![]()`、文档走 `[]()`，前端拿到直接插入光标）
- `GET /api/attachments/{id}[?preview=1]`：matter ACL 重校验 → `StreamingResponse`，`preview=1` → `Content-Disposition: inline`
- `DELETE /api/attachments/{id}`：`file_path IS NULL` 且 `uploaded_by == 当前用户` → 通过；已绑定一律 403

### 验收清单（Phase 1）

- [ ] httpx 直接调 POST 上传 PDF → 拿到 id；GET 下载内容一致
- [ ] 上传同名第二份 → 409；超 2MB → 413；非白名单类型 → 415
- [ ] 草稿态 DELETE 成功；模拟绑定后再 DELETE → 403
- [ ] `<DATA_DIR>/attachments/<matter_id>/` 出现文件，`git status` 不感知
- [ ] 跨 matter 鉴权：用户 A 不能 GET 用户 B 私有 matter 的附件

---

## Phase 2：publish bind 钩子 + 渲染层

> **目标**：DB 里有附件 + body 里写 `attachment://att_xxx`，发布后 `file_path` 自动绑定且 FileCard 渲染正常。
> **依赖**：Phase 1。

### Task 2.1：publish bind 钩子

**File:** [server/publish.py](../../server/publish.py)

设计稿 §8 要求 bind 在 `write_session` **退出之后**执行。当前 publish.py 实际函数清单：

| 函数 | 是否写 body | 是否挂 bind |
|---|---|---|
| `publish_matter_create` | ✓ | ✓ |
| `publish_matter_append` | ✓ | ✓ |
| `publish_matter_mention` | ✓ | ✓（评论 v1 不开 UI，但允许手粘） |
| `publish_matter_annotation` | ✓ | ✓ |
| `publish_matter_owner_change` | ✓ reason | ✗ v1 不挂 |
| `publish_matter_event` | ✗ | ✗ |

实现路线：

- 抽公共函数 `_bind_attachments_in_body(repo, body, file_rel)`：regex 抽 `attachment://(att_\w+)` → `bind_to_file()`
- 仅 `WHERE file_path IS NULL` 时绑定（重复发布 / restore 不影响已绑定附件）
- id 不存在 / 不属于当前 matter → 静默丢弃，不阻断发布

### Task 2.2：`_render_item` 注入 attachments

**File:** [server/api/matters.py](../../server/api/matters.py) `_render_item` (L1342)

- 渲染 timeline item 时，按 `item.file_rel` 调 `list_by_file()`，输出元数据字段（设计稿 §10.1，不暴露 `storage_key`、不解析正文、不发原图）
- N+1 风险：单 matter timeline 通常 < 50 条，v1 每条一查；后期监控再批量

### Task 2.3：FileCard 渲染

**File:** [web/src/components/matter/FileCard.tsx](../../web/src/components/matter/FileCard.tsx)

- `markdownComponents.img` 拦截 `attachment://` → 替换为预览 URL，CSS 限宽，点击放大
- `markdownComponents.a` 拦截 `attachment://` href → 替换为下载 URL
- body 下方新增附件区，按 `item.attachments` 平铺 chip（含 inline_image，设计稿 §3 统一列表）
- 鉴权失败 → `<img onError>` 兜底占位

### Task 2.4：API types

**File:** [web/src/api.ts](../../web/src/api.ts)

- `Attachment` 类型 + `TimelineFileItem.attachments?`
- `attachmentUrl(id, preview?: boolean)` helper

### 验收清单（Phase 2）

- [ ] 直接 SQL 插一条附件 + curl POST 一条带 `![x](attachment://att_xxx)` 的 think → 浏览器 FileCard 显示图片
- [ ] 同一附件在正文 + 附件区两处显示
- [ ] mention / annotation body 含 `attachment://` → bind 后渲染正常
- [ ] body 引用不存在的 id → 不阻断发布、渲染兜底为占位

---

## Phase 3：前端上传组件

> **目标**：用户能从 CreateFileDialog / Append 编辑器走 UI 上传，含粘贴、拖拽、删除。
> **依赖**：Phase 1（API）+ Phase 2（types）。

### Task 3.1：AttachmentUploader 组件

**File:** [web/src/components/matter/AttachmentUploader.tsx](../../web/src/components/matter/AttachmentUploader.tsx)

实现路线：

- Props：`matterId / pendingAttachments / onChange / onInsertMarkdown`
- 三种上传方式：点击 → `<input type="file">`；拖拽 → `onDrop`；**粘贴 → `onPaste`** 监听 Textarea 父元素，捕获 `clipboardData.files`，图片自动 POST + 在光标处插入 `markdown_ref`
- description 输入：每个 chip 上方 placeholder 引导（"如：客户验收签字截图 / 方案 v3 PDF"），留空由后端文件名兜底
- 错误提示按状态码分支：413 / 422 / 409 / 415 各一句中文
- 草稿态删除：调 DELETE + 同步从 body 移除指向该 id 的所有 `attachment://` 行（设计稿 §3）

### Task 3.2：CreateFileDialog 集成

**File:** [web/src/components/matter/CreateFileDialog.tsx](../../web/src/components/matter/CreateFileDialog.tsx)

- `BodyMarkdownEditor` 外包一层 div，容纳 Textarea + `<AttachmentUploader>`
- 提交时不需要传附件 id 列表，bind 钩子从 body 自动抽
- dialog 取消 → 调 DELETE 清未绑定附件（避免孤儿，可选优化）

### Task 3.3：NewMatter 集成

**File:** [web/src/pages/NewMatter.tsx](../../web/src/pages/NewMatter.tsx)

- 难点：创建前 `matter_id` 不存在，无法上传
- **v1 决策**：NewMatter 不开附件入口，文档化「请创建后通过 append 添加」；不引入 `pre_create_token` 复杂度

### 验收清单（Phase 3）

- [ ] 在 think dialog 上传图片 → 自动插入 `![](attachment://att_xxx)` + chip 出现
- [ ] 粘贴系统截图 → 自动上传 + 插入
- [ ] 提交后 → FileCard 渲染图片正常
- [ ] 触发 3 类边界提示：3 个附件后第 4 个失败、上传同名、超 2MB
- [ ] 草稿态删 chip → body 里的 `attachment://` 行同步消失

---

## Phase 4：AI 集成（元数据透传 + read_attachment 工具）

> **目标**：AI 在 think AI / list_matter 等场景看到附件元数据，按需调 `read_attachment` 拿正文 / 图像。
> **依赖**：Phase 2（`_render_item.attachments[]` 已就位）。

### Task 4.1：AI prompt 透传

**File:** [server/api/ai.py](../../server/api/ai.py)

- 复核 AI 路由是否复用 `_render_item`：是 → 零改动；否 → 加同样的 `list_by_file` 投影
- 不区分 vision / 非 vision 模型；不解析正文；不内联 base64

### Task 4.2：read_attachment MCP 工具

**Files:** [server/mcp/tools.py](../../server/mcp/tools.py)、[server/mcp/instructions.py](../../server/mcp/instructions.py)

实现路线：

- 入参：`id: str`；鉴权：复用 MCP context user → matter ACL
- 文档类（pdf / docx / xlsx / pptx / txt / md）：走 `attachments_parsing.extract_text` → 按 `ATTACHMENT_TEXT_TOKEN_LIMIT`（默认 8000）截断 → 返回 `{kind: 'text', text, total_chars, truncated}`
  - token 估算用字符数近似（中文 ~1.5 字符/token、英文 ~4），简化为 `max_chars = limit * 2`
- 图片类：返回 `{kind: 'image', content_type, base64}`，多模态模型自决；纯文本模型按 instructions 自觉跳过
- system instructions 追加段（设计稿 §10.2）：「附件元数据已附在每条 timeline 上。根据 description 判断相关性、根据自己能力决定是否调 `read_attachment(id)`。」

### Task 4.3：附件解析

**File:** [server/attachments_parsing.py](../../server/attachments_parsing.py)

- 各 MIME → 解析器映射：`pypdf` / `python-docx` / `openpyxl` / `python-pptx` / 直接读
- 失败 → 返回空字符串，工具层透传 `total_chars=0`，AI 自动降级到只看 description
- 依赖 lazy import 避免冷启动膨胀

### 验收清单（Phase 4）

- [ ] 真实跑一条 think AI，timeline 含 1 份 PDF + 1 张截图
- [ ] AI 输出能引用 PDF 内容（说明工具被调用并解析成功）
- [ ] 多模态模型调 `read_attachment` 拿 base64 → 输出能描述图片内容
- [ ] 纯文本模型只看 description 回答，不主动调 image 类附件
- [ ] 截断阈值生效：临时调 `ATTACHMENT_TEXT_TOKEN_LIMIT=100`，长文档返回 `truncated=true`

---

## 风险 & 决策项

| # | 议题 | 风险 | 决策建议 |
|---|---|---|---|
| 1 | mention / annotation v1 不开上传 UI | 用户必须先在别处上传得到 id 再手粘，体验差 | bind 钩子保留即可，UI 推到 v2 |
| 2 | NewMatter 阶段 `matter_id` 未生成 | 创建对话框无法上传 | v1 走"先创建再 append"；不引入 `pre_create_token` |
| 3 | description 留空 + 文件名无语义（如 `IMG_20260101.jpg`） | AI 难判断相关性 | 接受 — 设计稿 §10.3 明确兜底；UI placeholder 引导用户填 |
| 4 | 8000 token 截断阈值 | 长合同 / 大份纪要被截到前 1/3 | v1 不做分页（设计稿 §10.2 已确认）；实际不够再加 `offset/limit` |
| 5 | invalidate 后附件访问 | 设计稿 §3 要求"返回所属帖子已作废" | GET 路由加 timeline item 状态校验，invalidated → 410 或 200 + 错误 JSON；前端按状态显占位 |
| 6 | 同 matter 同名约束 | 多版本工作流不便（合同 v1 / v2） | 设计稿明确建议改文件名（`方案-v2.pdf`）；UNIQUE 强制 + 冲突时 UI 给清晰提示 |
| 7 | `_render_item` N+1 查询 | 长 timeline matter 性能下降 | v1 简化每条一查；P95 > 200ms 后改批量 |
| 8 | 附件目录磁盘占用 | 删帖不删附件，长期累积 | v1 不做 GC；运维直接观测 `<DATA_DIR>/attachments/`，后期看需求加清理工具 |

---

## 落地建议

1. **Phase 1 先行**：纯后端，可独立完成并合并；阻塞性最低
2. **Phase 2 + 3 串行**：Phase 2 打通读路径后立刻接 Phase 3 写路径
3. **Phase 4 可并行**：元数据透传 (4.1) 在 Phase 2 完成后即可启动，与前端 Phase 3 并行
4. **首次手测**：Phase 3 完成后找一个真实 matter 跑通完整流程：粘贴截图 → 提交 think → 浏览器看图 → think AI 引用图片描述 → 验证设计稿 §2 全部 4 个使用场景
5. **annotation v1 容忍度**：bind 钩子已就位，[AnnotationDialog.tsx](../../web/src/components/matter/AnnotationDialog.tsx) 不接 uploader（与 mention 一致）；v2 评估是否开放
