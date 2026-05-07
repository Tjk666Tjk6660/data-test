---
type: think
author: zhangbo
created: '2026-05-07T09:43:08+08:00'
index_state: indexed
---
@dengke 反馈很准，已按这条路全面重写。原版"默认全注入 + 后端 capability 判断"出发点是"AI 看不到附件答得不准"，但你点出几个我没想到的：维护负担、容量爆炸、解析失败影响主链路、契合 Agent 范式。换路了。

## 1. 后端只透传元数据（§10.1）

`_render_item` 在 timeline item 上挂 `attachments[]`，字段：`{ id, filename, content_type, size, usage, description }`。不解析正文、不发原图、不暴露 `storage_key`。`server/api/ai.py` 几乎零改动。

## 2. AI 端用 `read_attachment(id)` 按需取（§10.2）

- 文档类（pdf/word/excel/ppt/txt/md）→ 解析为文本返回；**默认截断阈值 8000 token**
- 图片类 → 返回 base64 + content_type；纯文本模型自动跳过，多模态模型自决塞下一轮 message

8000 token 的由来（写进 §10.2 表格了）：§4 单文件 2 MB，Office 解析后通常 5K–30K 字；8000 token 在主流分词器下覆盖 5K–8K 字，多数 Pivot 实际附件一次拿全。4000 太紧、16000 太膨胀，8000 是折中。v1 不做分页参数。

## 3. description 留空兜底（§10.3 + §5.1）

不强制必填。留空时后端用去扩展名后的 filename 兜底（`客户验收签字.png` → `"客户验收签字"`）。前端 UI 仍给 placeholder 引导主动填更有意义的说明。

## 4. 顺手收紧的几条

- **§4 同名约束**：从"同帖"扩到"同 matter"——根上消除版本歧义（要新版本就改名 `方案-v2.pdf`）
- **§11 开放问题**：把"自动 description vision pipeline"换成"后端主动注入附件正文/原图"，固化哲学
- **§11 附件版本管理** v1 决策改写为"靠同 matter 不可同名 → 同一份文件只有一个 id"

## 5. read_attachment 工具落地

- `server/mcp/tools.py` 新增工具
- `server/attachments_parsing.py` 文档解析仅在工具内部调用，不在主链路
- v1 无 `offset/limit` 分页参数

要再过一轮吗？
