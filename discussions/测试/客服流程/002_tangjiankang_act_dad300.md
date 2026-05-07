---
type: act
author: tangjiankang
created: '2026-05-07T18:10:04+08:00'
body_source: manual
index_state: indexed
---
0. 与代码现状的核对结论
设计假设	现状	备注
server/db.py 的 SCHEMA 末尾加 attachments 表	✅ server/db.py:10-397 是单一 SCHEMA 字符串，新表加在末尾即可	无需写 _migrate 分支（全新表）
3 个 publish 函数（create / append / comment）末尾加 bind 钩子	⚠️ 现实是 6 个函数：publish_matter_create / publish_matter_append / publish_matter_mention / publish_matter_annotation / publish_matter_owner_change / publish_matter_event。设计里说的 "comment" 对应代码中的 publish_matter_mention；publish_matter_annotation 也有 body 但设计未提及	建议把 annotation 也加入 bind 钩子（一致性 + 保险），见下方 §3
_render_item 注入 attachments	✅ server/api/matters.py:1342，分支干净（owner_change / invalidation 早返）	在 file 分支末尾追加 out["attachments"] = repo.list_for_file(matter_id, file_rel)
<DATA_DIR>/attachments/... 不在 git 仓库下	✅ git 在 <DATA_DIR>/git/<repo>（server/app.py:133），attachments 与 git 同级	无需改 .gitignore
MCP 增 read_attachment 工具	✅ server/mcp/tools.py + server/mcp/instructions.py 注册新工具	instructions 里能力清单要从 9 个加成 10 个
鉴权用 cookie session + 每次重校验	现有 make_current_user / matter ACL 已经齐备，attachment 路由直接复用	
设计基本可直接执行，无大方向冲突。

1. 推荐分阶段推进
Phase	范围	可独立验证
P1 后端存储层	DB schema + AttachmentRepo + 本地 IO + 配置	跑单测验证增删查 + 同名约束 + 大小限制
P2 上传/下载/预览 API	server/api/attachments.py 三条路由，还不接 publish	curl multipart 上传、下载、预览均正常
P3 publish bind 钩子	publish_matter_create / publish_matter_append / publish_matter_mention 末尾扫 body 调 bind_to_file	跑端到端：草稿态附件未绑定 → 发布后 file_path 已绑
P4 _render_item 透传 + AI prompt	matters.py + ai.py 元数据透传	timeline 接口返回 attachments[]
P5 MCP read_attachment	tools.py + instructions.py + 文档 / 图片解析	MCP 客户端调用真实附件
P6 前端编辑器	AttachmentUploader + CreateFileDialog + NewMatter	拖拽 / 粘贴 / 上传按钮三种入口；草稿态删改
P7 前端渲染	FileCard：attachment:// img/a 拦截 + 附件列表区	看正文图片渲染 + 点击放大 + 下载
P8 收尾	限制可配置化、e2e 测试、文档	
每个 Phase 都不阻塞下一个调试，P1 + P2 完成就能 curl 验证，前端不必等到。

