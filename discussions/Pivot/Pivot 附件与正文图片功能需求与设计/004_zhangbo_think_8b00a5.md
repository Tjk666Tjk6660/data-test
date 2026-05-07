---
type: think
author: zhangbo
created: '2026-05-07T09:48:27+08:00'
index_state: indexed
---
这条贴上**当前最新完整方案**，覆盖前面所有讨论与修订。后续以这条为准。

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
| 文本 | txt、md |

| 限制 | 默认值（可配置） |
|---|---|
| 单文件大小 | 2 MB，超出后上传失败提示"文件过大" |
| 单帖附件数 | 3 个（`usage=attachment` 和 `usage=inline_image` 合并计数），超出后上传失败提示"附件数已达上限" |
| 同 matter 不可同名 | 同一 matter 下（跨所有帖子）不允许出现两个同名附件（按 filename 全字符串匹配），冲突时上传失败提示"已存在同名附件，请先删除或改名"。从根上消除"附件版本"的歧义。需要替换走"删除原文件 → 再次上传"（仅草稿态可行，发布后不可删） |

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
  "description": "用户填写的内容说明；留空则后端用去扩展名后的 filename 兜底；AI 判断要不要展开读取此附件的唯一依据（§10）",
  "created_at": "2026-04-29T18:30:00+08:00"
}
```

`file_path` 草稿态为 NULL、发布后绑定 timeline 文件路径。

### 5.2 正文引用语法

```
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

设计哲学：**附件首要价值是人与人协作，AI 是辅助阅读者，不应强制介入**。后端不主动注入正文 / 原图，只透传元数据；AI 自己看元数据决定要不要展开。

### 10.1 后端：只透传元数据

`server/api/matters.py` 的 `_render_item` 在每条 timeline item 上挂 `attachments[]`：

```
{
  "id": "att_001",
  "filename": "验收截图.png",
  "content_type": "image/png",
  "size": 204800,
  "usage": "attachment | inline_image",
  "description": "客户对最终方案的签字截图，确认验收"
}
```

不暴露 `storage_key`、不解析正文、不发原图。`server/api/ai.py` 在拼 prompt 时把这个数组直接随 timeline 一起给 AI 即可，几乎零额外逻辑。

### 10.2 AI 端：用工具按需读取

新增 MCP 工具 `read_attachment(id)`：

- **文档类**（pdf / word / excel / ppt / txt / md）→ 后端解析为文本返回；超长按阈值截断，只返前 N 字符 + 总长度
- **图片类** → 后端读文件返回 base64 + content_type，AI 自己决定怎么用（多模态模型把它塞进下一轮 message，纯文本模型直接忽略）

**截断阈值默认 8000 token，可配置**。由来：

| 维度 | 数据 |
|---|---|
| §4 单文件大小上限 | 2 MB |
| Office 文件解析后纯文本量级 | 通常 5K–30K 字（PDF / Word / PPT 体积主要是排版和图） |
| 8000 token 在主流分词器下 ≈ 中文字数 | Claude / GPT 系 ~5K–8K 字；第三方 OpenAI-compat ~3K–5K 字 |
| 覆盖效果 | 大多数 Pivot 实际附件（方案、纪要、验收说明、合同摘要、短文档）能一次拿全；少数长合同 / 大份会议纪要会被截断 |

设 4000 token 会偏紧，多数 Office 附件被截到前 1/3 就丢；设 16000 token 又过度膨胀消耗 AI 输出配额。8000 是覆盖率与上下文消耗的折中。v1 不做分页参数（`offset/limit`），靠这个阈值一次返回足够内容；后期实际跑下来不够再加。

System prompt 里告诉 AI：附件元数据已经给你了，根据 `description` 判断相关性、根据自己的能力决定是否调 `read_attachment`。这样：

- **纯文本模型** 看到 `content_type: image/*` 通常会跳过，靠 description 回答
- **多模态模型** 觉得关键时主动调取原图
- **后端零兼容性维护**：不需要 `supports_vision` 开关、不需要 known-model 白名单、不需要在 prompt 拼装时区分模型类型

### 10.3 description 默认值与引导

`description` 是这套设计的关键依赖——AI 判断要不要展开附件的唯一信号。处理：

- **不强制必填**：用户嫌烦可以直接上传
- **留空时后端用 `filename` 作为默认 description**（取去除扩展名的部分，比如 `客户验收签字.png` → description = "客户验收签字"）
- 前端 UI 仍然给一句 placeholder 引导用户主动填一句更有意义的说明（"如：客户验收签字截图 / 方案 v3 PDF / 错误日志"），文件名足够自描述时（"项目 PRD.pdf"）就让默认值兜底

这样既保住了 AI 判断附件相关性的最低信号（文件名通常已经携带语义），又不打断用户的上传流程。

## 11. 开放问题（v1 不做）

| 问题 | v1 决策 |
|---|---|
| 对象存储 | 本地优先，`storage_key` 抽象层预留切换 |
| OCR / Office 在线预览 | 不做 |
| 附件版本管理 | 不做。靠 §4 的"同 matter 不可同名"约束从根上消除版本歧义——同一份文件就只有一个 `id`，要新版本就改文件名（如 `方案-v2.pdf`） |
| 病毒扫描 | 不做 |
| 评论附图 | 不做（后续扩展 `comments[].attachments`） |
| 服务端缩略图 | 不做（CSS 限宽 + 点击原图） |
| 后端主动注入附件正文 / 原图 | 不做（哲学：见 §10，由 AI 调工具自决） |

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
- `server/api/ai.py` —— `_render_item` 的 `attachments[]` 元数据随 timeline 透传给 AI（几乎零改动，不做 vision 判断、不做正文注入）
- `server/mcp/instructions.py` + `tools.py` —— 新增 `read_attachment(id)` 工具：文档类返回截断文本，图片类返回 base64 + content_type，由 AI 自决
- `web/src/api.ts` —— 类型 + helpers
- `web/src/components/matter/CreateFileDialog.tsx` —— BodyMarkdownEditor 包 uploader
- `web/src/pages/NewMatter.tsx` —— Textarea 包 uploader
- `web/src/components/matter/FileCard.tsx` —— markdownComponents 加 img/a，body 下加附件列表

**不动**

- `server/workspace.py`、`server/posts.py`、`server/matter_index.py`
- `web/src/pages/MatterDetailPane.tsx`
- 任何 Git / `write_session` 流程
