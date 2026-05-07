---
type: act
author: liuyu
created: '2026-04-30T11:44:18+08:00'
index_state: indexed
---
# Matter 评分体系 — 系统详细设计 v0.3

> 落地仓库：team-pivot-web
> 版本说明：v0.3 收敛了入口范围、明确了评分对象、加入评论加权、统一用 `pivot_user.id`

---

## 0. 摘要与术语对齐

需求文档说「Matter 进入 result」时由 AI 基于时间线生成人员评分。代码里：

| 文档语义 | 代码事实 | 落点 |
|---|---|---|
| Matter 进入 `result` | append 一个 `type=result` 文件，状态从 `executing` → `finished` 或 `cancelled` | server/matter_status.py |
| 完整时间线 | `discussions/{cat}/{matter_id}/*.md` + `index/{matter_id}.index.yaml` 的 `timeline` | server/matter_index.py |
| 触发事件 | 已有的 `TOPIC_RESULT_CREATED`（在 `publish_matter_append` 内 emit） | server/events.py |
| 后台执行 | 复用 daily_report 的 `asyncio loop + offload to thread + AI oneshot` 模式 | server/daily_report/scheduler.py、server/ai/oneshot.py |
| 用户身份 | `pivot_user.id` (uuid hex) 是新的主键身份 | server/db.py、server/pivot_users.py |

下文统一用 `finished`/`cancelled` 表述 Matter 收口。

---

## 1. 关键决策（产品 + 技术）

> 这一章把 v0.3 与 v0.1/v0.2 的差异以及为什么这么决策显式写下来，避免后面落地时反复争论。

### 1.1 决策 A：评分对象只评 matter.owner（主实施人员）

**结论**：一个 Matter → 一次评分 → **一行评分**，subject 永远等于 `matter.owner`（finished 时刻的 owner）。

**为什么不评所有参与人**

需求 §6 的示例表给了 3 行（zhangsan / lisi / wangwu），看起来在评多人。但实际工作场景里：

| matter 类型 | 占比估计 |
|---|---|
| 个人独立完成 + 一人 verify | ~60% |
| owner 主导 + 1-2 人补充评审 | ~30% |
| 多人协作（跨职能、有交付链路） | ~10% |

**多数 matter 是一个人主导，其他人只是评审者/留意见的"好心人"**。如果把评审者也评分，会带来三个负面后果：

1. **挫伤评审积极性**：同事帮 owner 留意见还要被打分，下次就不愿意发声了
2. **评分稀释**：评审者只说一句"方案不错"，AI 强行给个 3 分中性分，把评分体系噪音化
3. **责任主体模糊**：matter 失败时分散到多人头上，违背"谁交付谁负责"的本意

**为什么仍然把"全量 timeline"喂给 AI**

不评其他人不等于忽略其他人的发言。**其他人的 act/verify/comment 都作为评价 owner 的证据来源**：

- 李四的 verify 通过 → 是 zhangsan（owner）delivery 的正向证据
- 王五的评论"前期字段确认不充分" → 是 zhangsan 的 judgment 负向证据
- CEO 的"做得很好" → 是 zhangsan 的 delivery / accountability 高权重证据（详见 §3.5）

需求 §5.2 的"协作贡献"维度也保留：评的是 owner 是否在过程中主动协作（帮别人推进、积极响应），而不是去评帮 owner 的人。

**owner 转交（owner_change）怎么处理**

代码里 owner 是可以中途转交的（server/api/matters.py `transfer_owner`）。决策：

> **评 finished 时刻的 matter.owner，把 owner_change 之前的所有 timeline 也归入证据池。**

理由：
- 谁拿到结果谁负责——拿了 owner 就拿了交付责任
- 转交本身就是 owner 之间的协作过程，前任的工作通过 timeline 体现为后任的"接手质量"或"返工"证据
- 极端情况（接手太晚、前任坑深）由 AI 的 accountability / process 两个维度自然体现

如果未来发现"接手者背锅"是普遍问题，再加配置开关或拆成"前任评分 + 接手人评分"。v1 不动。

**枝节边界**

- AI 输出非 owner 的 subject → run 标 `failed, error="subject_not_owner"`，**不允许半成品落库**
- matter.owner 为空（未分配） → run 标 `skipped, reason="no_owner"`
- matter.owner 已被删除（pivot_user.status='deleted'） → 仍然评，但前端呈现"已离职：xxx"

### 1.2 决策 B：v1 入口仅 admin，先跑一段再考虑放出

**结论**：v1 评分结果只在 admin 后台展示，Matter 详情页不挂任何入口，普通用户看不到评分相关的任何 UI。

**为什么这么收**

1. **AI 评分质量需要先验证**：先让管理员看几周，AI 是否能稳定输出符合预期的评分；如果偶发抽风，普通用户看到会很难解释
2. **可见范围设计需要数据支撑**：是否要让被评人看到自己分数、要不要全员可见，等有真实数据再决定
3. **避免心理负担**：员工知道每个 finished matter 都被打分会下意识改变行为（比如不敢 finished、堆 review 等好话），先不暴露
4. **降低 v1 实现复杂度**：少一组用户侧 API + 权限分支 + 前端 ScoreCard 组件

**未来开放路径**

设计上保留扩展：`scoring.visibility` 配置项的三个枚举值 `admin_only` / `subjects` / `all` v1 在数据层都支持，只是 UI 上只露 `admin_only` 一个选项。后续：

- v1.x：开 `subjects`（被评人可看自己），admin 配置开关一键放开
- v2：开 `all`（全员可见），看是否真的需要

### 1.3 决策 C：仅 finished 触发评分，cancelled 不评

**结论**：`outcome=finished` 触发评分；`outcome=cancelled` 不触发。

**为什么**

- `cancelled` 的语义是"决定不做了"——可能是市场变了、可能是技术不可行、可能是优先级降级，**不一定是 owner 的责任**
- 给"决定不做"的 matter 评分，容易让人不敢 cancel（本来 cancel 是好习惯）
- 真要复盘失败原因，应该走专门的 `insight` 文件 + 状态 `reviewed`，不是评分系统的事

