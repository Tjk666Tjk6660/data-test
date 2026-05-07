---
type: think
author: zhangbo
created: '2026-05-06T22:33:43+08:00'
index_state: indexed
---
# Pivot 附件与正文图片功能需求与设计

## 1. 背景与目标

Matter 协作中需要上传材料（PDF / 截图 / 表格 / 设计图等），并在正文嵌入图片。新增两类能力：

- **附件**：think / act / verify / result / insight 帖子可附文件，列表展示，可下载
- **正文图片**：编辑器内直接插入图片，与文字一起渲染

不影响无附件场景的现有流程。评论 v1 不支持上传。

## 2. 使用场景

| 场景 | 用法 |
|---|---|
| 上传方案 / 交付物 | 在 think / act / result / insight 帖子下挂 PDF / Word / Excel |
| 嵌入截图说明 | 编辑器粘贴 / 拖入图片 → 自动插入 `attachment://` 引用 |
| 验收上传证明 | verify 时附验收截图、客户确认文件 |
| 复盘材料 | insight 时附复盘 deck、看板截图；推进到 reviewed 也走 insight |

## 3. 编辑与权限

**上传方式**：点击 / 拖拽 / 粘贴。选了文件就立刻独立 POST，不等帖子提交。

**草稿态自由删改**：帖子未发布前（`file_path` 未绑定），uploader 在编辑器内可随意删 / 换附件，无额外授权。

**附件与正文图片统一管理**：编辑器和渲染页都把"普通附件 + 正文图片"按一个列表展示，列出标题 / 文件名。在编辑器删除任意一项（含正文图片）→ 同步把 body 里所有指向该 `att_id` 的 `attachment://` 引用删掉，避免正文里残留死链。反方向：草稿态在编辑器手动删除 body 里的 `attachment://` 那一行，**不会**删除附件本身，附件区还能看到。

**发布后不可删除**：附件随帖子一起冻结，任何人（包括 uploader / Matter owner / admin）都不能删除单个附件。需要更正走 invalidate 流程整条作废帖子，附件文件保留、访问时返回"所属帖子已作废"；restore 后恢复访问。

**鉴权**：附件继承 matter 的角色 + 用户权限。每次访问 → `attachment_id` 反查 `matter_id` → 调 matter ACL 模块判定。下载 / 预览短链由 cookie session + 每次重校验实现，不签外部 URL。

## 4. 文件类型与限制

| 类型 | 白名单 |
|---|---|
| 图片 | png、jpg、jpeg、webp、gif |
| 文档 | pdf、doc、docx、xls、xlsx、ppt、pptx |
| 文本 | txt |

| 限制 | 默认值（可配置） |
|---|---|
| 单文件大小 | 2 MB，超出后上传失败提示"文件过大" |
| 单帖附件数 | 3 个（`usage=attachment` 和 `usage=inline_image` 合并计数），超出后上传失败提示"附件数已达上限" |
| 同帖不可同名 | 同一帖子下不允许出现两个同名附件（按 filename 全字符串匹配），冲突时上传失败提示"已存在同名附件，请先删除或改名"。需要替换走"删除原文件 → 再次上传"（仅草稿态可行，发布后不可删） |

> 后端返回的 HTTP 状态码：单文件超限 = 413（Payload Too Large），数量超限 = 422（Unprocessable Entity），同名冲突 = 409（Conflict）。前端按状态码区分失败提示。

后端在 `server/api/attachments.py` 上传路由用 `content_type` + 扩展名双重校验，写盘前生效。

## 5. 数据结构

### 5.1 SQLite `attachments` 表

加在 `server/db.py` 的 `SCHEMA` 末尾，老库下次启动自动建：

```json
{
  "id": "att_001",
  "matter_id": "matter_001",
  "file_path": null,
  "filename": "验收截图.png",
  "content_type": "image/png",
  "size": 204800,
  "storage_key": "matters/matter_001/att_001.png",
  "uploaded_by": "zhangsan",
  "usage": "attachment | inline_image",
  "alt_text": "用户填写的图片说明",
  "description": null,
  "created_at": "2026-04-29T18:30:00+08:00"
}
```

字段说明：`file_path` 草稿态为 NULL、发布后绑定 timeline 文件路径；`alt_text` 可选，AI fallback 用（§10）；`description` 预留 vision pipeline 自动生成（§10）。

