---
type: think
author: dengke
created: '2026-04-27T00:48:43+08:00'
index_state: indexed
---
# 需求：AI 调度台与事项提醒机制

## 背景

Pivot 的核心对象是 matter。matter 的推进不是通过传统 task/subtask 树完成，而是通过时间线上的文件流完成，包括 think、act、verify、result、insight。

当 matter 数量变多后，用户不应该每天手动翻看所有 matter，系统应该主动发现哪些事情需要被推动、哪些行动需要验收、哪些事项已经逾期、哪些暂停事项应该重新检查。

这个能力不应该依赖 AI 直接阅读所有正文。第一步应该由 scheduler code 读取结构化 index，先发现确定性问题，再把候选提醒交给 scheduler AI 做进一步判断和表达。

## 目标

建设一个 AI 调度台，用来展示系统自动扫描出的事项提醒。

它要回答三个问题：

1. 我今天需要处理什么？
2. 哪些 matter 需要某个人采取行动？
3. 系统建议下一步创建什么文件来推动 matter 继续前进？

调度台不是新的任务系统，也不是新的事实来源。真正改变 matter 状态的，仍然只能是创建新的 matter 文件，并更新对应 index。

## 需求描述

调度台由两层组成。

第一层是 scheduler code。

scheduler code 负责读取所有 matter 的 index，根据结构化字段发现候选提醒，例如：

- act 已经过了 due_at，但没有 verify
- matter 长时间没有新文件
- paused matter 到了 next_check_at，需要复查
- act 被 verify 为 failed，但之后没有新的 think 或 act
- matter 已经有较充分的 verify，可能可以创建 result
- finished 或 cancelled 的 matter 长时间没有 insight
- result 缺少 verify 支撑，存在流程风险

第二层是 scheduler AI。

scheduler AI 不直接扫描全量文档，而是读取 scheduler code 生成的候选包。它负责：

- 把候选问题整理成人能理解的提醒
- 判断提醒的重要程度
- 给出建议下一步动作
- 判断应该找谁处理
- 在必要时建议创建 think、act、verify、result 或 insight

AI 的输出是建议，不直接改变 matter。

## 与我相关的判定规则

调度台首先应该展示“与我相关”的提醒，而不是全公司所有提醒。

一条提醒与我相关，可以由以下规则判定：

1. 我是相关文件的 owner。
2. 我是相关文件的 creator。
3. 我被相关文件的 comments mention。
4. 我是某个 act 的 owner，而这个 act 需要 verify。
5. 我是某个 verify 的 owner，而它对应的判断还没有完成。
6. 我创建或负责的 matter 长时间没有进展。
7. 我负责的 matter 到了 due_at 或 next_check_at。
8. 我所在团队的 matter 出现高优先级风险，并且我具有团队负责人视角。

第一版可以先实现个人视图，之后再扩展团队视图。

个人视图回答：“我现在应该处理什么？”

团队视图回答：“哪些人负责的 matter 卡住了，哪些事情需要我推动？”

## 未读提示规则

提醒卡片应该有自己的状态，但这个状态只属于调度台，不属于 matter。

第一版只需要四种状态：

- new：新提醒，还没有看过
- read：用户看过，但问题没有解决
- snoozed：用户延后提醒
- resolved：提醒已经被后续 matter 文件解决

用户点击提醒卡片进入 matter，但没有创建任何文件时：

- 卡片从 new 变成 read
- 卡片继续保留有效
- 它不能因为被看过就消失

只有当用户创建了正确的后续文件，并且 scheduler code 下次扫描发现触发条件已经不存在时，这张提醒卡片才进入 resolved。

例如：

- 原提醒是“003_act 需要 verify”
- 用户创建了覆盖该 act 的 verify
- scheduler code 下次扫描发现该 act 已经被验证
- 提醒进入 resolved

暂时不建议第一版加入 dismissed 状态。

原因是 dismissed 容易制造假完成。用户如果认为提醒不对，应该通过 think 文件说明原因，或者通过 snooze 延后检查。

## 列表筛选行为

调度台应该表现为 Attention Inbox，而不是传统 todo list。

第一版列表可以按以下分组展示：

