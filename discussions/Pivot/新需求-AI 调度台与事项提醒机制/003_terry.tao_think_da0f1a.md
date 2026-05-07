---
type: think
author: terry.tao
created: '2026-04-29T10:35:49+08:00'
index_state: indexed
---
# Pivot 5月 MVP 产品功能全景文档

**版本：** v0.2 **状态：** 待评审 **作者：** Terry（产品经理） **日期：** 2026年4月29日

**变更说明：** 基于 v0.1 讨论反馈，完善两处落地前的细节，感谢@张菠补充：

1. **状态机增补 `paused → cancelled` 路径**，由 `think` 触发，避免"先恢复 executing 再创建 result cancelled"的语义绕弯。
2. **为时间型字段补全写入时机与默认规则**：`due_at`（matter / act）、`next_check_at`、`outcome` 各自的填写时点、默认值、可编辑性、AI 建议机制全部明确。统一约定 **"AI 可建议候选日期，最终必须由用户确认后由程序写入 index"** 的写入边界。

同步将 v0.1 开放问题 Q-03 / Q-04（due_at 与 next_check_at 由谁填写）的结论吸收进正文，从开放问题清单中移除。新增两条 v0.2 改动衍生的开放问题（Q-05 / Q-06）。

------

## 一、MVP 交付边界与核心目标

### 一句话定义成功

5月底上线时，用户能围绕 matter 完成 **"创建文件 → 执行推进 → 验收判断 → 调度台主动发现待办 → 补录闭环"** 的完整工作流。系统第一次具备主动推动事项前进的能力，而不只是被动记录。

### 核心用户画像

**主要用户：** 5-15 人技术团队的一线成员（工程师、产品、设计），每人同时负责 3-8 个并行 matter。

**高频场景：**

- 每天打开 Pivot，快速知道"今天我该处理什么"
- 给一件事创建 act 并指定 owner，推动执行
- 对已完成的 act 做验收，判断 passed / failed / cancelled
- 发现某件事卡住了，补录一篇 think 说明暂停原因
- 一件事全部完成后，创建 result 正式收口
- 一件事在 paused 期间被判断不再推进，直接通过 think 记录终止

### 本期包含

- Matter 状态机完整闭环（planning → executing → paused → finished / cancelled → reviewed），含 v0.2 新增的 **`paused → cancelled` 直接路径**
- 五种文档类型的创建与展示（think / act / verify / result / insight）
- Timeline-first 的 index 结构落地
- Matter 详情页改造为时间线视图
- 文件卡片交互（基于已有文件创建新文件，自动写入 quote）
- 时间型字段（due_at / next_check_at / outcome）的写入时机与默认规则落地
- AI 调度台 MVP（scheduler code + scheduler AI）
- 补录引导流程
- 个人视图的提醒列表

### 明确不包含

- 团队视图（二期）
- 外部系统集成（飞书 / Slack / GitHub 通知推送）
- 跨 matter 依赖关系管理
- 多级 task / subtask 树
- 复杂权限与审批流
- AI 自动修改 matter 状态或自动创建文件
- AI 直接写入 due_at / next_check_at（候选必须经用户确认）
- AI 全量阅读所有文件正文
- 知识图谱与组织记忆

------

## 二、产品架构与数据流总览

### 现有能力盘点（复用，不动）

以下能力已在线上运行，本期直接复用，不做结构性改动：

- **Git 存储层**：所有 Markdown 文件以 Git 为单一事实源，支持版本追溯与回滚。
- **Markdown 文件创建与展示**：用户可在 matter 下创建 Markdown 文件，系统渲染展示。
- **Discussion 话题体系**：现有话题讨论功能，本期将其语义从"讨论帖"重新解释为 matter 在 planning 阶段的基础设施。
- **基础 AI 能力**：AI 可理解当前 matter 上下文并生成候选摘要。
- **用户身份体系**：已有基础用户身份，可识别 creator。

### 本期新增模块

本期新增三个层级的能力，下图展示完整数据流转：