trigger.py 里加注释，未来如果产品发现需要评 cancelled，admin 配置加开关即可。

### 1.4 决策 D：人员身份统一用 pivot_user.id

**结论**：所有 Scoring 表里的"人"字段都用 `pivot_user_id`（外键到 `pivot_user.id`），不用 `pinyin`。

**为什么**

- pivot_user.id 是 uuid，永久不变；pinyin 是用户可改的（ProfileSetup）
- pivot_user 是新主用户表，长期方向就是它
- 外键约束让数据完整性可被 SQLite 直接保证

**Git timeline 里仍是 pinyin 怎么办**

Git 端的 timeline.creator / owner / comments[].author 落的是写入时的 pinyin，是不可变的。这就形成两个识别系统：

```
   SQLite 端                    Git timeline 端
┌───────────────┐             ┌─────────────────┐
│ pivot_user.id │   解析层    │ pinyin (历史值) │
│  (uuid hex)   │ ◄─────────► │                 │
└───────────────┘             └─────────────────┘
```

**解析点**：

| 时机 | 怎么解析 | 失败处理 |
|---|---|---|
| 入队前（trigger） | matter.owner pinyin → 查 pivot_user → 取 id | 找不到 → run skipped (`reason=owner_not_resolved`) |
| Prompt 注入权重时 | 查 `scoring_commenter_weights` join `pivot_user` → 取当前 pinyin | 用户被 soft-delete → 该权重失效 |
| AI 输出落库时 | 输出的 subject_pinyin / source_comment_author（pinyin）→ 查 pivot_user → 取 id | 匹不上 → 该证据 source_comment_author_id 留 NULL，不报错 |
| 历史 pinyin 改名 | 不做版本回溯 | 老 timeline 文件里的旧 pinyin 匹不上现行用户 → 该证据失去权重，但仍入链（按 weight=1.0 处理） |

`pivot_user.pinyin` 改名是边界场景，做到"匹不上就降级、不报错"即可，v1 不做历史 pinyin 版本表。

---

## 2. 总体架构

```
┌─────────────────────────────────────────────────────────────┐
│  React 前端                                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ AdminPage (/admin)                                   │  │
│  │  └─ ScoringConfigSection                             │  │
│  │      ├─ 总开关 / 模型 / 超时                         │  │
│  │      ├─ 高权重发言人配置（commenter_weights）        │  │
│  │      └─ 最近 5 次 run 概览 + 跳转入口                │  │
│  │                                                      │  │
│  │ AdminScoringPage (/admin/scoring)                    │  │
│  │  ├─ 跨 matter 列表 + 筛选                            │  │
│  │  └─ 点行 → MatterScoresView (drawer)                │  │
│  │      └─ EvidenceDialog                               │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
              │  /api/admin/scoring/*
┌─────────────────────────────────────────────────────────────┐
│  FastAPI                                                    │
│  build_admin_scoring_router  (X-Admin-Password gated)       │
└─────────────────────────────────────────────────────────────┘
              │
┌─────────────────────────────────────────────────────────────┐
│  server/scoring/                                            │
│  ├─ trigger.py        events.subscribe(TOPIC_RESULT_CREATED)│
│  ├─ worker.py         asyncio queue + offload to thread     │
│  ├─ resolve.py        pinyin ↔ pivot_user.id 解析层         │
│  ├─ prompt.py         timeline + weights → system+user prompt│
│  ├─ ai_runner.py      oneshot.generate_text + 解析          │
│  ├─ schema.py         pydantic ScoringOutput + 反伪造校验    │
│  └─ store.py          SQLite repo                           │
└─────────────────────────────────────────────────────────────┘
              │
┌─────────────────────────────────────────────────────────────┐
│  数据层                                                     │
│  ├─ Git 仓库 (discussions/+index/)  ← 时间线证据源         │
│  └─ SQLite (data.db) — 新表                                 │
│     ├─ matter_scoring_runs                                  │
│     ├─ matter_scores                                        │
│     ├─ matter_score_evidence                                │
│     └─ scoring_commenter_weights                            │
└─────────────────────────────────────────────────────────────┘
```

**核心原则**

- 评分**结果**只放 SQLite（事后产物，不污染 git）
- 证据指针指向 git 文件名 + comment 时间戳（可追溯，不复制原文，原文权威源仍是 git）
- AI 不参与状态机判断；只在 `finished` 后被动跑一次
- 用户身份 SQLite 端用 `pivot_user.id`，git 端用 pinyin，解析在 server/scoring/resolve.py 集中处理

---

## 3. 评分规则（核心章节）

> 这一章是 AI 的"宪法"，会作为 system prompt 的核心内容编入。落地时需要 reviewer（产品 + 技术 lead）共同签字。

### 3.1 双层证据模型

评分规则的心智模型是 **「事实层是骨架，评论语义是肌肉」**：

```
                        ┌─────────────────────────────┐
                        │   AI 综合评分（5 维度）      │
                        └─────────────┬───────────────┘
                                      │
              ┌───────────────────────┴───────────────────────┐
              │                                                │
   ┌──────────▼──────────┐                       ┌────────────▼────────────┐
   │  事实层（硬证据）    │                       │  语义层（软证据 + 加权）│
   │  来源：timeline 结构 │                       │  来源：comments 文本    │
   ├─────────────────────┤                       ├─────────────────────────┤
   │ • verify 是否 passed │                       │ • 评论的褒/贬/中性       │
   │ • result.outcome     │                       │ • 评论是否有具体事实     │
   │ • 是否被 owner_change│                       │ • 评论作者是否在权重表   │
   │ • timeline 节奏      │  ◄── 决定能否评 ──►  │ • 提及与归因             │
   │ • act/verify 完整度  │                       │                         │
   │ • 文件类型分布       │                       │                         │
   └──────────────────────┘                       └─────────────────────────┘
            │                                                │
            │ 决定：                                          │ 决定：
            │ • 维度是否可计                                  │ • 维度的置信度（low/med/high）
            │ • 维度的基础分位（1-5 大致档位）                │ • 维度分数的"+/-"调整
            │ • overall 的下限                                │ • overall 的上限
            └─────────────────┬──────────────────────────────┘
                              ▼
                 加权综合（不允许语义层独自支撑维度评分）
```