- 今天必须处理
- 已经逾期
- 等待验收
- 长时间无进展
- 暂停事项到期复查
- 可以考虑结束
- 流程风险
- 已处理

每一张提醒卡片代表一个需要注意的调度结果。

提醒卡片需要包含：

- matter 名称
- 提醒类型
- 优先级
- 责任人
- 触发原因
- 相关证据文件
- 建议下一步动作
- 可执行操作

示例结构：

    type: needs_verify
    priority: high
    matter: 登录链路改造
    owner: liuyu
    reason: 003_act 已经超过 due_at，但还没有对应 verify
    evidence:
      - discussions/auth-redesign/003_liuyu_act_xxx.md
    suggested_action: 请 liuyu 或相关验收人创建 verify，判断该行动是 passed、failed 还是 cancelled
    available_actions:
      - 创建 verify
      - 创建 think
      - 创建 result
      - 提醒 owner
      - 延后提醒

列表筛选第一版可以支持：

- 只看我的
- 看团队的
- 看全部
- 只看未处理
- 只看已逾期
- 只看等待验收
- 只看流程风险
- 只看已处理

这里的筛选只影响调度台展示，不改变 matter。

## 进入 matter 后的展示

用户点击提醒卡片后，不应该直接完成提醒。

点击后应该发生三件事：

1. 打开对应 matter 详情页。
2. 跳转到相关文件位置。
3. 展示这条提醒为什么出现，以及建议下一步做什么。

例如：

    为什么提醒你：

    003_liuyu_act_xxx.md 已经超过 due_at，并且当前没有任何 verify 文件覆盖它。

    建议下一步：

    请 owner 或对应验收人创建 verify，判断该 act 是 passed、failed 还是 cancelled。

进入 matter 后，系统应该提供最短路径操作：

- 创建 verify
- 创建 think
- 创建 result
- 提醒 owner
- 延后提醒

其中：

- 创建文件会进入正常的 matter 文件创建流程。
- 提醒 owner 只发送通知，不改变 matter。
- 延后提醒只改变调度台状态，不改变 matter。
- 如果用户认为 matter 应该暂停、恢复、结束或取消，也应该通过创建对应文件来表达，而不是直接点击按钮修改状态。

## 补录机制

现实工作中，很多事情可能已经在线下发生，但还没有被记录进 Pivot。

例如：

- 研发已经演示过功能，但没有创建 verify。
- 团队已经决定暂停某个 matter，但没有创建 think。
- 某个 matter 实际已经完成，但没有创建 result。
- 某个 act 实际已经取消，但没有留下验证记录。

调度台发现这些缺口后，不应该直接替用户修改状态，而应该引导用户补录正确的文件。

补录仍然使用正常文件类型：

- 用 think 补录判断、暂停、恢复、原因说明。
- 用 verify 补录对 act 的验收判断。
- 用 result 补录 matter 的最终完成或取消。
- 用 insight 补录结束后的复盘。

补录时可以由 AI 协助整理内容。

流程可以是：

1. 用户点击提醒。
2. 系统打开对应 matter 和相关文件。
3. 用户说明实际发生了什么。
4. AI 整理出建议创建的文件内容和结构化字段。
5. 用户确认。
6. API 同步写入 MD 文件和 index。

补录不应该绕过文件流。所有重要事实最终都应该落到 matter timeline 里。

## 暂不包含

第一版暂不包含：

- 多级任务树
- task/subtask 管理
- 复杂权限控制
- 自动修改 matter 状态
- AI 全量阅读所有正文
- dismissed / archived 等复杂提醒状态
- 提醒之间的依赖关系管理
- 自动替用户创建文件
- 自动替用户判断 matter 是否完成

## 期望结果

用户每天打开 Pivot 后，不需要自己翻所有 matter。

系统会主动告诉他：

- 哪些事情今天必须处理
- 哪些事情已经逾期
- 哪些行动需要验收
- 哪些暂停事项应该复查
- 哪些 matter 可能已经可以结束
- 哪些流程存在风险
- 哪些事情需要补录关键文件

用户点击提醒后，会被带到对应 matter 的现场，并通过最短路径创建正确的下一篇文件。

这样 Pivot 仍然保持 matter + timeline + file flow 的简洁模型，同时具备主动推动组织事项前进的能力。