```
用户操作                  数据层                    智能层                   展示层
────────────────────────────────────────────────────────────────────────────────

创建/编辑文件  ──────►  Git 写入 MD 文件
                              │
                              ▼
                        程序更新 Index  ◄────  AI 提供候选摘要 / 候选日期
                              │                 （用户确认后写入）
                              │
                              ├──────────────►  Matter 详情页
                              │                 （时间线视图）
                              │
                              ▼
                        Scheduler Code
                        （结构化扫描 Index）
                              │
                              ▼
                        Scheduler AI
                        （整理 + 判断优先级）
                              │
                              ▼
                        调度台提醒列表  ──────►  用户处理
                              │                     │
                              │                     ▼
                              │               创建新文件（闭环）
                              │                     │
                              ◄─────────────────────┘
                        下次扫描确认 resolved
```

### 模块间依赖关系

开发顺序上，模块之间存在明确的先后依赖：

```
第一步：Index 字段补齐 + 字段写入时机规则（due_at / updated_at / next_check_at / outcome）
  │
  ├──► 第二步-A：Matter 详情页改造 + 文件卡片交互 + 状态迁移规则
  │
  └──► 第二步-B：Scheduler Code 扫描规则实现
                    │
                    ▼
              第三步：Scheduler AI 整理层
                    │
                    ▼
              第四步：调度台 UI + 补录引导
```

第二步 A 和 B 可以并行，但都依赖第一步 index 字段到位。第三步依赖第二步 B 的候选提醒输出。第四步依赖第三步的完整提醒数据。

------

## 三、功能模块清单

### 3.1 基础数据与索引层（已有补齐）

当前 index 采用 timeline-first 结构，顶层头部 + timeline 文件项。以下为本期需要补齐的字段，已实现字段不再赘述。

**需补齐字段：**

| 字段            | 位置          | 类型                       | 说明                                          |
| :-------------- | :------------ | :------------------------- | :-------------------------------------------- |
| `due_at`        | matter 头部   | datetime / null            | matter 级别的截止时间                         |
| `due_at`        | act 文件项    | datetime / null            | 单个 act 的截止时间                           |
| `updated_at`    | matter 头部   | datetime                   | matter 最后活跃时间（有新文件或新评论时更新） |
| `next_check_at` | matter 头部   | datetime / null            | 仅 paused 状态下生效，标记下次复查时间        |
| `outcome`       | result 文件项 | enum: finished / cancelled | result 文件专属，标记事项最终结果             |

**写入规则不变：** 程序定义 schema、校验约束、执行写入。AI 只提供候选摘要和建议，不直接写 index。

**已有字段确认复用（不做改动）：** `file`、`created_at`、`creator`、`owner`、`type`、`summary`、`quote`、`refer`、`verifications`、`comments`、`status_change`、`current_status`。

#### 3.1.1 字段写入时机与默认规则（v0.2 新增）

时间型字段的写入边界必须在落地前明确，否则前端无从判断"该不该提示填、什么时候填、能不能编辑"。本节统一约定如下。

**总原则：**

- 所有时间型字段（due_at、next_check_at）由用户填写或确认，程序负责写入
- AI 可以基于上下文提供候选日期，但 **AI 不直接写入 index**——候选必须经用户确认后才落地
- 这与产品设计文档中"程序定义 index schema、AI 提供候选输入、程序负责写入"的边界完全一致

**字段级规则：**

| 字段                   | 写入时机                                | 默认值          | 是否可编辑                        | AI 建议机制                                                  |
| ---------------------- | --------------------------------------- | --------------- | --------------------------------- | ------------------------------------------------------------ |
| `matter.due_at`        | 创建 matter 时**可选**填写              | null            | 后续在 matter 详情页可编辑        | AI 可基于事项标题与已有 think 内容给出候选日期；用户确认后写入 |
| `matter.updated_at`    | 程序自动维护                            | matter 创建时间 | 用户不可直接编辑                  | 不需要                                                       |
| `matter.next_check_at` | 创建触发 paused 的 think 时**必须**填写 | 无默认值，必填  | paused 状态期间可编辑             | AI 基于 think 中描述的暂停原因给出候选日期（如"等待外部 API 稳定" → 建议两周后）；用户确认后写入 |
| `act.due_at`           | 创建 act 时**可选**填写                 | null            | 后续可在 act 文件元数据中编辑     | AI 可基于 quote 内容与 act summary 给出候选日期；用户确认后写入 |
| `result.outcome`       | 创建 result 时**必填**，二选一          | 无默认值        | 不可后续修改（result 是终态文件） | 不适用（用户主观判断）                                       |