#### 三条核心原则

**P1. 事实先行（硬约束）**
- 没有事实证据，光有评论不能给分
- 反例：CEO 说"做得不错"但 timeline 里 verify 都是 failed → delivery **不能**给高分
- 极端：没有 verify、没有 result 的 matter 进入 finished → 多数维度走 `null`

**P2. 语义加权是调整器，不是发动机**
- 高权重评论 **可以** 让某维度的 confidence 从 medium 升到 high
- 高权重评论 **可以** 让某维度的分数微调 ±0.5（不能跨整档）
- 高权重评论 **不能** 让一个本来证据为 0 的维度产生分数

**P3. 维度对两层证据的依赖度不同**
- 不同维度天然对事实层 / 语义层的依赖不同，AI 应按下表的偏重判断（详见 §3.2）

### 3.2 五维度定义与证据依赖度

| 维度 | 中文 | 看什么 | 事实层依赖 | 语义层依赖 | 权重 |
|---|---|---|---|---|---|
| `delivery` | 交付质量 | owner 负责事项是否通过验收、结果是否达成 | **强** | 中 | 0.30 |
| `accountability` | 责任闭环 | owner 是否把事情推进到可验收、可收口 | **强** | 弱 | 0.25 |
| `judgment` | 判断质量 | owner 的判断、方案、验收是否准确 | 中 | **强** | 0.20 |
| `collaboration` | 协作贡献 | owner 是否积极响应他人、推进 / 帮助协作 | 中 | **强** | 0.15 |
| `process` | 过程规范 | owner 是否留下必要过程证据、文件齐全 | **强** | 弱 | 0.10 |

#### 每个维度的判断依据细化

**`delivery`（交付质量，权重 0.30）**

- 事实层 ✅
  - verify 文件的 `verifications.result`：passed → 强正向，failed → 强负向
  - result 文件的 outcome：finished → 正向，cancelled → 不评
  - 是否有返工（多次 act 后才 verify 通过）→ 弱负向
- 语义层
  - 验收方（verifier）的评价语义
  - 客户/上层的认可评论（高权重时强化）
- 评分锚定
  - 5：所有 verify 一次通过 + result 明确达成 + 上层认可
  - 4：有小返工但最终通过
  - 3：通过但无明显亮点
  - 2：有 verify failed 但最终补救通过
  - 1：result outcome=finished 但 verify 失败痕迹明显，或者多次返工

**`accountability`（责任闭环，权重 0.25）**

- 事实层 ✅
  - 是否中途被 owner_change 转走（强负向，但要看转走原因）
  - timeline 中是否长期无进展（act 后超 14 天无 verify → 弱负向）
  - 出问题后是否有补救文件（act 失败后再 act → 正向）
- 语义层（弱）
  - "推进慢"、"还没收到"等催促类评论 → 负向
- 评分锚定
  - 5：自始至终 owner，节奏紧凑，主动收口
  - 4：节奏正常，有主动推进
  - 3：节奏正常但被动推进
  - 2：长期停滞过 / 被催促过 / 中途被转走（接手 owner）
  - 1：被强制转走 / 多次反复 / 没有收口意识

**`judgment`（判断质量，权重 0.20）**

- 事实层（中）
  - think 文件中的判断 vs result 是否吻合（事前 think 说"会有 X 风险" vs 后续真出现/未出现 X）
  - act 中的方案选择 vs 后续 verify 是否反复修改
- 语义层 ✅
  - 同行 / 上层对方案的认可或质疑
  - 评论中"判断准确"、"前期没考虑到"等明确判断质量评价
- 评分锚定
  - 5：事前判断被结果强力验证 + 同行认可
  - 4：判断基本准确
  - 3：判断中规中矩，无明显错判
  - 2：有判断失误但补救
  - 1：判断与结果显著相反

**`collaboration`（协作贡献，权重 0.15）**

- 事实层（中）
  - owner 是否对他人的评论 / mentions 有及时响应（comment.created_at 间隔）
  - owner 是否给他人的 matter 留下 verify / 评论
- 语义层 ✅
  - 他人评论中对 owner 协作行为的评价
  - 高权重发言人对 owner 协作的评价（强化）
- 评分锚定
  - 5：积极协作 + 多方好评
  - 4：协作良好
  - 3：基本响应，无亮点
  - 2：有响应不及时的迹象
  - 1：被多次催促仍无响应

**`process`（过程规范，权重 0.10）**

- 事实层 ✅
  - timeline 文件类型完整度：是否有 think / act / verify / result 闭环
  - act / verify / result 是否齐全配对
  - 是否有补录痕迹（filename 时间戳与 created_at 倒序）
- 语义层（弱）
  - 评论中"线下完成了"、"补一下记录"等过程缺失信号
- 评分锚定
  - 5：timeline 干净齐全，节奏清晰
  - 4：基本齐全
  - 3：有缺漏但不影响理解
  - 2：补录痕迹明显
  - 1：关键节点完全未留痕

### 3.3 评分计算规则

```
overall = Σ (dimension_score × weight) / Σ weight
       （仅对 dimension_score ≠ null 的维度求和；null 维度不参与）

confidence:
- high   = 至少 3 个维度有 high confidence 证据
- medium = 默认
- low    = 多数维度只有 low confidence 证据 / 全部维度证据 ≤ 2 条
```

特殊情况：

- 全部维度都 null（证据完全不足） → `skipped_subjects` 加入，**不出评分行**
- 至少 1 个维度有分 → 出评分行，其他 null 维度照常显示 `—`

### 3.4 证据准入规则

