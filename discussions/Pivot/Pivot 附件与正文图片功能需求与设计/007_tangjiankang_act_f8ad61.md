---
type: act
author: tangjiankang
created: '2026-05-08T11:16:10+08:00'
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

> **目标**：附件作为独立资源能上传、能鉴权访问、草稿态可删；不接 publish.py、不改 `_render_item`、前端零感知。

### Task 1.1：数据模型

**File:** [server/db.py](../../server/db.py)

新增 `attachments` 表，字段对照设计稿 §5.1。关键约束：

- `file_path` 可空 — 区分草稿态 / 已发布态
- `(matter_id, filename)` 唯一 — 兜底设计稿 §4 同名禁止规则

### Task 1.2：AttachmentRepo + 存储抽象

**File:** [server/attachments.py](../../server/attachments.py)

模块职责：

- **AttachmentRepo**：附件元数据的 CRUD + 状态迁移（草稿 → 已发布通过 `bind_to_file`）
- **Storage IO**：本地磁盘读写，根目录在 `<DATA_DIR>/attachments/<matter_id>/`，**Git 仓库目录之外**

接口契约（粒度到方法名 + 用途，不展开签名）：

| 方法 | 用途 |
|---|---|
| `create` | 写元数据 + 落盘；description 留空时用 filename 兜底 |
| `get / list_by_matter / list_by_file` | 读取，分别面向单条 / 草稿期前端列表 / 渲染期反查 |
| `bind_to_file` | publish 钩子调用，把草稿态附件锚到 timeline 文件 |
| `delete_draft` | 仅草稿态 + uploader 校验通过才删 |

### Task 1.3：HTTP 层

**File:** [server/api/attachments.py](../../server/api/attachments.py)

3 条路由，鉴权全部委托 matter ACL：

| 方法 | 路径 | 职责 |
|---|---|---|
| POST | `/api/matters/{matter_id}/attachments` | 上传，校验白名单 + 大小 + 数量 + 同名，返回元数据 + `markdown_ref` 让前端直接插入光标 |
| GET | `/api/attachments/{id}[?preview=1]` | 流式下载或预览（`Content-Disposition` 切换） |
| DELETE | `/api/attachments/{id}` | 仅草稿态可删，已绑定一律拒绝 |

阈值集中到 [server/settings.py](../../server/settings.py)（单文件大小、单帖数量、token 截断阈值），具体数值参考设计稿 §4 / §10.2。HTTP 状态码语义对齐设计稿 §4。

---

## Phase 2：发布期绑定 + 渲染

> **目标**：附件与 timeline item 建立关联，前端读路径打通。

### Task 2.1：publish bind 钩子

**File:** [server/publish.py](../../server/publish.py)

设计稿 §8 要求绑定在 `write_session` **退出之后**执行，避免 Git 锁。当前 publish.py 函数清单与挂钩决策：

| 函数 | 写 body | 挂 bind |
|---|---|---|
| `publish_matter_create` | ✓ | ✓ |
| `publish_matter_append` | ✓ | ✓ |
| `publish_matter_mention` | ✓ | ✓（v1 不开 UI，但允许手粘） |
| `publish_matter_annotation` | ✓ | ✓ |
| `publish_matter_owner_change` | ✓ reason | ✗ v1 不挂 |
| `publish_matter_event` | ✗ | ✗ |

抽公共绑定函数，从 body 提取 `attachment://` 引用 → 调 Repo 完成绑定。错误兜底：id 不存在 / 跨 matter → 静默丢弃，不阻断发布。

### Task 2.2：`_render_item` 注入元数据

**File:** [server/api/matters.py](../../server/api/matters.py)

按 timeline item 的 `file_rel` 反查附件，元数据字段对齐设计稿 §10.1（不暴露 `storage_key`、不解析正文、不发原图）。N+1 风险 v1 接受，单 matter timeline 通常 < 50 条。

### Task 2.3：前端渲染层

**Files:** [web/src/components/matter/FileCard.tsx](../../web/src/components/matter/FileCard.tsx)、[web/src/api.ts](../../web/src/api.ts)

- markdown 渲染拦截 `attachment://` 协议：图片 → 预览 URL，链接 → 下载 URL
- body 下方新增附件区，按 `item.attachments` 平铺展示（含 inline_image，设计稿 §3 统一列表）
- 鉴权失败 → 占位兜底
- API 类型补 `Attachment` + `TimelineFileItem.attachments?` + URL helper

---

## Phase 3：前端上传集成

> **目标**：用户能从 CreateFileDialog / Append 编辑器走 UI 上传，含粘贴、拖拽、删除。
> **依赖**：Phase 1（API）+ Phase 2（types）。

### Task 3.1：AttachmentUploader 组件

**File:** [web/src/components/matter/AttachmentUploader.tsx](../../web/src/components/matter/AttachmentUploader.tsx)

组件职责：

- 三种上传方式：点击选择、拖拽、粘贴（图片粘贴自动 POST + 在光标处插入 markdown 引用）
- 每个 chip 上方留 description 输入框 + placeholder 引导
- 错误提示按 HTTP 状态码分支给中文文案
- 草稿态删除 chip 时同步从 body 移除对应 `attachment://` 行

### Task 3.2：编辑器集成

**File:** [web/src/components/matter/CreateFileDialog.tsx](../../web/src/components/matter/CreateFileDialog.tsx)

`BodyMarkdownEditor` 外包一层 div 容纳 Textarea + uploader。提交时不需要传附件 id 列表，绑定钩子从 body 自动抽。dialog 取消时调 DELETE 清未绑定附件（避免孤儿，可选优化）。

### Task 3.3：NewMatter 决策

**File:** [web/src/pages/NewMatter.tsx](../../web/src/pages/NewMatter.tsx)

- 难点：创建前 `matter_id` 不存在，无法上传
- **v1 决策**：不开附件入口，文档化「请创建后通过 append 添加」；不引入 `pre_create_token` 复杂度

---

## Phase 4：AI 集成

> **目标**：AI 在 think AI / list_matter 等场景看到附件元数据，按需调工具拿正文 / 图像。
> **依赖**：Phase 2。

### Task 4.1：AI prompt 透传

**File:** [server/api/ai.py](../../server/api/ai.py)

复核 AI 路由是否复用 `_render_item`：是 → 零改动；否 → 加同样的元数据投影。**不区分 vision / 非 vision 模型；不解析正文；不内联 base64**。

### Task 4.2：read_attachment MCP 工具

**Files:** [server/mcp/tools.py](../../server/mcp/tools.py)、[server/mcp/instructions.py](../../server/mcp/instructions.py)、[server/attachments_parsing.py](../../server/attachments_parsing.py)

工具职责：

- 入参：附件 id；鉴权复用 MCP context user → matter ACL
- 文档类（pdf / docx / xlsx / pptx / txt / md）→ 调用 parsing 模块提取纯文本，按 token 阈值截断
- 图片类 → 返回 base64 + content_type，由 AI 自决用法
- system instructions 追加引导（设计稿 §10.2）：「根据 description 判断相关性、根据自己能力决定是否调用」

parsing 模块按 MIME 派发到对应解析器，失败降级返回空字符串（AI 自动回退到只看 description）。

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