**配套行为：**

- `matter.due_at` 为 null 时，调度台的 R-01（act 逾期）和逾期相关规则不对该 matter 触发提醒
- `act.due_at` 为 null 时，调度台 R-01 不对该 act 触发 needs_verify
- `matter.next_check_at` 在 matter 从 paused 恢复或终止（→ planning / executing / cancelled）时由程序自动清空
- 创建触发 paused 的 think 时，前端表单必须包含 `next_check_at` 输入项；未填写不允许提交（区别于"目标为 cancelled 的 think"，详见 3.2.3）
- AI 候选日期以"建议值 + 推断依据"两段呈现给用户，例如：
  - 建议值：2026-05-15
  - 依据：你提到等待外部 API 稳定，外部团队最近一次更新预期是两周后
- 用户对 AI 建议的接受动作必须显式（点击"使用此日期"或手动修改后保存），不存在"沉默接受"

### 3.2 核心交互层（增强）

本节覆盖用户在 matter 内部的核心操作交互，重点描述增量改动。

#### 3.2.1 Matter 详情页改造

现有右侧详情页从"帖子列表"改造为时间线视图：

- 详情页顶部展示一条时间轴，按时间顺序排列该 matter 所有文件
- 时间轴下方为完整文件流，按时间从上往下展示
- 每个文件卡片默认折叠，点击展开，沿用现有展开体验
- 页面级动作区（靠近时间轴位置）放置"生成 Result"入口

#### 3.2.2 文件卡片交互（核心增量）

每个已有文件卡片上提供三个快捷入口：

- **基于此新增 think**
- **基于此新增 act**
- **基于此新增 verify**

用户点击后，系统自动将当前文件写入新文件的 `quote`，实现"从已有文件继续长出新文件"的交互心智。

**Result 是特例：** 不从文件卡片发起，而是从详情页顶部页面级入口发起。点击后系统明确提示"创建 result 代表正式结束该 matter"，进入 result 创建流程。

**状态允许的文件类型约束：**

| 当前状态  | 允许创建                      | 不允许                          |
| :-------- | :---------------------------- | :------------------------------ |
| planning  | think / act / verify          | result / insight                |
| executing | think / act / verify / result | insight                         |
| paused    | think                         | act / verify / result / insight |
| finished  | insight                       | think / act / verify / result   |
| cancelled | insight                       | think / act / verify / result   |
| reviewed  | 不再新增任何文件              | —                               |

界面上应当根据当前状态，自动隐藏不允许创建的文件类型入口。

> 注：paused 状态下虽然只允许 think 文件类型，但该 think 可以承担三种不同的判断作用——继续作为分析记录、触发恢复（→ planning / executing）、或触发终止（→ cancelled）。具体的状态迁移触发规则见 3.2.3。

#### 3.2.3 状态迁移规则（v0.2 新增）

为了让 MVP 阶段的状态机闭环可走通，本节明确所有状态迁移的触发文件类型与必备字段。

**所有状态迁移路径：**

| 迁移路径                 | 触发文件  | 必备字段 / 内容                                              |
| ------------------------ | --------- | ------------------------------------------------------------ |
| (创建 matter) → planning | —         | matter 创建即进入 planning                                   |
| planning → executing     | act       | act 文件本身（无额外必备字段，act.owner 必填）               |
| planning → paused        | think     | think 须说明：暂停原因、当前阻塞、恢复条件；同时**必填** `next_check_at` |
| executing → paused       | think     | 同上                                                         |
| paused → planning        | think     | think 须说明：恢复原因、阻塞条件变化、为什么回到 planning    |
| paused → executing       | think     | 同上，回到 executing                                         |
| **paused → cancelled**   | **think** | **(v0.2 新增)** think 须说明：为什么确认不再推进、之前停止时的状态、是否已与相关方达成一致 |
| executing → finished     | result    | `outcome = finished`                                         |
| executing → cancelled    | result    | `outcome = cancelled`                                        |
| finished → reviewed      | insight   | insight 触发，标志生命周期收口                               |
| cancelled → reviewed     | insight   | 同上                                                         |

