---
type: think
author: huangshengli
created: '2026-04-28T20:26:16+08:00'
index_state: indexed
---
# 失效文档需求 — 设计方案评审稿

根据需求整理出可执行的数据结构与行为规范。本文是设计层面的完整方案,提交团队评审,确认无误后再进入实施阶段。

## 一、设计原则

保持 Pivot timeline 不可篡改的核心前提下,提供"作者本人将自己已发布文档标记为失效"的能力。

**三条不变量:**

- 原文件不删除、不修改、不抹除
- 失效操作本身作为 timeline 上的可审计事件,与文件具有同等的不可篡改性
- 失效不影响 matter 状态机(`current_status` / `status_change` 历史)

## 二、数据结构

> **以下是建议的数据结构方案,请评审确认是否可行。** 如有任何字段命名、形态、语义上的疑虑,请提出。

### 2.1 文件项新增反写字段(4 个)

```yaml
- file: discussions/Pivot/xxx/003_dengke_act_abc.md
  type: act
  creator: dengke
  ...   # 原有字段不变
  # ↓ 反写字段
  invalidated: bool                              # 当前是否失效(权威)
  invalidated_at: <iso datetime>                 # 最近一次失效时间
  invalidated_reason: misposted | inaccurate | sensitive
  invalidated_by: <pinyin>                       # 最近一次失效操作人
```

恢复后保留 `invalidated_at` / `invalidated_reason` / `invalidated_by`,仅翻转 `invalidated: true → false`。这三个字段含义变成"最近一次失效的元数据",再次失效时被新值覆盖。

### 2.2 失效 / 恢复事件项(新增 timeline entry 形态)

事件项是 timeline 上的一种**非文件型记录**,沿用现有 timeline event 的字段约定,但不在 `type` 枚举中:

```yaml
- creator: dengke                                  # 必填
  created_at: 2026-04-26T10:00:00+08:00            # 必填
  quote: <被操作文件 path>                         # 必填
  reason: misposted | inaccurate | sensitive | restored   # 必填
  summary: "误发,撤回此文档"                       # 可选自由说明
```

**reason 枚举共用,单字段同时编码"事件类型 + 具体原因":**

| reason 值 | 事件类型 | 系统反写 |
|---|---|---|
| `misposted` | 失效 | invalidated=true, invalidated_at=now, invalidated_reason=misposted, invalidated_by=creator |
| `inaccurate` | 失效 | 同上,reason=inaccurate |
| `sensitive` | 失效 | 同上,reason=sensitive |
| `restored` | 恢复 | invalidated=false(仅翻转 bool,其他三个字段保留) |

### 2.3 完整数据结构示例

下面是一个 matter index 完整示例,展示文件项、失效事件、恢复事件三者在 timeline 上的共存形态:

```yaml
matter_id: 示例-某 matter
title: 示例-某 matter
current_status: planning
created_at: 2026-04-25T21:00:00+08:00
updated_at: 2026-04-27T15:00:00+08:00

timeline:
  # ── 文件项 #1: 普通的 think,从未被失效 ─────────────────
  - file: discussions/Pivot/xxx/001_dengke_think_aaa.md
    type: think
    creator: dengke
    owner: dengke
    created_at: 2026-04-25T21:00:00+08:00
    summary: "初始想法"
    quote: null
    refer: []
    comments: []

  # ── 文件项 #2: 一度被失效又恢复的 act ───────────────────
  - file: discussions/Pivot/xxx/003_dengke_act_bbb.md
    type: act
    creator: dengke
    owner: dengke
    created_at: 2026-04-25T22:00:00+08:00
    summary: "推进登录回跳"
    quote: null
    refer: []
    # ↓ 反写字段:当前 invalidated=false,但保留最近一次失效的元数据
    invalidated: false
    invalidated_at: 2026-04-26T10:00:00+08:00
    invalidated_reason: misposted
    invalidated_by: dengke
    comments: []

  # ── 事件项 #1: 2026-04-26 dengke 失效了 003 ─────────────
  - creator: dengke
    created_at: 2026-04-26T10:00:00+08:00
    quote: discussions/Pivot/xxx/003_dengke_act_bbb.md
    reason: misposted
    summary: "误发,撤回此文档"

  # ── 事件项 #2: 2026-04-27 dengke 恢复了 003 ─────────────
  - creator: dengke
    created_at: 2026-04-27T15:00:00+08:00
    quote: discussions/Pivot/xxx/003_dengke_act_bbb.md
    reason: restored
    # 无 summary,无文字补充
```

要点说明:

- 文件项通过 `type` 字段在枚举中(think/act/...)标识自己,事件项**没有 `type`**,通过其有 `reason` 而无 `type` 自然区分
- 文件项 #1 没有任何 invalidated 相关字段(向前兼容,旧数据不需要 migration)
- 文件项 #2 当前 `invalidated: false`(可正常展示),但保留了之前失效时的时间戳、理由、操作人——这是"曾经失效过"的审计痕迹,timeline 上配合事件项 #1 + #2 形成完整故事
- 事件项之间按 `created_at` 排序,与文件项混在同一 timeline 中

