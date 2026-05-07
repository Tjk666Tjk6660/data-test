---
type: act
author: lishuai
created: '2026-04-29T21:02:26+08:00'
index_state: indexed
---
## 最终实现方案（2026-04-29）

本节记录已落地版本，优先级高于上方早期设计草案中的旧描述。

### 目标与边界

- Matter 增加 matter 级 owner，表示“当前由谁继续推进这件事”。
- 文件级 owner 继续保留在 `timeline[i].owner`，表示单条 think / act 文件的作者或执行人。
- matter owner 与 file owner 语义独立，互不覆盖。
- v1 不做审批、权限限制、多人 owner、撤销。
- owner 变更不再联动 matter status，也不再展示“转交 + 状态变更”的合并操作。

### 数据模型

`{matter_id}.index.yaml` 的 `matter` block 增加：

```yaml
matter:
  id: example
  title: example
  current_status: planning
  owner: zhangsan
  created_at: '2026-04-29T10:00:00+08:00'
  updated_at: '2026-04-29T10:00:00+08:00'
```

owner 变更作为 timeline 事件落盘：

```yaml
timeline:
  - type: owner_change
    created_at: '2026-04-29T10:30:00+08:00'
    actor: lisi
    from_owner: zhangsan
    to_owner: wangwu
    reason: 后续由王五继续推进
```

`owner_display` / `owner_avatar_url` / `actor_display` / `to_owner_display` 等展示字段不落盘，由 API render 层实时解析。

### 历史数据兼容

最终采用“显式 owner 优先，缺失时回退”的 effective owner 规则：

1. 如果 `matter.owner` key 存在，直接使用该值；即使值为 `null`，也表示显式未分配。
2. 如果 `matter.owner` key 缺失，回退到第一条非 `owner_change` timeline item 的 `owner`，再回退到 `creator`。
3. 如果仍无法解析，API 返回 `null`，前端展示“未分配”。

该规则用于列表、详情、`?owner=` 查询、owner 变更校验和 owner 变更通知，避免旧 index 在页面显示 A、通知却显示“未分配”的不一致。

仍保留 `server/scripts/backfill_matter_owner.py`，上线时可把历史 index 的 `matter.owner` 回填出来。回填脚本幂等；不回填也能依靠运行时 fallback 正常展示和转交。

### 后端 API

创建 matter：

```http
POST /api/matters
```

请求体新增可选字段：

```json
{
  "category": "Pivot",
  "title": "客服优化方案",
  "owner_open_id": "ou_xxxxxx",
  "initial_file": {
    "type": "think",
    "summary": "...",
    "body": "...",
    "owner": "ou_yyyyyy"
  }
}
```

- `owner_open_id` 是 matter 级 owner，缺省为创建者。
- `initial_file.owner` 是文件级 owner，继续写入 `timeline[0].owner`。
- 创建时指定他人为 owner 不生成 `owner_change` timeline 事件。

更改 owner：

```http
POST /api/matters/{matter_id}/owner
```

```json
{
  "to_owner": "ou_xxxxxx",
  "reason": "后续由他继续推进"
}
```

行为：

- 校验 matter 存在。
- 校验 `to_owner` 能解析为用户或联系人。
- 校验 `to_owner` 与当前 effective owner 不同。
- 校验 `reason` 非空，最长 200。
- 写入一条 `owner_change` timeline event。
- 更新 `matter.owner` 和 `matter.updated_at`。
- 返回渲染后的 `{ matter, item }`，不是 raw yaml 数据。
- 对外 SSE 继续使用 `matter.updated`，payload 带 `reason: "owner_changed"`。
- 内部 topic 使用 `matter.owner_changed`，与实现保持一致。

`GET /api/matters?owner=` 的语义已统一为优先查 matter effective owner；历史缺失 `matter.owner` 的 index 会按 fallback owner 参与匹配。

### 前端交互

新建 matter：

- classic form 和 guided flow 都支持指定 matter 责任人。
- guided flow 中责任人步骤位于“标题”之后、“圈人”之前。
- 默认责任人是当前用户。
- 责任人选择框保持在底部输入区，不覆盖对话内容；搜索框和结果列表可正常选择。

详情页：

- 顶部卡片展示 owner chip。
- 不再展示单独的“转交”按钮。
- 点击 owner chip 直接打开“更改负责人”弹窗。
- 弹窗仅包含当前负责人、新负责人、变更原因。
- 不再提供“同时推进至 executing”或任何状态变更选项。
- 新负责人不能等于当前负责人，错误提示使用中文。

左侧列表：

- 左侧 matter 列表展示 owner，但不覆盖帖子标题。
- owner 作为 meta 行中的轻量信息展示，和文件数、最近活动、状态 badge 同行或相邻排列。
- 标题仍是列表项主信息，owner 只是辅助扫描信息。

Timeline：

- `owner_change` 使用独立事件行展示。
- 不展示状态变更块。
- 文案使用“负责人变更 / 更改负责人”，避免继续出现“转交 + 状态变更”的语义。

### 飞书通知

新建 matter 群通知：

- 群消息仍发送新讨论主题通知。
- 卡片正文中展示 `负责人：@张三`，而不是只在卡片顶部单独圈人。
- 如果还有额外圈人设置，负责人和圈人都可出现在通知中，但负责人必须在“负责人”字段里可读可见。
- 卡片内容使用左侧 label + 右侧内容的正常对齐格式。

owner 变更通知：

- 不再发送群消息。
- 只给新 owner 发送飞书单聊 IM。
- 文案提示“负责人变更”，字段包括操作、负责人、新负责人、原因和查看讨论按钮。
- 发送路径使用 `_dm_many([to_owner_open_id], ...)`，避免打扰群。