##### 关于 `paused → cancelled` 的特别说明

这是 MVP v0.2 相对产品设计文档的扩展。补充原因：

如果一个 matter 已经处于 paused 状态，且团队判断其不再需要继续推进，强制要求用户先恢复到 executing 再创建 `result cancelled`，在语义上是绕的——事项实际从未恢复过执行，所谓"恢复"只是为了让 result 能被合法创建。

因此 v0.2 允许：

- 在 paused 状态下，由一篇 think 直接触发 `paused → cancelled`
- 该 think 是对"为什么这件事不再推进"的判断记录，而不是执行结果
- UI 在创建该 think 时给出二次确认提示："这将正式取消该 matter，之后不可恢复执行。是否继续？"

**边界：**

- `paused → cancelled` 是 think 文件可以触发的**唯一**终态迁移；其他终态迁移（`executing → finished` / `executing → cancelled`）仍由 result 触发
- `paused → finished` **不在** MVP 支持范围：finished 表示"已完成执行闭环"，paused 状态下并未发生执行；如果一个 paused matter 应当被视为完成，仍要求先恢复到 executing 再创建 `result finished`
- 创建该 think 时**不要求**填写 `next_check_at`（因为不再恢复）；前端表单根据用户选择的目标迁移动态调整字段必填性

##### Paused 下 think 的目标迁移选择

由于 paused 状态的 think 可以承担多种判断作用，前端创建表单需要让用户在提交前显式选择目标：

| 目标选项                | 触发的状态迁移         | 必备字段                                 | 提交时是否二次确认 |
| ----------------------- | ---------------------- | ---------------------------------------- | ------------------ |
| 保留 paused（继续分析） | 不触发迁移             | 无                                       | 否                 |
| 恢复到 planning         | paused → planning      | 恢复说明                                 | 否                 |
| 恢复到 executing        | paused → executing     | 恢复说明                                 | 否                 |
| 终止该 matter           | **paused → cancelled** | 终止说明（不再推进的原因、是否达成一致） | **是**             |

##### 程序约束

- 程序根据 `status_change` 字段的 `from / to` 写入状态变化
- 用户尝试创建当前状态不允许的文件类型时，前端禁用对应入口（如 paused 下不显示"新增 act / verify / result / insight"）
- AI 不直接修改 matter 状态。所有状态迁移必须经过"用户创建文件 → 程序校验 → 程序写入 index"的完整路径
- 表单提交时，程序对 `status_change` 的合法性做兜底校验：不在 3.2.3 表中的迁移路径一律拒绝

#### 3.2.4 上下文说明面板（新增）

从调度台点击提醒卡片进入 matter 后，在详情页顶部或右侧浮层展示一个上下文说明面板，包含：

- 为什么提醒你（触发原因，一句话）
- 相关证据文件（可点击跳转到时间线对应位置）
- 建议下一步动作
- 快捷操作按钮：创建 verify / 创建 think / 创建 result / 提醒 owner / 延后提醒

用户不从调度台进入、而是直接打开 matter 时，不展示此面板。

#### 3.2.5 补录引导（新增）

当调度台检测到可能存在线下已发生但未记录的事项（如 act 长期无 verify、matter 实际已完成但没有 result），上下文面板额外提示"这件事可能已经在线下完成，建议补录"。

补录流程：

1. 用户在面板内简要说明实际发生了什么（文字输入）
2. AI 整理出建议创建的文件内容和结构化字段（含可能的状态迁移目标与必备字段建议值，如 next_check_at 候选日期）
3. 用户确认或修改
4. 确认后走正常文件创建流程，写入 MD + 更新 index

补录不绕过文件流，所有事实仍落在 matter timeline 里。补录中如果涉及时间型字段（如 next_check_at），仍遵守 3.1.1 的"用户确认后写入"原则——AI 整理出的候选值需要用户在最终确认页显式接受。