### 5.2 正文引用语法

```markdown
![验收截图](attachment://att_001)
```

正文按字面存 `attachment://`，不转换。前端 `react-markdown` 的 `img` 自定义组件拦截前缀 → 替换为 `/api/attachments/{id}?preview=1`。

## 6. 存储

附件落 `<DATA_DIR>/attachments/<matter_id>/`，**天然不在 Git 仓库下**（仓库在 `<DATA_DIR>/git/<repo>`），无需改 `.gitignore`。`storage_key` 作为抽象层，后期切对象存储只改 IO。

## 7. API

新增三条路由集中在 `server/api/attachments.py`：

```
POST   /api/matters/{matter_id}/attachments   # multipart → 返回 {id, markdown_ref, ...}
GET    /api/attachments/{id}[?preview=1]      # 鉴权 → 流文件
DELETE /api/attachments/{id}                  # 仅草稿态可删（file_path IS NULL + 校验 uploader）；已绑定一律 403
```

## 8. write_session 与 publish bind

附件 IO **不进 `write_session`**（避免 Git 锁阻塞）。绑定时机：`server/publish.py` 的 3 个发布函数（`publish_matter_create` / `publish_matter_append` / `publish_matter_comment`）在 `write_session` **退出之后**扫一遍刚提交的 body / comment body 中的 `attachment://att_xxx` 引用 → 调 `repo.bind_to_file(att_ids, file_rel)`。

> `publish_matter_event`（invalidate / restore）不写 md 文件、事件直接进 matter index，没有 body 可扫，不需要 bind 钩子。

评论 v1 不开上传 UI，但 `publish_matter_comment` 的 bind 钩子保留：用户可能手粘 `attachment://` 到评论 body，bind 后附件 `file_path` 才能正确指向被评论的 timeline 文件，让 `_render_item` 在渲染 / AI 读取时挂上对应附件。

## 9. 前端改造

**编辑器**（`CreateFileDialog.tsx` 的 `BodyMarkdownEditor`、`NewMatter.tsx` 的 Textarea）：包一层 div 套 Textarea + 新组件 `<AttachmentUploader>`；监听 `onPaste` 自动上传图片并在光标处插入 `markdown_ref`；下方渲染附件 chip 列表。v1 保持 Textarea，不换富文本。

**渲染**（`FileCard.tsx`）：
- `markdownComponents.img` 拦截 `attachment://` → 预览 URL，点击放大
- `markdownComponents.a` 处理 `attachment://` href → 下载
- body 下方新增附件区，按 §3 的统一列表展示（含 inline_image），点击图片项可跳到正文锚点或开放大预览

**API（`web/src/api.ts`）**：`TimelineFileItem.attachments?: Attachment[]` + `uploadAttachment / deleteAttachment / attachmentUrl`。

**v1 不做服务端缩略图**：CSS `max-width:100%` 限宽，点击弹窗看原图。

## 10. AI 集成

### 10.1 上下文注入策略

**默认行为**：附件 + 正文图片**全部注入** LLM 上下文，让 AI 基于完整信息回答，不让用户在 AI 回复后才发现"它没看附件"。

| 内容 | 注入形式 |
|---|---|
| 元数据（filename / type / size / uploader / alt_text） | 总是挂在 timeline item 的 `attachments[]` 上 |
| 正文图片（`usage=inline_image`） | 按目标模型 vision 能力走多模态消息块或文本 fallback（§10.2） |
| 普通附件文档（pdf / doc / docx / xls / xlsx / ppt / pptx / txt） | 后端解析为文本，与 timeline 一起进上下文 |
| 普通附件图片（`usage=attachment` 但 content-type 是 image） | 同正文图片，走 §10.2 的 vision fallback |

注入点：`server/api/matters.py` 的 `_render_item` 在每条 timeline item 上挂 `attachments[]`，字段 `id / filename / content_type / size / usage / alt_text / parsed_text`（**不**暴露 `storage_key`）。`server/api/ai.py` 在拼 prompt 时把 `parsed_text` 与 body 一起送给模型，图片走 §10.2。

实施时面临两个约束：

**约束 1 · 上下文容量**：附件正文进 LLM 会占 token，纯文本类附件（如 2 MB 的 txt）解析后可达数十万 token，足以撑爆任何主流模型上下文。处理：单附件解析后超阈值（默认 4000 token，可配置）只注入 **前 N 字符 + 总长度**；AI 想看全文调 `read_attachment(id)` 工具按需获取。

