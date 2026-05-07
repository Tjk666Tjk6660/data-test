---
type: think
author: yezaiyong
created: '2026-05-06T11:44:13+08:00'
body_source: ai
index_state: indexed
---
# MCP 发布 matter 接入可见范围 — 改造需求

## 背景

后端的发布 matter 接口（`POST /api/matters`）已支持设置可见范围，可选择"全部用户可见"或"限定特定角色/人员"。Web 端发布页面已完成接入，用户可在双栏选择器中进行配置。

目前 MCP 侧尚未跟进：通过 MCP 创建的 matter 一律按"全部可见"处理。当用户在 AI 对话中表达"这条只给 dev 看""只让小张看"等意图时，MCP 无法将此类限制落地。

## 当前差距

| 维度 | 后端 `POST /api/matters` | MCP `create_matter` |
|---|---|---|
| 设置 matter 可见范围 | ✅ 支持（公开 / 限定角色 / 限定到人） | ❌ 不接受参数，全部走公开 |
| 同时新建分类时设分类可见范围 | ✅ 可一并传入 | ❌ 不接受 |
| 校验 1：分类不存在 + matter 受限 → 必须同时带分类可见范围，否则报 `missing_category_visibility` | ✅ | ❌ 无 |
| 校验 2：matter 选的角色不能超过分类允许范围，否则报 `visibility_scope_exceeds_category` | ✅ | ❌ 无 |
| 校验 3：创建人/负责人必须在可见范围内，否则报 `visibility_excludes_required_user` | ✅ | ❌ 无 |
| 候选数据来源：`GET /api/visibility-options?category=...` 返回可选角色/成员 | ✅ | ❌ AI 无感知 |

## 改造方案

### 1. `create_matter` 新增"可见范围"参数

- **参数结构**：模式（公开 / 受限）、可见角色列表、可见成员列表。
- **向后兼容**：不传该参数时默认按"公开"处理，老调用方不受影响。
- **触发约定**：仅当用户在对话中明确说出"限制 / 只给 X 看 / 不要让 Y 看到"等话时才设置受限，不从 matter 正文中猜测意图（与现有 `mentions` / `status_change` 口径一致）。
- **MCP 层前置校验**：选受限模式时，至少需选中一个角色或一个人，否则在 MCP 层直接拒绝，不请求后端。

### 2. `create_matter` 新增"新分类可见范围"参数

- 仅在用户同时新建一个尚不存在的分类、且该 matter 为受限可见时才使用。
- **参数结构**：模式（公开 / 受限）、授权角色列表。
- 同样遵循"用户未明确说明则不传"的原则。

### 3. 新增 `list_visibility_options` 工具

- 封装后端 `GET /api/visibility-options?category=...`，使 AI 能获取当前分类下可选的角色和成员名单（含拼音、显示名等信息）。
- 用于 AI 在用户表达受限意图时，引导用户从合法候选中做选择。

## Review 人员

- @叶再勇（matter 责任人）
- 另邀 2 位同事参与 review（待补充）

## 待确认事项

1. MCP 层对后端三个校验错误的返回格式和 AI 侧的纠错话术如何设计？
2. `list_visibility_options` 的调用时机——是每次创建前都拉一次，还是按需缓存？
3. 用户表达可见范围意图的自然语言边界（哪些算"明确"，哪些需要 AI 反问确认）是否需要额外规则？