### 3.3 智能调度层（核心增量）

调度台是本期最核心的新增模块，目标是让系统第一次具备主动感知和推动事项的能力。

#### 3.3.1 两层架构

**Scheduler Code（第一层）**

读取所有 matter 的 index，基于结构化字段执行确定性规则扫描，输出候选提醒包。不读正文，不做主观判断，只做发现。

**Scheduler AI（第二层）**

接收候选提醒包，将其整理为用户可理解的自然语言提醒，判断优先级（high / medium / low），给出建议下一步动作和建议责任人。不直接修改 matter。

#### 3.3.2 扫描规则（MVP 七条）

| 编号 | 触发条件                                        | 提醒类型          | 默认阈值 |
| :--- | :---------------------------------------------- | :---------------- | :------- |
| R-01 | act 超过 due_at，无 verify 覆盖                 | needs_verify      | —        |
| R-02 | matter 超过 N 天无新文件                        | no_progress       | 7 天     |
| R-03 | paused matter 到达 next_check_at                | paused_check      | —        |
| R-04 | act 被 verify 判为 failed，无后续 think 或 act  | failed_unresolved | —        |
| R-05 | matter 所有 act 均有 verify 且无 failed         | ready_to_close    | —        |
| R-06 | finished / cancelled matter 超过 N 天无 insight | needs_insight     | 14 天    |
| R-07 | result 不存在任何 verify 前置                   | process_risk      | —        |

关于 R-01 的前置条件：仅当 act 有 `due_at` 字段（非 null）时才参与扫描。`due_at` 为 null 的 act 不触发逾期提醒。

关于 R-03 的前置条件：仅当 matter 处于 paused 状态且 `next_check_at` 非 null 时才参与扫描。MVP 中 `next_check_at` 在进入 paused 时强制填写（详见 3.1.1），因此 paused 状态的 matter 默认都会被该规则覆盖。

关于 R-05 的"充分 verify 支撑"判定：MVP 采用最严格的简单规则——**该 matter 下所有 act 均已被某篇 verify 覆盖，且最新一轮 verify 中不存在 failed 判断。** 满足此条件即触发 ready_to_close 提醒。此规则在上线后根据误报率反馈调整。

#### 3.3.3 "与我相关"的判定规则

调度台默认展示个人视图。一条提醒与当前用户相关的判定条件（满足任一即匹配）：

- 我是相关文件的 owner
- 我是相关文件的 creator
- 我在相关文件 comments 中被 mention
- 我是某个 act 的 owner，且该 act 需要 verify
- 我创建或负责的 matter 长时间无进展
- 我负责的 matter 到了 due_at 或 next_check_at

#### 3.3.4 提醒卡片状态

提醒卡片有独立生命周期，与 matter 状态完全解耦。

| 状态     | 含义                 | 触发方式                                  |
| :------- | :------------------- | :---------------------------------------- |
| new      | 新提醒，未看过       | scheduler 扫描生成                        |
| read     | 已看过，问题未解决   | 用户进入对应 matter 但未创建文件          |
| snoozed  | 用户主动延后         | 点击"延后提醒"，选择明天 / 3天后 / 一周后 |
| resolved | 问题已被后续文件解决 | scheduler 下次扫描确认触发条件不再存在    |

**关键原则：** 卡片看过不等于解决。只有 scheduler 确认触发条件消失，才进入 resolved。MVP 不加入 dismissed 状态，避免假完成。

#### 3.3.5 提醒列表分组

调度台列表按以下分组展示，组内按优先级排序：

- 今天必须处理
- 已经逾期
- 等待验收
- 长时间无进展
- 暂停事项到期复查
- 可以考虑结束
- 流程风险
- 已处理（折叠展示）

#### 3.3.6 列表筛选（MVP）

- 只看我的 / 看全部
- 只看未处理 / 只看已逾期 / 只看等待验收 / 只看流程风险 / 只看已处理

筛选只影响调度台展示，不改变 matter。

#### 3.3.7 提醒卡片结构