### 2.4 timeline 形态约束

文件项与事件项通过"是否有 `type` 字段"自然区分,不引入显式判别字段:

- **文件项**:保留 `file` 字段指向自身 md 文件,所有现有约束不变
- **事件项**:不带 `file` 字段,靠 `quote` 引用被操作的目标文件

## 三、校验规则(matter_validator)

建议新增 6 条规则:

1. 事件项 `creator` 必须等于 `quote` 指向文件的 `creator`(v1 作者本人限定)
2. 已失效(`invalidated == true`)文件不能再发失效事件,必须先恢复
3. 未失效(`invalidated != true`)文件不能发恢复事件
4. 事件项的 `quote` 不能指向另一条事件项(失效不能套娃)
5. 事件项的 `quote` 必须指向同一 matter 内的文件(不允许跨 matter)
6. comment 不能被失效(comment 不在失效能力范围)

## 四、可见性规则

失效文档的可见性按以下三态规则处理,无 AI 特例:

| 角色 | 失效文档可见性 |
|---|---|
| 普通用户(非作者、非 admin) | ❌ 完全隐藏(列表不出现、详情 404) |
| 作者本人 | ✅ 可见,带"已失效"标识 + 恢复入口 |
| Admin | ✅ 审计视图(明确标注失效状态) |

`server/ai/tools.py` 与 `server/mcp/` 按调用者身份走以上三态,**不开特例**。当前阶段 Pivot 没有"系统 AI 大脑",外部 agent 通过 PAT 调 MCP 等同于该用户在调,可见性受用户身份约束。

实现层建议提供单一辅助:

```python
def is_visible_to(item, user) -> bool:
    if not item.get("invalidated"):
        return True
    if user.is_admin:
        return True
    return item.get("creator") == user.pinyin
```

所有 GET / SSE / 搜索 / AI tool 路径在 yield item 之前过这一道。

## 五、衍生行为

### 5.1 实时推送

- 失效 / 恢复事件触发 SSE 推送,新主题 `matter.invalidated` / `matter.restored`
- `MatterEventsProvider` 收到后刷新对应 matter 的时间线视图

### 5.2 通知

- 失效 / 恢复事件触发飞书卡片广播(同等于新文件的通知级别)
- 卡片需要清晰展示"作者已撤回某条内容",让团队知晓

### 5.3 引用阻断

- 已失效文件不能被新的 `quote` / `refer` 引用
- 校验器在文件创建路径阻断(不在写盘后才发现)

### 5.4 用户私有状态

- 已失效文件的 `read_state` / `favorites` / `drafts` 引用保留(历史不动)
- 但 unread 计数减 1(不再贡献"待读")

### 5.5 状态机解耦

- 失效不触发任何 `status_change`
- 即使被失效的是 result 文件且曾把 matter 推到 finished,matter 状态依然 finished
- 这是有意为之:保持 timeline 与状态机的两条独立事实链

## 六、v2 延展(本期不做)

| 议题 | v2 方向 |
|---|---|
| 离职员工兜底失效 | admin override 路径,写入时 `invalidated_by ≠ creator`。当前 schema 已能容纳(`invalidated_by` 字段已存在) |
| verification 链联动失效 | 事件项加 `cascade: true` 字段(额外可选字段)。当前 schema 易扩展 |
| 系统 AI 大脑扫描 | 待该能力上线时再设计权限模型;现阶段不与现有 AI tool 混淆 |

## 七、本期不在范围

- comment 失效能力(明确剥离)
- 失效后的物理擦除(产品决策上不引入此层 tension)
- 失效操作的撤销窗口(操作即生效,靠"恢复事件"实现"反悔")

## 八、涉及代码改动点

| 模块 | 改动 |
|---|---|
| `server/matter_index.py` | `_normalize_item` canonical key 顺序扩展;新增 `_reverse_write_invalidations` |
| `server/matter_validator.py` | 6 条新校验规则 |
| `server/doc_types.py` | 新增 `VALID_INVALIDATION_REASONS` 枚举 |
| `server/api/matters.py` | 新 endpoint 处理失效/恢复事件;读路径加可见性过滤 |
| `server/api/matters_events.py` | 新 SSE 主题 `matter.invalidated` / `matter.restored` |
| `server/notify.py` | 新飞书卡片模板 |
| `server/ai/tools.py` + `server/mcp/` | 可见性过滤接入 `is_visible_to` |
| `web/src/pages/MatterDetailPane.tsx` | 时间线渲染分支(事件项 vs 文件项)+ 已失效徽标 |
| `web/src/components/matter/` | 新失效/恢复操作 UI(按钮 + 理由选择 dialog) |
| `AI-docs/pivot-product.md` | 同步更新 schema 规范(per CLAUDE.md 约束) |
| `AI-docs/pivot-interface.md` | 同步更新 API 文档(per CLAUDE.md 约束) |