**约束 2 · 多模型 vision 兼容**：见 §10.2。

### 10.2 图片要怎么发给大模型

**问题**：大部分大模型只能读文字，不能直接看图。Pivot 用户还能在 `SettingsExternalAI` 自己换模型，所以系统不能假设 AI 一定看得懂图。

**做法**：给每张图准备三种"替身"，能用图就用图，不能就用文字描述：

| 替身 | 哪来的 | 谁都能读 |
|---|---|---|
| ① 用户写的图片说明 | 上传图片时让用户填一句"这张图说什么"（alt 文本） | ✓ 文字，所有模型都行 |
| ② 系统自动生成的描述 | 上传时让一个支持图片的模型先看一遍，把它说的话存下来（v1 留字段不实现） | ✓ 文字，所有模型都行 |
| ③ 原图本身 | 直接把图片发给模型 | ✗ 只有支持图片的模型能读（如 Claude / GPT-4o） |

**怎么选**：当前对话用的模型支不支持图片，由 `server/api/ai.py` 在拼上下文前判断：

- 支持图片的模型 → 直接发原图，再把①或②当配文一起发，效果最好
- 不支持图片的模型 → 退而求其次，发自动描述②；没有就用用户写的①；都没有就只告诉模型"这里有一张图"

**capability 判断怎么做**：现有 `ai.py` 是单 OpenAI-compat 透传通道，没有 vision capability 元数据。新增方式：
- `SettingsExternalAI` 加一个用户自报开关 `supports_vision`（谁配的模型谁勾，最稳）
- 配合一个内置 known-model 名单（`gpt-4o*` / `claude-3*` / `gemini-*-vision` 等）按 model 字符串前缀给默认值，用户可覆盖

**多模态消息格式**：图片走 OpenAI 多模态约定，messages.content = `[{"type":"text",...},{"type":"image_url",...}]`。`server/ai/client.py` 的 `stream_chat` 是透明转发，**不需要改**；但用户配的 `base_url` 必须接到支持多模态的 OpenAI-compat 服务，否则模型勾了 vision 也发不过去——这条作为已知前置依赖，不在代码范围内解决。

## 11. 开放问题（v1 不做）

| 问题 | v1 决策 |
|---|---|
| 对象存储 | 本地优先，`storage_key` 抽象层预留切换 |
| OCR / Office 在线预览 | 不做 |
| 附件版本管理 | 不做（同名重传产生新 id） |
| 病毒扫描 | 不做 |
| 评论附图 | 不做（后续扩展 `comments[].attachments`） |
| 服务端缩略图 | 不做（CSS 限宽 + 点击原图） |
| 自动 description vision pipeline | 字段预留，v1 不实现 |

## 12. 改动清单速查

**新建**

- `server/attachments.py` —— `AttachmentRepo` + 本地 IO
- `server/attachments_parsing.py` —— pdf / doc / xls / ppt 文本提取
- `server/api/attachments.py` —— 三条路由
- `web/src/components/matter/AttachmentUploader.tsx` —— 上传组件

**修改**

- `server/db.py` —— `SCHEMA` 加 `attachments` 表
- `server/app.py` —— 接路由
- `server/publish.py` —— 3 个 publish 函数末尾加 bind 钩子（`create / append / comment`，event 无 body 不参与）
- `server/api/matters.py` —— `_render_item` 注入 attachments
- `server/api/ai.py` —— 模型 vision 能力判断 + 图片三种替身 fallback；附件文本超阈值截断
- `server/mcp/instructions.py` + `tools.py` —— 新增 `read_attachment` 工具（取超阈值截断后的全文）
- `web/src/api.ts` —— 类型 + helpers
- `web/src/components/matter/CreateFileDialog.tsx` —— BodyMarkdownEditor 包 uploader
- `web/src/pages/NewMatter.tsx` —— Textarea 包 uploader
- `web/src/components/matter/FileCard.tsx` —— markdownComponents 加 img/a，body 下加附件列表

**不动**

- `server/workspace.py`、`server/posts.py`、`server/matter_index.py`
- `web/src/pages/MatterDetailPane.tsx`
- 任何 Git / `write_session` 流程