```
matter 名称
提醒类型（needs_verify / no_progress / 等）
优先级（high / medium / low）
责任人
触发原因（一句话）
相关证据文件（可跳转）
建议下一步动作
可执行操作按钮
```

### 3.4 辅助与触达层

本期仅做最轻量的触达能力，不建设完整通知系统。

**调度台角标：** 主导航中调度台入口显示未读数字角标（new 状态的提醒数量），让用户无需点入即可知道有新提醒。

**身份识别（复用已有）：** 基于现有用户身份系统识别 owner / creator / mention 关系，不新增角色体系。

**站内通知（仅 MVP 最小集）：** "提醒 owner"操作触发一条站内简单通知，通知内容为提醒卡片摘要 + 跳转链接。不对接外部渠道。

------

## 四、关键用户链路

### 场景一：一件事从 planning 推进到 executing

**用户：** 工程师 liuyu

1. liuyu 创建一个新 matter "登录链路改造"，可选填写 `due_at`（本次留空），系统自动进入 `planning` 状态。
2. liuyu 创建第一篇 think，分析当前登录方案的问题。
3. dengke 基于 liuyu 的 think 创建第二篇 think，补充安全相关考虑。系统自动将 liuyu 的 think 写入 quote。
4. liuyu 基于两篇 think 创建一篇 act，明确执行方案，owner 设为自己，可选填写 act 的 `due_at`（AI 基于 quote 建议 5月15日，liuyu 确认采用）。系统触发 `planning → executing` 状态迁移，写入 index 的 status_change。

**涉及模块：** Git 写入 → Index 更新（含 due_at 用户确认写入）→ Matter 详情页时间线展示 → 文件卡片 quote 自动填充

### 场景二：调度台发现逾期 act 并驱动验收

**用户：** 工程师 liuyu

1. liuyu 之前创建的 act 设定了 due_at 为 4月25日，现在已过期，但没有人创建 verify。
2. **Scheduler Code** 扫描 index，发现该 act 超过 due_at 且无 verify 覆盖，生成候选提醒。
3. **Scheduler AI** 接收候选包，生成提醒："登录链路改造的 003_act 已超过截止日期，尚未验收。建议 liuyu 或对应验收人创建 verify。"优先级判定为 high。
4. liuyu 打开调度台，看到该提醒卡片（new 状态），点击进入 matter 详情页。
5. 系统跳转到 003_act 位置，上下文面板展示触发原因和建议操作。
6. liuyu 点击"创建 verify"，进入 verify 创建流程，对 003_act 给出 passed 判断。
7. 下次 scheduler 扫描时，确认该 act 已有 verify 覆盖，提醒卡片自动进入 resolved。

**涉及模块：** Index（due_at 字段）→ Scheduler Code → Scheduler AI → 调度台 UI → 上下文面板 → 文件创建流程 → Index 更新 → Scheduler 再次扫描确认

### 场景三：线下已发生的事补录进系统（含 paused 进入）

**用户：** 工程师 dengke

1. dengke 负责的一个 matter 实际上已经在线下讨论后决定暂停，但系统里没有任何记录。
2. 调度台检测到该 matter 7 天无新文件，生成 no_progress 提醒。
3. dengke 点击提醒进入 matter，上下文面板提示"这件事可能已经在线下发生了变化，建议补录"。
4. dengke 在补录输入框输入"上周五和团队讨论后决定暂停，等外部接口 API 稳定后再恢复"。
5. AI 整理出一篇 think 文件的建议内容，包含暂停原因、当前阻塞点、恢复条件，建议触发 `executing → paused` 状态迁移；同时给出 next_check_at 候选日期（建议两周后），并附推断依据。
6. dengke 接受 AI 的内容，但将 next_check_at 修改为三周后；在最终确认页点击提交。
7. 系统写入 think 文件 + 更新 index（状态变为 paused，next_check_at 写入用户确认值）。
8. 下次 scheduler 扫描时，该 matter 已有 paused 状态和 next_check_at，no_progress 提醒进入 resolved。到达 next_check_at 时，会触发新的 paused_check 提醒。

**涉及模块：** Scheduler Code → 调度台 UI → 补录引导 → AI 内容整理（含候选日期）→ 用户确认 → 文件创建 → Index 更新（状态迁移 + next_check_at）

