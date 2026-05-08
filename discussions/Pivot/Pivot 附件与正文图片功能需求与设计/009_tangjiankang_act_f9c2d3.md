---
type: act
author: tangjiankang
created: '2026-05-08T11:28:26+08:00'
body_source: manual
index_state: indexed
---
## 附件与正文图片 — 落地实施 Plan

> 基于 Pivot matter「Pivot 附件与正文图片功能需求与设计」timeline 004（zhangbo 完整方案最新版，已合并 dengke 反馈）
---

## 架构设计

### 模块边界

```
┌────────────────────────────────────────────────────────────┐
│ 前端                                                        │
│  AttachmentUploader  ──┐                                   │
│  CreateFileDialog      ├─→  /api/attachments               │
│  FileCard 渲染         ─┘                                   │
└────────────────────────────────────────────────────────────┘
                              │
┌────────────────────────────────────────────────────────────┐
│ HTTP 层  api/attachments.py                                 │
│   上传 / 下载 / 草稿删除 + 鉴权委托给 matter ACL            │
└────────────────────────────────────────────────────────────┘
        │                                    │
        ▼                                    ▼
┌──────────────────┐              ┌──────────────────────────┐
│ AttachmentRepo   │              │ publish.py bind 钩子      │
│  (DB 元数据)     │ ◀──绑定──── │  (write_session 退出后)  │
│  + Storage IO    │              └──────────────────────────┘
│  (本地文件)      │
└──────────────────┘
        ▲
        │  list_by_file
        │
┌────────────────────────────────────────────────────────────┐
│ 渲染 / AI 层                                                │
│   _render_item   ──→ timeline item 上挂 attachments[]      │
│   read_attachment MCP tool ──→ AI 按需展开                  │
└────────────────────────────────────────────────────────────┘
```

### 关键边界约定

- **附件 IO 与 Git write_session 解耦**：附件落本地磁盘，绑定钩子在 `write_session` 退出**之后**执行，避免 Git 锁阻塞（设计稿 §8）
- **存储抽象**：`storage_key` 作为 IO 抽象层，未来切对象存储只改 IO 实现，不改上层（设计稿 §6）
- **鉴权一致性**：附件没有自己的权限模型，每次访问通过 `matter_id` 反查走 matter ACL，与 timeline item 同源
- **元数据透传不解析正文**：`_render_item` 只挂元数据；正文 / 原图通过 AI 工具按需获取，后端不主动膨胀 prompt

### 数据流：上传 → 发布 → 渲染

1. **上传期**：前端拿到 `matter_id` 后即可 POST 上传，附件入 DB 但 `file_path` 为空（草稿态）
2. **发布期**：publish 函数完成 `write_session` 后，从 body 抽 `attachment://` 引用 → 调 `bind_to_file` 把附件锚到具体 timeline 文件
3. **渲染期**：`_render_item` 按 `file_path` 反查附件元数据挂上去，前端按 markdown + 附件区双重展示
4. **AI 读取期**：AI 看到元数据后自决是否调 `read_attachment`，文档类返回截断文本、图片类返回 base64

---

## Phase 1：后端基础设施

> **交付**：附件作为独立资源能上传、鉴权访问、草稿态删除；不接 publish.py、不改 `_render_item`、前端零感知。

涉及文件：

- [server/db.py](../../server/db.py) — 新增 `attachments` 表（字段对照设计稿 §5.1）
- [server/attachments.py](../../server/attachments.py) — 元数据 Repo + 本地存储 IO；落盘根目录在 Git 仓库之外
- [server/api/attachments.py](../../server/api/attachments.py) — HTTP 路由（鉴权委托 matter ACL）：

  | 方法 | 路径 | 职责 |
  |---|---|---|
  | POST | `/api/matters/{matter_id}/attachments` | multipart 上传，校验白名单 + 大小 + 数量 + 同名；返回元数据 + `markdown_ref` 让前端直接插入光标 |
  | GET | `/api/attachments/{id}[?preview=1]` | 流式下载或预览（`Content-Disposition` 切换 inline / attachment） |
  | DELETE | `/api/attachments/{id}` | 仅草稿态可删（`file_path IS NULL` + uploader 校验），已绑定一律 403 |

  HTTP 状态码语义对齐设计稿 §4：超大 413、超数量 422、同名 409、白名单外 415。