| 原文情况 | 是否入链 | confidence |
|---|---|---|
| 明确指向 owner + 描述工作行为 / 结果 | ✅ 入链 | medium / high（看具体度） |
| 有事实 + 有结果 + 有影响 | ✅ 入链 | high |
| "不错 / 很差"，无上下文 | ⚠️ 弱入链 | low |
| 只有情绪化表达，无事实 | ❌ 不入链 | — |
| 同一句有褒有贬 | ✅ 拆成多条 | 各自评判 |
| 评价对象不明确 | ❌ 不入链 | — |
| comment 没选 mentions，但能从所在帖子推断 owner | ✅ 弱入链 | low |
| 同一人重复评价同一事实 | ✅ 合并入链 | 不重复加权 |
| 与 Matter 无关 | ❌ 不入链 | — |

**硬约束**：
- 每个非 null 维度至少 **1 条** evidence 指向该维度
- evidence.source_filename 必须是真实存在的 timeline 文件名（schema 校验拒收伪造）
- evidence.quote 是原文截取（≤200 字符），不允许改写

### 3.5 评论加权机制

#### 3.5.1 配置数据

`scoring_commenter_weights` 表里维护"高权重发言人"白名单：

| pivot_user_id | weight | label | 含义 |
|---|---|---|---|
| `ou_xxxxxx_ceo` | 2.0 | CEO | 强权重 |
| `ou_xxxxxx_cto` | 1.5 | CTO | 中强权重 |
| `ou_xxxxxx_tech` | 1.3 | 技术负责人 | 中权重 |
| 不在表里的人 | 1.0（默认） | — | 默认权重 |

由 admin 在配置页手工维护（详见 §9）。

#### 3.5.2 Prompt 注入

序列化 timeline 给 AI 时，在评论作者后附加权重元信息：

```
[文件 002] type=verify  creator=lisi (CTO, 权重 1.5x)
   verifications: passed - "客户当天验收通过"

[文件 003] type=think  creator=zhangbo  comments:
   - zhangbo (CEO, 权重 2.0x) @ 2026-04-29
     "这个交付质量很高，方向也对，可以推广到其他项目"
   - chenliu @ 2026-04-29
     "+1，做得不错"
```

#### 3.5.3 加权规则（写入 system prompt）

```
【评论权重规则】
- 评论作者标注权重时，该评论作为证据的 confidence 不能低于：
  - weight ≥ 2.0：positive 证据 = high；negative 证据 = high
  - 1.3 ≤ weight < 2.0：positive 证据 = medium；negative 证据 = high
  - weight = 1.0：按通用证据规则判
- 高权重的"明确认可+具体理由"才能被视为 high confidence
- 权重不替代证据规则（P1 事实先行优先）：
  - 高权重的人若仅说"很好"/"+1"等程式化客套，仍按"弱入链或不入链"处理
  - 事实层为负（如 verify failed），高权重评论也不能将该维度抬到正向高分
- 权重作用范围：影响 confidence 等级 + 微调分数（±0.5）；不影响维度可评性
```

#### 3.5.4 期望效果

| 场景 | 处理 |
|---|---|
| CEO（2.0x）说 "case 很好，做得不错" | **high positive**（CEO weight + 给了"很好"理由） |
| CEO（2.0x）只说 "+1" | 不入链（光有权重没事实，规则先于权重） |
| CTO（1.5x）说 "设计可支撑业务，可以开始实施" | **medium-high positive**（具体判断 + CTO weight） |
| 普通同事说 "做得不错" | 弱入链或不入链（无变化） |
| CEO（2.0x）说 "做得很好" 但 verify 全 failed | delivery 维度仍不能高分（事实先行） |

---

## 4. 数据模型

### 4.1 SQLite 新表

追加进 server/db.py 的 `SCHEMA`：

```sql
-- 一次评分任务
CREATE TABLE IF NOT EXISTS matter_scoring_runs (
    run_id              TEXT PRIMARY KEY,
    matter_id           TEXT NOT NULL,
    matter_category     TEXT NOT NULL,
    subject_user_id     TEXT NOT NULL REFERENCES pivot_user(id),
    triggered_by        TEXT NOT NULL,
    triggered_actor_id  TEXT REFERENCES pivot_user(id),
    status              TEXT NOT NULL,
    error               TEXT,
    model               TEXT,
    prompt_tokens       INTEGER,
    completion_tokens   INTEGER,
    started_at          REAL NOT NULL,
    finished_at         REAL,
    timeline_hash       TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_scoring_runs_matter
    ON matter_scoring_runs(matter_id, started_at DESC);
CREATE UNIQUE INDEX IF NOT EXISTS idx_scoring_runs_idempotency
    ON matter_scoring_runs(matter_id, timeline_hash)
    WHERE status IN ('queued','running','success');

-- 单 matter 一行评分（owner 唯一）
CREATE TABLE IF NOT EXISTS matter_scores (
    run_id            TEXT NOT NULL REFERENCES matter_scoring_runs(run_id),
    subject_user_id   TEXT NOT NULL REFERENCES pivot_user(id),
    matter_id         TEXT NOT NULL,
    overall           REAL NOT NULL,
    confidence        TEXT NOT NULL,
    rationale         TEXT NOT NULL,
    delivery          REAL,
    accountability    REAL,
    collaboration     REAL,
    judgment          REAL,
    process           REAL,
    human_override_overall REAL,
    human_override_note    TEXT,
    human_override_by      TEXT REFERENCES pivot_user(id),
    human_override_at      REAL,
    PRIMARY KEY (run_id, subject_user_id)
);
CREATE INDEX IF NOT EXISTS idx_matter_scores_matter
    ON matter_scores(matter_id, subject_user_id);

-- 证据条目
CREATE TABLE IF NOT EXISTS matter_score_evidence (
    id                          INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id                      TEXT NOT NULL,
    matter_id                   TEXT NOT NULL,
    subject_user_id             TEXT NOT NULL REFERENCES pivot_user(id),
    dimension                   TEXT NOT NULL,
    polarity                    TEXT NOT NULL,
    confidence                  TEXT NOT NULL,
    source_kind                 TEXT NOT NULL,
    source_filename             TEXT NOT NULL,
    source_file_type            TEXT NOT NULL,
    source_comment_created_at   TEXT,
    source_comment_author_id    TEXT REFERENCES pivot_user(id),
    weight_applied              REAL NOT NULL DEFAULT 1.0,
    quote                       TEXT NOT NULL,
    explanation                 TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_evidence_run_subject
    ON matter_score_evidence(run_id, subject_user_id, dimension);

-- 高权重发言人配置
CREATE TABLE IF NOT EXISTS scoring_commenter_weights (
    pivot_user_id   TEXT PRIMARY KEY REFERENCES pivot_user(id),
    weight          REAL NOT NULL CHECK(weight >= 0.1 AND weight <= 5.0),
    label           TEXT NOT NULL,
    note            TEXT,
    updated_at      REAL NOT NULL,
    updated_by      TEXT NOT NULL REFERENCES pivot_user(id)
);

-- 配置沿用 settings 表
-- key='scoring.enabled'         '0'|'1'                                 默认 '0'
-- key='scoring.visibility'      'admin_only'|'subjects'|'all'           v1 默认 'admin_only'
-- key='scoring.model'           覆盖默认模型；空字符串 = 跟主 ai.model
-- key='scoring.timeout_seconds' 默认 '120'
```