### 场景四：paused 状态下直接终止事项（v0.2 新增）

**用户：** 工程师 dengke

1. dengke 负责的 matter "数据迁移方案"已处于 paused 状态 30 天，原因是依赖的外部接口长期未稳定。
2. 团队周会确认外部接口在可见周期内不会推进，决定终止该 matter。
3. dengke 进入 matter 详情页，在某篇 think 卡片上点击"基于此新增 think"。
4. 在新 think 创建表单中，dengke 在"目标迁移"下拉中选择 **"终止该 matter（→ cancelled）"**。
5. 表单收起 next_check_at 字段（因不再恢复），保留必填的终止说明：暂停期间发生了什么、为什么不再推进、是否已与相关方达成一致。
6. 提交时 UI 弹出二次确认："这将正式取消该 matter，之后不可恢复执行。是否继续？"
7. dengke 确认后，系统写入 think 文件 + 更新 index（current_status: cancelled，status_change: paused → cancelled），并清空 next_check_at。
8. 下次 scheduler 扫描确认该 matter 已 cancelled，原本因 next_check_at 触发的 paused_check 提醒自动 resolved。

**涉及模块：** 文件创建表单（动态字段切换 + 目标迁移选择）→ 二次确认弹窗 → think 写入 → index 更新（status_change paused→cancelled + 清空 next_check_at）→ Scheduler 再次扫描确认

------

## 五、验收标准与上线节奏

### 核心验收标准

| 验收项                | 标准                                                         | 说明                                                         |
| :-------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 状态机闭环            | planning → executing → finished → reviewed 完整链路可走通；paused 分支支持 `→ planning / → executing / → cancelled` 三种出路 | 含 v0.2 新增的 `paused → cancelled` 路径                     |
| 五种文件类型          | think / act / verify / result / insight 均可正常创建、展示、写入 index | 含 quote 自动填充                                            |
| 时间字段写入正确性    | due_at / next_check_at / outcome 按 3.1.1 规则被正确写入或保持 null | 含 next_check_at 在 paused 进入时强制填写、恢复 / 终止时被清空 |
| AI 候选日期不直接落地 | AI 给出的 due_at / next_check_at 候选必须经用户显式确认才能写入 index | 抽查后端日志，确认无 AI 直写记录                             |
| Paused think 表单分流 | paused 状态下创建 think 时，前端按"目标迁移"动态调整必备字段；选择 cancelled 时弹出二次确认 | 含字段联动测试                                               |
| 调度台提醒覆盖率      | 七条扫描规则均可正常触发                                     | 人工构造测试数据验证                                         |
| 提醒误报率            | 试跑期间 snoozed 率 < 30%                                    | 超过 30% 需排查规则                                          |
| 提醒闭环率            | resolved 率 > 40%                                            | 低于 40% 需排查流程断点                                      |
| 操作路径步数          | 从调度台到创建文件 ≤ 3 步                                    | 点击提醒 → 进入 matter → 创建文件                            |
| 补录流程              | 用户输入说明 → AI 整理（含候选日期）→ 用户确认 → 写入，完整可走通 | 含状态迁移场景                                               |

### 上线节奏建议

**第一阶段：Index 字段补齐 + 字段写入规则 + 文件创建交互**

- 补齐 due_at、updated_at、next_check_at、outcome 字段
- 落地 3.1.1 字段写入时机规则（含 AI 候选日期的"用户确认"链路）
- 文件卡片三个快捷入口上线（think / act / verify）
- Result 页面级入口上线
- 状态约束逻辑上线（不同状态下隐藏不允许的文件类型）

**第二阶段：Matter 详情页改造 + 状态迁移规则 + Scheduler Code**

- 详情页从帖子列表改造为时间线视图
- 文件流按时间排序、默认折叠、点击展开
- 落地 3.2.3 完整状态迁移规则，含 `paused → cancelled` 与 paused think 的目标分流表单
- 七条扫描规则在后端落地，输出候选提醒包
- 内部试跑扫描结果，校验误报

**第三阶段：Scheduler AI + 调度台 UI**