2. P1 — 后端存储层
2.1 数据库表（加在 server/db.py:397 SCHEMA 字符串末尾，""" 之前）

CREATE TABLE IF NOT EXISTS attachments (
    id              TEXT PRIMARY KEY,                      -- 'att_' + ULID
    matter_id       TEXT NOT NULL,
    file_path       TEXT,                                  -- 草稿态 NULL；发布后 'discussions/<cat>/<slug>/<file>'
    filename        TEXT NOT NULL,                         -- 原始文件名（含扩展）
    content_type    TEXT NOT NULL,
    size            INTEGER NOT NULL,
    storage_key     TEXT NOT NULL,                         -- 'matters/<matter_id>/<id>.<ext>'
    uploaded_by     TEXT NOT NULL,                         -- pivot_user.id
    usage           TEXT NOT NULL CHECK(usage IN ('attachment','inline_image')),
    description     TEXT NOT NULL DEFAULT '',
    created_at      REAL NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_attachments_matter        ON attachments(matter_id);
CREATE INDEX IF NOT EXISTS idx_attachments_matter_file   ON attachments(matter_id, file_path);
CREATE UNIQUE INDEX IF NOT EXISTS idx_attachments_matter_filename
    ON attachments(matter_id, filename);    -- §4 同 matter 不同名
UNIQUE(matter_id, filename) 在 DB 层兜底设计中的"同名约束"，重复上传时返回 IntegrityError → API 层翻成 409。

2.2 新文件 server/attachments.py

class AttachmentRepo:
    def __init__(self, db: Database, root: Path): ...
    # CRUD
    def create(*, matter_id, filename, content_type, size, storage_key,
               uploaded_by, usage, description) -> Attachment        # 返 dataclass
    def get(att_id) -> Attachment | None
    def list_for_matter(matter_id) -> list[Attachment]               # 编辑器用
    def list_for_file(matter_id, file_rel) -> list[Attachment]       # _render_item / 渲染用
    def list_unbound_for_matter_user(matter_id, uploader) -> list    # 编辑器草稿态拉自己未发布的
    # 草稿态删除（仅 file_path IS NULL）
    def delete_unbound(att_id, uploader) -> bool                     # 删 DB + 删盘
    # 发布时绑定
    def bind_to_file(ids: list[str], file_rel: str) -> int
    # 计数（同帖上限校验）
    def count_for_file(matter_id, file_rel) -> int
    def count_unbound(matter_id, uploader) -> int

    # 本地 IO
    def save_blob(storage_key: str, fileobj) -> int                  # 写盘 + 返 size
    def open_blob(storage_key: str) -> BinaryIO
    def delete_blob(storage_key: str) -> None
落盘根目录：cfg.data_dir / "attachments"，server/app.py:92 创建 repo 后注入 router（与现有 DraftRepo 同模式）。

2.3 新文件 server/attachments_parsing.py

TRUNCATE_TOKENS = int(os.getenv("ATTACHMENT_READ_TOKEN_LIMIT", "8000"))

def extract_text(content_type: str, blob_path: Path) -> tuple[str, int]:
    """Returns (truncated_text, total_chars). 实际按字符近似 token，
    保守 1 token ≈ 1.5 中文字。"""
    # pdf  → pypdf
    # docx → python-docx
    # xlsx → openpyxl 仅取单元格文本
    # pptx → python-pptx
    # txt/md → 直接读
依赖加进 pyproject.toml：pypdf、python-docx、openpyxl、python-pptx。

2.4 配置参数（server/config.py:11）
新增三个可选环境变量（带默认值，无需新 frozen 字段也可）：


ATTACHMENT_MAX_SIZE_MB=2
ATTACHMENT_MAX_PER_FILE=3
ATTACHMENT_READ_TOKEN_LIMIT=8000
放进 Config 比 os.getenv 更整齐，但短期可先 os.getenv 直接读。

3. P2 — server/api/attachments.py 三条路由

@router.post("/api/matters/{matter_id}/attachments")
async def upload(
    matter_id: str,
    file: UploadFile,
    description: str = Form(""),
    usage: Literal["attachment", "inline_image"] = Form("attachment"),
    user: PivotUser = Depends(current_user_dep),
):
    # 1. ACL: matter 可见性 → 复用 acl_cache / visibility 模块
    # 2. 扩展名 + content_type 双重白名单
    # 3. size 限制 → 413 'file_too_large'
    # 4. 本帖（unbound by uploader）+ 已绑定到 file 的总数 → 422 'too_many_attachments'
    #    草稿态难点：还没决定要绑哪个 file。改成 "uploader 在该 matter 下未绑定附件数 ≤ 3"
    # 5. 同 matter 同名 → 409 'duplicate_filename'（DB UniqueConstraint 转译）
    # 6. 写盘 + repo.create → 返 {id, markdown_ref, ...}
    return {
        "id": att.id,
        "filename": att.filename,
        "content_type": att.content_type,
        "size": att.size,
        "usage": att.usage,
        "description": att.description or _strip_ext(att.filename),
        "markdown_ref": f"![{att.description or _strip_ext(att.filename)}](attachment://{att.id})",
    }


@router.get("/api/attachments/{att_id}")
def stream(att_id: str, preview: int = 0, user: PivotUser = Depends(current_user_dep)):
    # 1. attachments[id] → matter_id → ACL 校验
    # 2. invalidated 检查：file_path 对应的 timeline item 已 invalidated → 409 'post_invalidated'
    # 3. StreamingResponse(repo.open_blob(storage_key))
    #    Content-Disposition：preview=1 → inline；否则 attachment; filename*=...UTF-8''<urlencoded>
    #    Content-Type 用 attachments.content_type
    #    Cache-Control: private, max-age=60（可选）


@router.delete("/api/attachments/{att_id}")
def delete(att_id: str, user: PivotUser = Depends(current_user_dep)):
    # 仅草稿态 (file_path IS NULL) + uploader 自己；其它一律 403
⚠️ 数量上限的实现：因为附件是"先传后绑"，发起上传时还不知道会绑到哪个 file。建议把 §4 的"单帖 ≤3"在草稿态以"该 uploader 在该 matter 下 unbound + 当前 file 已绑 ≤ 3"两段加和近似（绝大多数场景 uploader 一次只编辑一个 file）。简单起见，v1 直接用 count_unbound(matter_id, uploader) 作为上限，文案保持"附件数已达上限"。如果后续要精确按 file 算，再让前端在请求中带"目标 file_rel"提示。

4. P3 — publish bind 钩子
在 server/publish.py 三个写函数 write_session 退出之后 加扫描钩子：


import re
_ATT_REF_RE = re.compile(r"attachment://(att_[A-Za-z0-9]+)")

def _scan_and_bind(repo: AttachmentRepo, body: str, matter_id: str, file_rel: str):
    ids = sorted(set(_ATT_REF_RE.findall(body or "")))
    if ids:
        repo.bind_to_file(ids, file_rel, matter_id=matter_id)  # repo 内部校验 matter_id 一致
接入点：

函数	body 来源	bind 目标 file_rel
publish_matter_create（publish.py:699）	md_body	file_rel（已经在函数里算好）
publish_matter_append（publish.py:873）	md_body	file_rel
publish_matter_mention（publish.py:1010，对应设计中的 "comment"）	body 参数	target_file
publish_matter_annotation（publish.py:1120，设计未提）	body 参数	target_file
建议把 annotation 也加进 bind 列表 — 跟 mention 同样有 body，行为应一致；不接的话 annotation 里写 attachment:// 会变死链。在 PR 里单独提一下这点请 zhangbo 确认。

publish_matter_event / publish_matter_owner_change 无 body，跳过（设计已说明）。

把 attachments_repo: AttachmentRepo | None = None 加到这 4 个函数签名，由 app.py 注入到对应 API 路由的依赖。

5. P4 — _render_item 透传 + AI
5.1 server/api/matters.py:1342 _render_item
在 file 分支末尾（out["annotations"] = ... 之后）加：


file_rel = item.get("file") or ""
if file_rel and attachments_repo is not None:
    out["attachments"] = [
        {
            "id": a.id, "filename": a.filename,
            "content_type": a.content_type, "size": a.size,
            "usage": a.usage,
            "description": a.description or _strip_ext(a.filename),
        }
        for a in attachments_repo.list_for_file(matter_id, file_rel)
    ]
else:
    out["attachments"] = []
attachments_repo 通过 _render_matter_detail → _render_item 链路注入（router build 时传进去）。

5.2 server/api/ai.py
AI 模块也需要拿到 attachments 元数据。它要么自己再 render 一次，要么调 matters 模块的 render 函数。建议把 _render_item/_render_matter_detail 提到 server/matter_render.py（共享）；api/matters.py 和 api/ai.py 都 import。改动小、副作用可控。

不解析正文、不发原图（设计§10.1）。

6. P5 — MCP read_attachment
6.1 server/mcp/tools.py 新增

async def read_attachment(input: ReadAttachmentIn, user: PivotUser) -> ReadAttachmentOut:
    # ACL 复用 attachments 路由的 matter ACL
    att = repo.get(input.id)
    if att is None: raise ToolError("not_found")
    enforce_matter_acl(att.matter_id, user)  # 注意 MCP 的 user 是 PAT 解出来的
    blob_path = repo.path_for(att.storage_key)
    if att.content_type.startswith("image/"):
        b64 = base64.b64encode(blob_path.read_bytes()).decode()
        return ReadAttachmentOut(kind="image", content_type=att.content_type, base64=b64)
    text, total = extract_text(att.content_type, blob_path)
    return ReadAttachmentOut(
        kind="text", text=text, total_chars=total,
        truncated=(len(text) < total),
    )
Schema in server/mcp/schemas.py：ReadAttachmentIn { id: str }、ReadAttachmentOut { kind, content_type?, text?, base64?, total_chars?, truncated? }。

6.2 server/mcp/instructions.py:31
能力列表加一行：


- read_attachment — 按 id 读取附件正文 / 原图（"展开 X 这个附件" / "看下 003 那篇里那张截图"）
并在能力总数从"9 个"改为"10 个"。

6.3 server/mcp/server.py 工具注册
按照现有 9 个工具的注册模板照抄一份。

7. P6 — 前端编辑器
7.1 新组件 web/src/components/matter/AttachmentUploader.tsx

type Props = {
  matterId: string;          // 草稿态 matter 还没创建？— 见下方"草稿态对接"
  attachments: Attachment[];
  onUploaded: (att: Attachment, markdownRef: string, atSelection: boolean) => void;
  onRemoved: (id: string) => void;
};
主要交互：

按钮 / 拖拽 / 粘贴三入口都走同一个 uploadAttachment；图片走 usage=inline_image 默认插入光标位置 markdown_ref，其它走 usage=attachment 仅挂 chip。
上传成功后调 onUploaded，编辑器在 cursor 位置插入 markdown_ref（图片）或不插入（普通附件）。
列表渲染：每行 [icon][filename][size][description?] [删除]。删除任意一项 → DELETE 接口 + 本地 state 移除 + 同步把 body 中所有 attachment://<id> 行删掉（用一个简单的 regex ^.*attachment://${id}.*$，整行去掉，避免空 alt 残留）。
7.2 web/src/components/matter/CreateFileDialog.tsx
BodyMarkdownEditor 包一层 div，下方挂 <AttachmentUploader>；onPaste 先尝试图片 → 上传 → 插入 markdown_ref，否则 native paste。

7.3 web/src/pages/NewMatter.tsx
注意：新建 matter 时 matter_id 还不存在！设计里的"草稿态自由删改"前端对此未明说。两种处理：

选项 A（推荐）：新建 matter 流程的附件先用一个临时 draft_id（前端 uuid），上传 API 改成 POST /api/matters/_draft/{draft_id}/attachments，提交 matter 时把 draft_id 一起 POST 给 POST /api/matters，后端把 attachments 的 matter_id 重写为新创建的 matter_id 后再 bind。需要 attachments 表 matter_id 列允许临时值，加一个 draft_id 列做映射。
选项 B（更简）：新建 matter 这一类草稿态不开附件，仅在已存在的 matter 里 append 时支持附件。在 v1 这能覆盖 90% 场景。
⚠️ 这是设计文档里没明确的一个空档，建议在动手前找 zhangbo 确认走 A 还是 B。我倾向 B（v1 简化，UX 损失小：要附件就先建空 matter 再传），减少一条临时存储路径。

7.4 web/src/api.ts

export type Attachment = {
  id: string;
  filename: string;
  content_type: string;
  size: number;
  usage: 'attachment' | 'inline_image';
  description: string;
  markdown_ref?: string;  // 仅上传响应带
};

export type TimelineFileItem = {
  // ...existing fields
  attachments?: Attachment[];
};

export const uploadAttachment = (matterId, file, opts) => ...
export const deleteAttachment = (id) => ...
export const attachmentUrl = (id, preview = false) =>
  `/api/attachments/${id}${preview ? '?preview=1' : ''}`;
8. P7 — 前端渲染（web/src/components/matter/FileCard.tsx）
8.1 markdownComponents（FileCard.tsx:50）

img: ({ src, alt }) => {
  if (typeof src === 'string' && src.startsWith('attachment://')) {
    const id = src.slice('attachment://'.length);
    return (
      <img
        src={attachmentUrl(id, true)}
        alt={alt}
        className="max-w-full cursor-zoom-in"
        onClick={() => openLightbox(id)}
      />
    );
  }
  return <img src={src} alt={alt} />;
},
a: ({ href, children }) => {
  if (typeof href === 'string' && href.startsWith('attachment://')) {
    const id = href.slice('attachment://'.length);
    return <a href={attachmentUrl(id, false)} download>{children}</a>;
  }
  return <a href={href}>{children}</a>;
},
8.2 body 下方附件区
读 item.attachments → 渲染统一 chip 列表（含 inline_image，与 §3 设计一致）；点击图片项滚到正文中对应的 attachment://<id> 锚点（用 id 作为 anchor 标记，渲染 img 时同时输出隐藏 <span id="att-<id>"/>）；普通附件点击下载。