**幂等关键**：`idx_scoring_runs_idempotency` 部分唯一索引保证「同一 matter 同一时间线版本」只能有一个进行中或已成功的 run。

### 4.2 Git 端不动

需求里证据规则中所有证据来源都已经在 timeline 文件 + matter index 的 `comments[]` 里。证据回指三元组：`(matter_id, source_filename)` 或 `(matter_id, source_filename, source_comment_created_at, source_comment_author_id)`。

---

## 5. 触发与执行流

### 5.1 触发

```python
# server/scoring/trigger.py
def install(*, settings, queue, workspace, pivot_users) -> Callable[[], None]:
    def on_event(ev: Event) -> None:
        if ev.topic != TOPIC_RESULT_CREATED:
            return
        if not _enabled(settings):
            return
        if ev.payload.get("outcome") != "finished":
            return
        owner_pinyin = ev.payload.get("matter_owner_pinyin")
        owner = pivot_users.find_by_pinyin(owner_pinyin)
        if owner is None:
            log.info("scoring skipped: owner not resolved %s", owner_pinyin)
            return
        queue.enqueue(
            matter_id=ev.matter_id,
            subject_user_id=owner.id,
            triggered_by="auto",
            triggered_actor_id=ev.payload.get("actor_user_id"),
        )
    return subscribe(on_event)
```

### 5.2 后台 worker

照抄 daily_report scheduler 的 「asyncio loop + offload to thread」 模式：FastAPI lifespan 起后台 task，`await queue.get()` → `asyncio.to_thread(run_scoring_once, job)`。v1 单 worker 串行（评分任务不密集，OpenRouter rate limit 友好）。

### 5.3 单次评分

```python
def run_scoring_once(job, *, store, workspace, settings, pivot_users, weights_repo):
    index = read_matter_index(matter_index_path(workspace.index_dir, job.matter_id))
    if index is None:
        store.mark_skipped(job, "matter_not_found"); return
    timeline_hash = sha256_canonical(index)
    if job.triggered_by != "admin:rerun" and store.has_success(job.matter_id, timeline_hash):
        store.mark_skipped(job, "duplicate_timeline"); return
    run_id = store.start_run(job, timeline_hash, model=...)
    try:
        weights = weights_repo.list_with_pinyin(pivot_users)
        prompt   = build_scoring_prompt(index, weights, subject_pinyin=...)
        raw      = generate_text(messages=prompt, ...)
        parsed   = parse_and_validate(raw, index, expected_subject=...)
        store.write_results(run_id, parsed, pivot_users)
        store.finish_run(run_id, "success")
    except (AIError, TimeoutError, SchemaError) as e:
        store.finish_run(run_id, "failed", error=str(e))
```

启动时 sweep：把 `running` 超 10 分钟的 run 标 `failed, error="orphan"`。

---

## 6. AI Prompt 设计

System prompt 由 §3 评分规则 + 评论加权规则组成；user prompt 是序列化后的 timeline。

```
【评分对象】
- 仅评分 matter.owner（本次为 {subject_pinyin}）
- 其他人的发言只作为证据来源，不要为他们生成 score 行
- 输出 scores 数组长度 ≤ 1

【双层证据模型】
事实层（硬证据，来自 timeline 结构）：
- verify 是否 passed、result.outcome、是否被 owner_change、timeline 节奏、文件完整度
语义层（软证据，来自 comments 文本）：
- 评论的褒贬、具体度、作者权重

【三条核心原则】
P1. 事实先行：没有事实证据，光有评论不能给分
P2. 语义加权是调整器，不是发动机
P3. 维度对两层证据的依赖度不同

【评分维度】五维度 + 权重见 §3.2 表
【评分含义】1-5 含义见 §3.3
【证据准入规则】见 §3.4
【评论权重规则】见 §3.5.3

【输出 schema（严格 JSON，不要 markdown 代码块）】
{ "scores": [{
    "subject_pinyin": "{subject_pinyin}",
    "overall": 4.2, "confidence": "high",
    "rationale": "…",
    "dimensions": {"delivery":4.5, ...},
    "evidence": [{
      "dimension":"delivery", "polarity":"positive", "confidence":"high",
      "source_filename":"003_lisi_verify_xxx.md",
      "source_comment_created_at": null,
      "source_comment_author": null,
      "weight_applied": 1.0,
      "quote":"…", "explanation":"…"
    }]
  }],
  "skipped_subjects": []
}

【硬约束】
- subject_pinyin 必须等于 {subject_pinyin}
- dimension 非 null 时至少 1 条 evidence 指向该维度
- source_filename 必须出自我提供的 timeline；不要编造
- evidence.quote 是原文截取（≤200 字符），不要改写
- 不评价私人态度，只评价工作行为
```