- Scheduler AI 整理层接入，输出自然语言提醒 + 优先级 + 建议
- 调度台列表页上线（分组展示 + 筛选 + 角标）
- 提醒卡片四种状态流转上线
- 上下文说明面板上线

**第四阶段：补录引导 + 联调 + 试跑**

- 补录引导流程上线（含 AI 候选日期的用户确认环节）
- 全链路联调
- 内部团队真实使用试跑
- 收集误报、漏报、交互摩擦反馈

**试跑：修复 + 灰度**

- 基于试跑反馈修复高优问题
- 调整扫描规则阈值
- 灰度放量或全量上线

### 灰度策略

MVP 阶段建议先内部团队全量使用一周，重点观察：

- 提醒数量是否合理（每人每天 3-8 条为健康区间，超过 15 条说明规则过宽）
- snoozed 集中在哪类规则上（用于判断哪条规则噪音最大）
- 用户从调度台到创建文件的实际路径是否顺畅
- AI 候选日期的接受率（用户直接采用 / 用户修改后采用 / 用户忽略）的分布

内部试跑通过后，再对外部用户开放。

------

## 六、开放问题（需开发前对齐）

| 编号 | 问题                                                 | 影响范围      | 建议                                                         |
| :--- | :--------------------------------------------------- | :------------ | :----------------------------------------------------------- |
| Q-01 | 调度台卡片状态存在哪里？                             | 数据层        | 建议轻量数据库，不放 Git（视图层状态，非事实层数据）         |
| Q-02 | Scheduler Code 扫描频率？                            | 性能 + 及时性 | 建议首版每小时一次，后续视负载调整                           |
| Q-03 | 补录流程中 AI 生成内容写入由谁调 API？               | 前后端边界    | 建议前端确认后调后端接口统一写入                             |
| Q-04 | 存量 discussion 到 timeline-first index 的数据迁移？ | 上线前置      | 需确认是否迁移存量，还是仅对新建 matter 生效                 |
| Q-05 | `paused → cancelled` 是否需要二人复核？              | 交互设计      | MVP 仅做单人二次确认；二人复核留待企业版二期                 |
| Q-06 | AI 候选日期的接受率如何观测与回流？                  | 质量监测      | 后端记录"AI 建议值 → 用户最终写入值"的差异分布，作为后续提示词调优依据 |

> **v0.1 中的 Q-03（due_at 由谁填写）和 Q-04（next_check_at 由谁填写）已在 3.1.1 中给出明确结论，不再保留为开放问题，编号同步前移。**

------

## 七、二期方向预留

以下能力不在 MVP 内，但 MVP 架构设计需为其预留扩展性：

| 方向           | 说明                                                 | MVP 架构预留                                                |
| :------------- | :--------------------------------------------------- | :---------------------------------------------------------- |
| 团队视图       | 回答"哪些人的 matter 卡住了"                         | 判定规则预留团队负责人视角；筛选预留"看团队的"              |
| 外部推送       | 飞书 / 邮件 / App 通知                               | 提醒卡片数据结构保持独立可序列化                            |
| 可配置规则     | 用户调整阈值和开关                                   | 七条规则阈值集中配置，不硬编码                              |
| 跨 matter 关系 | 识别 matter 间依赖和阻塞                             | index 头部预留 `related_matters` 字段位（本期不写入不使用） |
| 组织记忆       | insight 结构化检索                                   | insight 文件 index 结构保持一致，后续可加向量索引           |
| 二人复核       | 高风险动作（如 `paused → cancelled`）的二人确认机制  | think 文件评论流可承载第二人的确认意见，不需要新结构        |
| AI 候选回流    | 用 AI 候选 vs 用户最终写入差异训练更准的日期建议模型 | 写入接口已预留对比日志位                                    |

------

*本文档基于 v0.1 反馈调整。核心变更：(1) 状态机补齐 `paused → cancelled` 路径，由 think 触发；(2) 时间型字段（due_at / next_check_at / outcome）补齐写入时机、默认值、可编辑规则；(3) 明确"AI 候选日期不直接落地，须用户确认后由程序写入 index"的边界。*
