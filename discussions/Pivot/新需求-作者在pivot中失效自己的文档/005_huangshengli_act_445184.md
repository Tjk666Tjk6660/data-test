---
type: act
author: huangshengli
created: '2026-05-06T11:18:41+08:00'
index_state: indexed
---
按 004 评审稿全量落地，对应 `AI-docs/invalidate-self/` 下的 product-design.md (v2) 和 implementation-plan.md。

## 实现范围（P1–P5）

### P1 数据层
- `server/doc_types.py`：`VALID_INVALIDATION_REASONS = {misposted, inaccurate}`；`restored` 不在原因集合里，而是通过"独立的恢复事件项"语义化区分（与 002/004 设计一致）。
- `server/matter_index.py`：
  - file 项扩展 4 个反写字段 `invalidated / invalidated_at / invalidated_reason / invalidated_by`
  - 新增 `_INVALIDATION_EVENT_KEY_ORDER`，`_normalize_item` 按 `type` 是否存在 + `reason` 分派 file / owner_change / 失效事件三种 key 顺序
  - `_reverse_write_invalidation` 把事件反写到目标 file 的 4 字段；恢复事件只翻转 `invalidated=false`，保留另外 3 字段作为审计轨迹
  - `append_event(path, event, now_iso)`：原子 tmp+rename，**不**bump `matter.updated_at`（失效不算 matter 进度）
- `server/matter_validator.py`：`validate_append` 入口分派；新增 `_validate_invalidation_event_shape`（6 条规则：creator 必须等于 file.creator、不可重复失效、未失效不可恢复 …）；新增 `_validate_no_quote_to_invalidated` 实现设计 §5.3 引用块（新文件的 quote/refer 不能指向已失效文件）。

### P2 API + 写路径
- `POST /api/matters/{id}/events`（body：`target_file / reason / summary?`），rerequest 走标准 422 / 404 / 409 状态码映射。
- `server/publish.py::publish_matter_event` 在 `workspace.write_session(...)` 内调用 `matter_append_event`，紧接 `emit(TOPIC_MATTER_EVENT_APPENDED)` + `notifier.notify_matter_event`。

### P3 SSE
- 新增 `TOPIC_MATTER_EVENT_APPENDED`，映射为 `matter.updated, reason="event_appended"`。
- **沿用 thin SSE**（payload 只携带 `matter_id / actor / reason / at`，客户端 refetch 详情读真实数据），与现有 `file_appended / comment_appended / owner_changed` 一致，避免引入 fat payload 通道。

### P4 通知
- `Notifier` 协议加 `notify_matter_event`；`FeishuNotifier.build_matter_event_card` 用不同主题色（撤回黄 / 恢复蓝绿）；`NoOpNotifier` 走空实现。

### P5 前端
- 类型层（`web/src/api.ts`）：`TimelineFileItem` 扩 4 字段；新增 `TimelineInvalidationEventItem`（无 `type` 字段、有 `reason`）；3 个 type guard（`isTimelineFileItem` / `isTimelineOwnerChangeItem` / `isTimelineInvalidationEventItem`）。
- 组件：
  - `InvalidatedBadge`：FileCard 头部"已失效（理由）"标签，hover 露出 operator
  - `InvalidateDialog`：撤回 / 恢复二合一弹窗，原因 radio + 可选 summary
  - `InvalidationEventRow`：主流横向事件行（mirror `OwnerChangeRow` 风格：头像 + actor + 动词 + 目标文件 + 理由 chip + 时间）
- `FileCard`：作者专属"⊘ 撤回 / ↺ 恢复"按钮（鉴权 `me.pinyin === item.creator`）
- `TimelineStrip`：失效文件节点变灰 + 删除线 + 灰色圆点
- `CreateFileDialog`：§5.3 阻断 — quote / refer picker 屏蔽已失效文件

## 关键决策

1. **SSE thin pattern**（P3 对齐时确定）：和现有事件一致，前端收到 nudge 后调 `fetchMatter` 取真实数据，避免引入第二种 SSE 形态。
2. **失效事件在主流展示**（集成测试后调整）：`InvalidationEventRow` 渲染在主文件流，与 `OwnerChangeRow` 风格一致；重复撤回↔恢复会堆叠 = 审计轨迹。
3. **TimelineStrip 节点策略**：file 圆形彩色（可跳）+ owner_change 灰色斜方块（不可跳，恢复了 invalidate-self 改造时被误删的分支）+ 失效事件不绘节点。"时间轴 · N" = file + owner_change，与 strip 实际节点数严格一致；卡片头部 / 侧边栏的 "N 个文件" 仍是纯文件数。
4. **owner_change strip 标签用原始 `actor`（pinyin）**：和 file 节点的 `creator` 字段保持渲染口径一致，避免同一用户在 strip 上同时出现 pinyin 和飞书 display name。

## 文档同步

- `AI-docs/pivot-product.md`：新增 §八.5（timeline 三种事件项形态）+ §八.6（file 反写字段语义）
- `AI-docs/pivot-interface.md`：`POST /api/matters/{id}/events` 完整契约 + SSE 章节列出新 reason
- `AI-docs/invalidate-self/product-design.md` 与 `implementation-plan.md`：同步 thin SSE 决策

## 已交付（main 分支）

- `5b80526 feat(invalidate-self): author can withdraw / restore own files` — P1-P5 主体
- `dbb82be fix(matter-timeline): render invalidation events + restore owner_change strip node` — 集成测试发现的两处归位（失效事件未渲染主流；owner_change strip 节点缺失）

集成测试覆盖：单次撤回 / 撤回后恢复 / 多轮撤回-恢复 / §5.3 quote 阻断 / 非作者无按钮 / strip 节点 + 计数 + 主流事件行同步刷新。