**timeline 序列化**保留 `creator/owner/comment.author/comment.mentions` + 权重标注。超大 timeline 截断策略：每个文件 body ≤ 5000 字符，总 ≤ 50k 字符；超出按 `think/insight` 倒序丢弃，保留所有 `act/verify/result`。

---

## 7. 输出校验

```python
class EvidenceItem(BaseModel):
    dimension: Literal["delivery","accountability","collaboration","judgment","process"]
    polarity: Literal["positive","negative","neutral"]
    confidence: Literal["low","medium","high"]
    source_filename: str
    source_comment_created_at: str | None = None
    source_comment_author: str | None = None
    weight_applied: float = Field(ge=0.1, le=5.0)
    quote: str = Field(max_length=400)
    explanation: str = Field(max_length=300)

class SubjectScore(BaseModel):
    subject_pinyin: str
    overall: float = Field(ge=1.0, le=5.0)
    confidence: Literal["low","medium","high"]
    rationale: str = Field(max_length=500)
    dimensions: dict[str, float | None]
    evidence: list[EvidenceItem] = Field(min_length=1)

class ScoringOutput(BaseModel):
    scores: list[SubjectScore] = Field(max_length=1)
    skipped_subjects: list[str] = []

def parse_and_validate(raw, index, expected_subject):
    parsed = ScoringOutput.model_validate_json(_strip_code_fences(raw))
    valid_filenames = {Path(t["file"]).name for t in index.get("timeline") or []}
    for s in parsed.scores:
        if s.subject_pinyin != expected_subject:
            raise SchemaError(f"subject_not_owner: {s.subject_pinyin} ≠ {expected_subject}")
        for dim, score in s.dimensions.items():
            if score is None: continue
            if not any(e.dimension == dim for e in s.evidence):
                raise SchemaError(f"{s.subject_pinyin}.{dim} 给分但无证据")
        for e in s.evidence:
            if e.source_filename not in valid_filenames:
                raise SchemaError(f"伪造证据文件: {e.source_filename}")
    return parsed
```

校验失败 → run 标 `failed`，原始 raw 落 `error` 字段。**不允许半成品落库**。

---

## 8. API 设计

### 8.1 v1 仅 Admin 端（决策 B）

```
GET    /api/admin/scoring/config
PUT    /api/admin/scoring/config
GET    /api/admin/scoring/runs?status=&matter_id=&limit=&offset=
GET    /api/admin/scoring/runs/{run_id}
POST   /api/admin/scoring/matters/{matter_id}/rerun
POST   /api/admin/scoring/scores/{run_id}/override          # v1.1

# 高权重发言人 CRUD
GET    /api/admin/scoring/commenter-weights
POST   /api/admin/scoring/commenter-weights
PUT    /api/admin/scoring/commenter-weights/{user_id}
DELETE /api/admin/scoring/commenter-weights/{user_id}
```

### 8.2 用户侧端点（v1 不开放）

`GET /api/matters/{matter_id}/scores` 设计上保留接口规范，但 v1 **不实现/不暴露**。等灰度后开放时直接加上即可。

权限：所有 admin 端点都走 `X-Admin-Password` header。

---

## 9. 页面交互设计（v1 仅 admin）

### 9.1 入口总览

| 视图 | 路由 | 内容 | 谁能看 |
|---|---|---|---|
| **Admin 主页评分卡片** | `/admin` | 配置开关 + 高权重发言人 + 最近 5 次概览 | 仅 admin |
| **Admin 评分管理页** | `/admin/scoring` | 跨 matter 列表 + 筛选 + 单 matter 详情 drawer | 仅 admin |
| **MatterScoresView** | `/admin/scoring` 内的 drawer | 该 matter 的评分 + 证据 | 同上 |
| **EvidenceDialog** | drawer 内 | 按维度分组的证据 | 同上 |

> Matter 详情页 `/m/:matter_id` 在 v1 **不挂任何评分入口**。

### 9.2 入口 1：`/admin` 评分配置卡片

挂载位置：AdminPage 的右栏，与 `<MarkdownSettingsSection />`、`<AISettingsSection />` 平级。

```
┌─ 🏆 Matter 评分配置 ──────────┐
│ Matter 进入 finished 后由 AI  │
│ 基于时间线生成 owner 评分。   │
│ ───────────────────────────── │
│ ☑  评分功能总开关             │
│   关闭后 finished 不再触发评分 │
│                                │
│ 评分结果可见范围               │
│ ◉ 仅管理员（v1 默认）          │
│ ○ 仅本人和管理员（暂不可选）   │
│ ○ 全员可见（暂不可选）         │
│                                │
│ 评分模型（覆盖默认）           │
│ [anthropic/claude-sonnet-4-5 ▼]│
│ 留空时使用 AI 助手默认模型     │
│                                │
│ 单次超时(秒) [120]             │
│                                │
│ ── 高权重发言人 ────────────  │
│ 这些人的评论 AI 视为更强信号   │
│ 📋 张三  CEO         2.0x  ✏🗑 │
│ 📋 李四  CTO         1.5x  ✏🗑 │
│ 📋 王五  技术负责人  1.3x  ✏🗑 │
│ [+ 从联系人添加]                │
│                                │
│ ── 最近 5 次评分任务 ──────── │
│ ✅ 客户验收流程优化  04-29     │
│     张三  4.2 / high            │
│ ✅ 接口对接 v2     04-28        │
│     李四  3.8 / medium          │
│ ❌ 邮件模板更新   04-26         │
│     ai_timeout    [↻ 重跑]      │
│ ⏳ 月度复盘 排队中               │
│ ⏳ 客户回访  生成中…             │
│                                │
│ [前往评分管理 →]   [保存]      │
└────────────────────────────────┘
```