- [server/settings.py](../../server/settings.py) — 集中阈值（单文件大小、单帖数量、token 截断），数值取自设计稿 §4 / §10.2
- [server/app.py](../../server/app.py) — 注册路由

---

## Phase 2：发布期绑定 + 渲染

> **交付**：附件与 timeline item 建立关联，前端读路径打通。

涉及文件：

- [server/publish.py](../../server/publish.py) — 在 `write_session` 退出之后，从 body 抽 `attachment://` 引用并绑定。挂载点决策：

  | 函数 | 写 body | 挂 bind |
  |---|---|---|
  | `publish_matter_create` | ✓ | ✓ |
  | `publish_matter_append` | ✓ | ✓ |
  | `publish_matter_mention` | ✓ | ✓（v1 不开 UI，但允许手粘） |
  | `publish_matter_annotation` | ✓ | ✓ |
  | `publish_matter_owner_change` | ✓ reason | ✗ v1 不挂 |
  | `publish_matter_event` | ✗ | ✗ |

- [server/api/matters.py](../../server/api/matters.py) — `_render_item` 按 `file_rel` 反查附件，元数据透传对齐设计稿 §10.1
- [web/src/api.ts](../../web/src/api.ts) — `Attachment` 类型 + `TimelineFileItem.attachments?` + URL helper
- [web/src/components/matter/FileCard.tsx](../../web/src/components/matter/FileCard.tsx) — markdown 拦截 `attachment://` + body 下方附件区（设计稿 §3 统一列表）

---

## Phase 3：前端上传集成

> **交付**：从 CreateFileDialog / Append 编辑器走 UI 上传，支持点击、拖拽、粘贴。
> **依赖**：Phase 1 / 2。

涉及文件：

- [web/src/components/matter/AttachmentUploader.tsx](../../web/src/components/matter/AttachmentUploader.tsx) — 上传组件
- [web/src/components/matter/CreateFileDialog.tsx](../../web/src/components/matter/CreateFileDialog.tsx) — 编辑器接入 uploader
- [web/src/pages/NewMatter.tsx](../../web/src/pages/NewMatter.tsx) — **v1 不开**附件入口（创建前 `matter_id` 不存在），文档化"请创建后通过 append 添加"，不引入 `pre_create_token`

---

## Phase 4：AI 集成

> **交付**：AI 看到附件元数据，按需调工具读取正文 / 图像。
> **依赖**：Phase 2。

涉及文件：

- [server/api/ai.py](../../server/api/ai.py) — 复用 `_render_item`，不区分 vision 模型、不解析正文、不内联 base64
- [server/attachments_parsing.py](../../server/attachments_parsing.py) — 按 MIME 派发解析器（pdf / docx / xlsx / pptx / txt / md），失败降级返回空
- [server/mcp/tools.py](../../server/mcp/tools.py) + [server/mcp/instructions.py](../../server/mcp/instructions.py) — 新增 `read_attachment(id)` 工具，文档类返截断文本、图片类返 base64；instructions 引导 AI 按 description 自决

---

## E2E 验收

四阶段全部交付后，用一条真实 matter 跑通完整链路：

- [ ] 在 think dialog 粘贴一张截图 → 自动上传 + 在光标处插入 markdown 引用 + chip 出现
- [ ] 拖入一份 PDF → chip 出现，填一句 description
- [ ] 触发边界：上传第 4 个附件 / 同名文件 / 超过单文件大小 / 非白名单类型，分别看到对应中文错误提示
- [ ] 草稿态删 chip → body 中对应 `attachment://` 行同步消失
- [ ] 提交 think → FileCard 正常显示截图（点击放大）+ PDF 链接（点击下载）+ 附件区列表
- [ ] 用另一个无权限用户访问该附件 URL → 被拒
- [ ] 帖子 invalidate 后访问附件 → 按设计稿 §3 返回"所属帖子已作废"
- [ ] 在该 matter 跑 think AI：AI 输出能引用 PDF 正文内容（说明工具被调并解析成功）
- [ ] 多模态模型能描述截图内容；纯文本模型只用 description 回答、不调 image 类
- [ ] 全过程 `git status` 不感知附件文件（落在 `<DATA_DIR>/attachments/` 之外的仓库目录干净）
- [ ] 覆盖设计稿 §2 全部 4 个使用场景：方案上传、嵌入截图、验收证明、复盘材料
