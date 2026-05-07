---
type: think
author: dengke
created: '2026-04-28T13:03:22+08:00'
index_state: indexed
---
# 新需求：为 Matter 增加 Owner 责任人机制

## 背景

Matter 代表企业中的一个具体事项。一个事项可以包含讨论、行动、验证和结果，但在实际推进中，还需要有一个明确的人持续盯住它，推动它进入下一个明确状态。

因此，Matter 不应只是一个讨论容器，也应具备当前责任归属。每个 Matter 在任一时刻都应有一个当前 owner，用来表示“这件事现在由谁负责推进”。

## 目标

为每个 Matter 增加 owner 字段，并在列表、详情页和 timeline 中完整呈现 owner 的当前状态与变更历史。

Owner 的设计目标是：

- 让事项责任归属在入口层可见；
- 支撑“我负责的 Matter”“与我相关”等后续筛选能力；
- 让 scheduler code / scheduler AI 能低成本判断事项是否无人负责、长期无人推进或责任人异常；
- 通过 timeline 保留 owner 变更历史，保证责任转移可追溯。

## 需求描述

### 1. Matter 应增加当前 owner 字段

每个 Matter 应有一个当前 owner。Owner 表示该 Matter 当前由谁负责持续推进，而不是表示该人必须亲自完成 Matter 中的所有行动。

Matter owner 和 act owner 应区分：

- Matter owner：事项当前责任人，负责推动 Matter 进入下一个明确状态；
- Act owner：某个具体行动的执行人。

一个 Matter 可以由 A 负责推进，其中某个 act 由 B 执行。

### 2. Owner 应写入 Matter index

Owner 应作为 Matter 当前状态的一部分写入 index header，方便列表、筛选、调度扫描和 AI 分析低成本读取。

建议字段包括：

```yaml
owner: ou_xxx
owner_display: 张三
```

历史 Matter 迁移时，如果暂时无法确定 owner，可以允许为空或显示为“未分配”，但 UI 应能提示该 Matter 当前缺少 owner。

### 3. 新建 Matter 时默认 owner 为创建者

新建 Matter 时，如果用户没有主动指定 owner，系统默认将创建者设置为 owner。

后续可以通过详情页操作更换 owner。

### 4. Matter 列表应展示 owner

左侧 Matter 列表中，每个 Matter item 应展示当前 owner 的头像和姓名。

展示要求：

- owner 显示为“头像 + 姓名”，不要只显示名字；
- owner 信息应在列表中稳定占位，避免不同名字长度导致布局跳动；
- 没有 owner 时显示“未分配”，不要留空；
- 列表空间紧张时，可以优先保留头像，姓名截断，但完整姓名应能在 hover 或详情中看到；
- owner 不应抢过 title 的视觉优先级，title 仍是主信息，owner 是责任归属信息。

### 5. Matter 详情卡片应展示 owner 并支持转交

Matter 详情页顶部的 Matter 卡片中，应展示当前 owner 的头像和姓名。

同时，卡片中应提供 owner 变更入口，例如：

- 转交负责人
- 更换 Owner

第一版建议使用“转交负责人”，因为它表达的是责任移交，而不是随意修改字段。

点击后打开轻量弹窗或 popover，包含：

- 当前 owner；
- 新 owner 选择器，复用联系人搜索；
- 变更原因；
- 确认按钮。

变更原因应为必填。

### 6. Owner 变更应作为独立 timeline 事件

Owner 变更不能只体现在 index header 或详情卡片中，必须作为一条独立 timeline 事件展示在时间轴上。

事件应记录：

- 操作人；
- 原 owner；
- 新 owner；
- 变更时间；
- 变更原因。

建议事件类型为：

```yaml
type: owner_change
actor: ou_operator
from_owner: ou_old
to_owner: ou_new
reason: 后续执行由李四负责推进
created_at: ...
```

Timeline 中可以展示为：

```text
邓柯 将 Owner 从 张三 转交给 李四
原因：后续执行由李四负责推进
```

这类事件可以在视觉上比 think / act / verify / result 更轻，但必须可见、可追溯、可被 AI 读取。

### 7. Owner 变更与状态变更的关系

Owner 变更本身不自动改变 Matter 状态。

例如：

```text
status = planning
owner = 张三
```

变更 owner 后可以仍然是：

```text
status = planning
owner = 李四
```

但在“正式转交执行”等场景下，系统应允许一次操作同时完成 owner 变更和状态变更。例如：

```text
owner: 张三 -> 李四
status: planning -> executing
```

这时 timeline 上应明确记录两件事，不要让 owner change 藏在 status change 的备注里。

### 8. Owner 变更采用开放协作模型

第一版不对 owner 变更做额外权限限制。

任意已登录成员都可以发起 owner 变更，但必须填写变更原因。系统不引入审批流，也不尝试用复杂权限限制协作行为。

如果未来出现恶意或随意修改，应优先通过团队纪律和事后追责解决，而不是在第一版用软件限制一切。

系统需要保证的是：所有 owner 变更都进入 timeline，完整记录操作人、变更前后 owner、时间和原因。

## 验收标准

- 新建 Matter 时，默认 owner 为创建者；
- Matter index header 中包含当前 owner 信息；
- 左侧 Matter 列表展示 owner 头像和姓名；
- Matter 详情页顶部卡片展示 owner 头像和姓名；
- 详情页可发起 owner 变更，并要求填写变更原因；
- Owner 变更后，index header 中的当前 owner 被更新；
- Owner 变更后，timeline 中新增独立 owner_change 事件；
- Owner 变更事件记录操作人、原 owner、新 owner、时间和原因；
- Owner 变更本身不自动改变 Matter 状态；
- 系统允许 owner 变更和状态变更在同一次操作中发生，并在 timeline 中清楚展示；
- 未分配 owner 的历史 Matter 在 UI 中显示为“未分配”，不要留空。