#### 9.2.1 高权重发言人添加流程

点 `[+ 从联系人添加]` 弹 modal：

```
┌─ 添加高权重发言人 ─────────────────── ✕ ─┐
│ 联系人 [⌕ 搜索…             ▼]            │
│         张三 (CEO, zhangbo)               │
│         李四 (CTO, lisi)        ← 已添加   │
│         王五 (技术负责人, wangwu)          │
│         陈六 (产品经理, chenliu)           │
│                                            │
│ 权重 [2.0]  范围 0.1 - 5.0                │
│  - 1.0  默认（不在表里就是这个）           │
│  - 1.3  中权重（资深 reviewer）            │
│  - 1.5  中强权重（CTO 级）                 │
│  - 2.0  强权重（CEO 级）                   │
│                                            │
│ 显示标签 [CEO]  必填，用于 prompt 注入     │
│ 备注 [........]  可选                      │
│                                            │
│              [取消]    [添加]              │
└────────────────────────────────────────────┘
```

### 9.3 入口 2：`/admin/scoring` 评分管理页

新路由，需要二次输入 admin password。

```
┌──────────────────────────────────────────────────────────────────────┐
│ ← 返回 admin           评分管理                                      │
│ 筛选: [状态 全部▼] [模型 全部▼] [范围 最近30天▼] [⌕ 搜索 matter…] │
│                                                                      │
│ ┌──────────────────────────────────────────────────────────────────┐ │
│ │ Matter             │ Owner │ 总分 │ 置信度 │ 时间      │ 操作   │ │
│ ├──────────────────────────────────────────────────────────────────┤ │
│ │ 🟢 客户验收流程优化│ 张三  │ 4.2  │ 🟢 high │04-29 18:30│[详情]  │ │
│ │   eng/auth-redesign│       │      │         │           │[↻重跑] │ │
│ ├──────────────────────────────────────────────────────────────────┤ │
│ │ 🟢 接口对接 v2     │ 李四  │ 3.8  │ 🟡 med  │04-28 12:11│[详情]  │ │
│ │   eng/api-v2       │       │      │         │           │[↻重跑] │ │
│ ├──────────────────────────────────────────────────────────────────┤ │
│ │ 🔴 邮件模板更新    │   —   │   —  │   —    │04-26 09:00│[详情]  │ │
│ │   ops/email-tpl    │       │      │         │ ai_timeout│[↻重跑] │ │
│ ├──────────────────────────────────────────────────────────────────┤ │
│ │ ⏳ 月度复盘        │ 陈六  │ 排队 │   —    │04-30 09:11│        │ │
│ │   ops/monthly      │       │ 中   │         │           │        │ │
│ └──────────────────────────────────────────────────────────────────┘ │
│                                                  [< 1 2 3 ... 12 >] │
└──────────────────────────────────────────────────────────────────────┘
```

**列表行为**：

- 一行一个 run（按 started_at 倒序）
- 「详情」打开右侧 drawer，展示 `<MatterScoresView />`
- 「重跑」二次确认 → `POST /admin/scoring/matters/{id}/rerun` → 行变 `queued` → 5 秒轮询直到 success/failed
- 失败行的 error 在「详情」drawer 中可见（含原始 AI 输出）

### 9.4 入口 3：MatterScoresView（drawer）

从 `/admin/scoring` 列表点「详情」打开右侧 drawer。

```
┌─ 客户验收流程优化 ─────────────────────────────────────── ✕ ─┐
│ matter_id: eng/auth-redesign · 已 finished 于 04-29 18:30    │
│                                                              │
│ ┌─ 📊 Matter 评分 ───────────────────────────────────────┐  │
│ │ run_id: a3f9...   生成于 04-29 18:35                    │  │
│ │ 模型 anthropic/claude-sonnet-4-5  · prompt 4123 tokens  │  │
│ │                          [🔄 重新生成]  [⚙ 元信息]      │  │
│ │ ───────────────────────────────────────────────────── │  │
│ │ ┌──────────────────────────────────────────────────┐   │  │
│ │ │ 👤 张三 (zhangbo)  ⭐ 4.2 / 5    🟢 high          │   │  │
│ │ │                                                    │   │  │
│ │ │ "负责的行动项按期完成，verify 通过且客户已接受     │   │  │
│ │ │  交付。CEO 张三在 result 评论中明确认可。"         │   │  │
│ │ │                                                    │   │  │
│ │ │ ┌─ 五维度 ───────────────────────────────────┐   │   │  │
│ │ │ │ delivery       4.5  ████████░░  high       │   │   │  │
│ │ │ │ accountability 4.0  ███████░░░  high       │   │   │  │
│ │ │ │ judgment       —    证据不足                │   │   │  │
│ │ │ │ collaboration  3.5  ██████░░░░  medium     │   │   │  │
│ │ │ │ process        4.0  ███████░░░  high       │   │   │  │
│ │ │ └─────────────────────────────────────────────┘   │   │  │
│ │ │                            [查看证据 (5) →]        │   │  │
│ │ └──────────────────────────────────────────────────┘   │  │
│ └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

**Admin 视角额外控件**：
- 「🔄 重新生成」：调 `POST /admin/scoring/matters/{id}/rerun`
- 「⚙ 元信息」：弹 modal 显示 timeline_hash、原始 AI 输出（折叠）、prompt token 用量
- 「✏ 人工修正」：v1.1 启用，v1 禁用

### 9.5 入口 4：EvidenceDialog

```
┌─ 张三的评分证据 ────────────────── 客户验收流程优化 ─── ✕ ─┐
│ 总分 4.2 / 5    🟢 high 置信                                │
│ ────────────────────────────────────────────────────────── │
│                                                              │
│ ▼ 交付质量 (delivery)        4.5  · 2 条证据                │
│ ┌────────────────────────────────────────────────────────┐  │
│ │ ✅ positive  🟢 high   weight 1.0   verify 文件          │  │
│ │ 003_lisi_verify_a1b2c3.md      作者 李四 (CTO, 1.5x)    │  │
│ │ 时间 2026-04-22 14:00                                   │  │
│ │ "verify 通过：交付质量符合预期，客户当天验收"           │  │
│ │ 🤖 李四在 verify 中确认张三的交付质量                    │  │
│ │                              [跳到 timeline ↗]          │  │
│ └────────────────────────────────────────────────────────┘  │
│ ┌────────────────────────────────────────────────────────┐  │
│ │ ✅ positive  🟢 high   weight 2.0   result 评论          │  │
│ │ 005_zhangbo_result_xx.md  →  评论                       │  │
│ │ 评论作者 张三 (CEO, 2.0x)  · 2026-04-29 18:00            │  │
│ │ "这个交付质量很高，方向也对，可以推广到其他项目"        │  │
│ │ 🤖 CEO 在 result 评论中明确认可（高权重证据）            │  │
│ │                              [跳到 timeline ↗]          │  │
│ └────────────────────────────────────────────────────────┘  │
│                                                              │
│ ▼ 责任闭环 (accountability)  4.0  · 1 条证据                │
│ ▼ 协作贡献 (collaboration)   3.5  · 1 条证据                │
│ ▼ 判断质量 (judgment)        — (证据不足，不评分)            │
│ ▼ 过程规范 (process)         4.0  · 1 条证据                │
│                                                  [关闭]      │
└──────────────────────────────────────────────────────────────┘
```

新增字段：每条证据显示 `weight 1.0` / `weight 1.5x` / `weight 2.0x`，让 admin 知道该证据是否走了加权路径。

### 9.6 路由与代码组织

```
web/src/components/admin/scoring/
  ScoringConfigSection.tsx
  CommenterWeightsSubsection.tsx
  AddCommenterWeightDialog.tsx

web/src/pages/admin/
  AdminScoringPage.tsx
  ScoringRunsTable.tsx
  ScoringRunDetailDrawer.tsx

web/src/components/scoring/
  MatterScoresView.tsx
  ScoreSummaryCard.tsx
  DimensionBars.tsx
  EvidenceDialog.tsx
  ScoreConfidenceBadge.tsx
  RunMetadataDialog.tsx
```

App.tsx 路由新增：

```
<Route path="/admin" element={<AdminPage />} />
<Route path="/admin/scoring" element={<AdminScoringPage />} />
```

---

## 10. 异常路径与边界

| 场景 | 处理 |
|---|---|
| `scoring.enabled=0` | finished 不入队；不留 run 记录 |
| matter.owner 未分配 / 无法解析为 pivot_user | 不入队，log info（不留 run） |
| matter.owner 已被软删除 | 仍入队评分；前端显示"已离职：xxx" |
| AI key 未配置 | 入队 → run failed, error="ai_key_missing"；admin UI 红色提示 |
| timeline 为空 / 单文件 | run skipped, reason="insufficient_timeline" |
| AI 输出非 JSON / schema 校验失败 | run failed，原始 raw 落 error 字段 |
| AI 输出 subject ≠ owner | run failed, error="subject_not_owner" |
| AI 输出 scores.length > 1 | pydantic 自动拒收（max_length=1） |
| AI 输出 scores=[]（all skipped） | run success，但前端显示"AI 判断证据不足，未生成评分" |
| 反复 finished ↔ executing | 仅当 timeline_hash 变化才自动重跑；admin rerun 绕过 |
| evidence 引 comment 但缺 created_at/author | 校验拒收，run 失败 |
| 重启服务 in-flight run | 启动 sweep 把 running >10min 的标 failed |
| Worker crash / SIGTERM | 队列非持久化（v1 接受丢失）；admin 在评分管理页手动触发补 |
| 单 matter 同时多次入队 | `idx_scoring_runs_idempotency` 部分唯一索引保证只一行 queued/running |
| pinyin 改名导致历史评论匹不上权重 | 该证据降级为 weight=1.0，run 不失败 |
| 高权重发言人 user 被删 | weights 表外键约束 ON DELETE CASCADE，自动清理 |

---

## 11. 测试策略

| 层级 | 用例 |
|---|---|
| `schema.py` | 拒收伪造文件名 / 缺证据 / 超分 / scores>1 / subject≠owner；接受合法 JSON；剥离 markdown 代码块 |
| `prompt.py` | timeline 序列化保留 owner/creator/mentions/weight；body 截断；超大 timeline 丢弃 think/insight；权重表正确注入 |
| `resolve.py` | pinyin → pivot_user.id 映射；改名兜底；soft-delete 用户兜底 |
| `store.py` | 幂等唯一索引；rerun 绕过；scores 限 1 行；human_override 字段读写；commenter_weights CRUD |
| `trigger.py` | 仅 finished 入队（cancelled / paused / reviewed 不入队）；scoring.enabled=0 不入队；owner 未分配不入队；多订阅器互不干扰 |
| `worker.py` | AI 失败 → run failed；超时；并发只一条；orphan sweep |
| `api/admin/scoring.py` | admin password gate；rerun；commenter_weights 增删改查 |
| 前端 | AdminScoringPage 状态轮询；CommenterWeights 添加/编辑/删除；EvidenceDialog 跳转 timeline |
| 集成 | 真实 matter index → mock AI（含权重场景）→ DB 写入 → API 读出 → React Testing Library 渲染 |

---

## 12. 与需求 §8 开放问题的对应

| 需求开放问题 | v1 决定 | 备注 |
|---|---|---|
| 是否需要人工修正 | v1 schema 留 `human_override_*` 字段；UI 入口禁用态；v1.1 开端点 | DB 改动一次到位避免后续 migration |
| 是否推送 IM 评分 | v1 不推送 | 复用 server/notify.py 的 `broadcast_card` 是 v1.x 范围；admin_only 可见时管理员主动看 |
| `cancelled` 是否评分 | v1 仅 `finished` 评分 | 决策 C；trigger.py 加注释，未来在 admin 配置加开关即可 |
