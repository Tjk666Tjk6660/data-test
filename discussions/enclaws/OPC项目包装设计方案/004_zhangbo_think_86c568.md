---
type: think
author: zhangbo
created: '2026-04-23T14:42:41+08:00'
index_state: indexed
---
# OPC 实施方案：执行版

## 一、产品机制（老板视角）

**一句话**：老板选公司类型 → 系统建一支 6 人班底（都是待启用态）→ 老板逐个启用（每次启用弹窗问几个参数）→ 员工按 cron 自动干活、文件共享接力 → 老板只看通知、审几个关键节点。

```
① 选公司类型（本期只有"内容工作室"）
② 系统建 6 员工（待启用）+ 初始化 workspace 目录
③ 绑通知 IM（OPC 助理 bot 加到老板自己的飞书/钉钉）
④ 逐个点员工「启用」→ 弹对话框问参数 → EC 一次性配完 agent+模型+skill+cron+prompt
⑤ 进 Today 主场，老板日常只做 5 件事：刷通知 / 丢灵感 / 审稿 / 审回复 / 看晚报
```

**渐进启用**：没想好的员工可以先不启用，其他员工不受影响；启用/暂停/重新配置随时做。

**员工协作靠文件接力**（不用事件总线）：通过共享 workspace 的文件 frontmatter 里 `status` 字段传递状态（`draft → approved → published` / `inquiry → gathering_info → ready_for_quote → negotiating → signed → completed`）。

---

## 二、每个员工具体配置

每个员工 6 维度：职责 / 模型 / cron / skill / 启用参数 / prompt 要点。完整 systemPrompt 模板见**附录 C**。

---

### 2.1 选题策划官

**① 职责** — 每日扫平台热榜 + 竞品爆款 + 平台算法偏好，产出今日 3-5 条候选选题（含合规注记）。

**② 模型** — `standard`

**③ Cron**

| 时刻 | action |
|---|---|
| 06:30 | scan-topics |

**④ Skill**
默认：`platform-hotlist` / `competitor-scan` / `compliance-check` / `workspace.*`

**⑤ 启用参数**

| 参数 | 必选 | 默认 | 说明 |
|---|---|---|---|
| `platforms` | ✅ | — | 要扫哪些平台热榜（小红书/微博/抖音/知乎/公众号）|
| `verticalKeywords` | ✅ | — | 垂类关键词（如"基金,理财,副业"）|
| `complianceLevel` | 可选 | `general` | 合规词库严格度（金融/医疗/法律/通用）|

**⑥ prompt 要点** — 按 3 个参数渲染；action=`scan-topics` 做：热榜扫描 + 竞品扫描 + 读 `published/` 近 30d 历史去重 + 合规打标 + 写 `topics/{date}.md` + 通知。

---

### 2.2 内容创作官

**① 职责** — 读选题 + 老板灵感 → 产出双平台可发布初稿（纯文案，本期不出配图）。

**② 模型** — `senior`（写作质量要求高）

**③ Cron**

| 时刻 | action |
|---|---|
| 08:30 | create-draft |

**④ Skill**
默认：`platform-format-adapt` / `workspace.*`

**⑤ 启用参数**

| 参数 | 必选 | 默认 | 说明 |
|---|---|---|---|
| `styleSamples` | 可选（强烈建议）| 无 | 老板过去爆款 3 篇，LLM 学风格 |
| `defaultLength` | 可选 | `1500` | 公众号版默认字数 |

**⑥ prompt 要点** — action=`create-draft` 做：读 `topics/{date}.md` + `inspiration/` 近 24h → 写双平台版本（公众号长文 + 小红书短文+emoji+3 个 hashtag）→ 写 `drafts/{date}-{shortId}.md(status=pending_review)` + 通知。

---

### 2.3 分发编辑

**① 职责** — 审过的初稿按平台调性改写 + 排期 + 到点发。**能自动发的自动发；不能自动的每平台发一条提醒通知（内含完整可复制文案），老板长按复制到 App 粘贴发送。**

**② 模型** — `standard`

**③ Cron**

| 时刻 | action |
|---|---|
| 11:00 | adapt-and-schedule |
| 14:00 | publish-or-pack |

**④ Skill**
默认：`platform-format-adapt` / `workspace.*`
随 `publishTargets` 动态绑：公众号 → `wechat-mp-publish`；任一半自动平台 → `platform-publish-pack`。

**⑤ 启用参数**

| 参数 | 必选 | 默认 | 说明 |
|---|---|---|---|
| `publishTargets` | ✅ | — | 多选 + 每项授权（公众号 appId+secret / 小红书 / 视频号 / 抖音 / 知乎）|

**⑥ prompt 要点** — `adapt-and-schedule`（11:00）扫 `drafts/(status=approved)` 做多平台适配 + 生成发布包 + 排 14:00；`publish-or-pack`（14:00）**按平台分叉**：公众号调 `wechat-mp-publish` 真发；半自动平台对每一条单独推通知（含完整可复制文案 + `[打开 OPC]` 按钮）。

---

### 2.4 社区管家

**① 职责** — 拉业务 IM 私信 + 平台评论 → 4 档分类（粉丝咨询/合作询盘/普通互动/spam）→ **每天精选上限 N 条起草回复**，老板审过才发。

**② 模型** — `standard`

**③ Cron**

| 时刻 | action |
|---|---|
| 每小时 | ingest-im（只分类归档，不起草普通回复）|
| 每天 21:30 | curate-and-reply（精选起草待审）|

**④ Skill**
默认：`message-classify` / `message-quality-rank` / `deal-inquiry-detect` / `workspace.*`
随 `businessChannels` 动态绑（成对）：`wechat-personal-{fetch,reply}` / `wechat-mp-comment-{fetch,reply}` / `xiaohongshu-comment-{fetch,reply}`

**⑤ 启用参数**

| 参数 | 必选 | 默认 | 说明 |
|---|---|---|---|
| `businessChannels` | ✅ | — | 粉丝/合作方在哪联系你（个人微信 / 公众号评论 / 小红书评论）|
| `dailyReplyMax` | 可选 | `8` | 每天最多起草 N 条，不强凑；当天不够优质就少于此数 |
| `contactMethod` | 可选 | `私信我` | 群聊/评论识别到合作询盘时的引导话术（"请联系 xxx"）；IM 私聊场景不用 |

**固定行为**（不暴露用户）：每小时拉、21:30 精选起草、全部老板审后发。

**⑥ prompt 要点**
- `ingest-im`（每小时）：按 channel 拉 → 写 `inbox/raw/` → 正则扫合作关键词 → `message-classify` 归 4 档 → 按分类分流：
  - `spam` → 归档不通知
  - `inquiry` + **IM 私聊** → 写 `leads/` + 即时唤醒商务经理
  - `inquiry` + **群聊/评论** → **不开 deal**，直接用 `contactMethod` 发固定引导话术（不走老板审）
  - `casual/faq` → 标 `pending_curation:true`，留待 21:30
- `curate-and-reply`（21:30）：读待筛池 → `message-quality-rank` 打分 → 取**质量达标**的 top N（N ≤ `dailyReplyMax`，没凑够就实际数，都不达标则 0）→ 起草到 `messages/replies/(pending_review)` → 通知老板。

---

### 2.5 商务经理

**① 职责** — 接 IM 私聊合作询盘，AI 先自主问清基本信息（品牌/产品/预算/时间/合作形式）→ 推老板审报价 → 老板改/确认后发出 → **进入谈判阶段，此后对方每一轮回复都推老板审回**（AI 只起草建议，不自主发）→ 直至 deal 到终态。**商务经理负责把 deal 推到终态**——对方超时未回主动提醒老板"是否放弃"。

**Deal 生命周期**：
信息收集中 → 待老板审报价 → 谈判中 → **【已签约 或 已放弃】**（两者均为终态）

异常分支：3 轮追问拿不到关键信息 → 标"卡住"求助老板

**签约后的履行/结款**不归商务经理管——分发编辑发稿时、财务助理对账时，会在 deal 上标记附加字段（`delivered_at` / `paid_at`），不影响商务经理状态机。

**② 模型** — `senior`（涉及真金白银 + 多轮对话判断要细）

**③ Cron**

| 触发源 | 做什么 |
|---|---|
| 社区管家即时唤醒（新 IM 私聊询盘）| 开启信息收集（首轮追问）|
| 社区管家拉到"信息收集中"deal 的新消息 | 继续追问缺失字段 |
| 信息收集完整 → "待老板审报价" | 生成报价建议 + 推老板审 |
| 社区管家拉到"谈判中"deal 的新消息 | 记录对方回复 + 起草建议回复 + 推老板审 |
| 每天 08:00 扫遗留 | "待老板审报价"超 6h 未审 / "谈判中"对方超过 3 天未回 → 通知老板 |
| 每天 20:30 | 今日商务汇总 + 对"谈判中"deal 做疑似谈妥识别 |

**④ Skill**
默认：`deal-inquiry-detect` / `message-classify` / `workspace.*`
随社区管家 `businessChannels` 联动绑 reply skill（仅 IM 私聊渠道）：个人微信 → `wechat-personal-reply`。

**⑤ 启用参数**

| 参数 | 必选 | 默认 | 说明 |
|---|---|---|---|
| `blacklist` | 可选 | 空 | 不接的品类（如"赌博,贷款,保健品"），命中直接跳过 |

**⑥ prompt 要点**
- **AI 自主发的边界**：只在"信息收集"阶段能自主发，且仅限纯问信息（品牌/产品/预算/时间/形式）；涉及金额/承诺/合同/折扣即便在此阶段也降级等老板审
- **收集信息**：追问★必需字段（品牌/合作形式/预算）；3 轮仍不齐标"卡住"让老板介入；齐了推进到"待老板审报价"
- **生成报价**：综合对方历史 / 类似账号成交价 / 本账号粉丝量 / 合作形式给出报价区间 → 写进 deal → 推老板 high priority 通知
- 老板 [确认发送] 即调 reply skill 真发，deal 进入"谈判中"
- **谈判中（报价已发之后）**：**对方每一轮回复都起草建议推老板审**（AI 不再自主发）；通知含对方原话 + AI 建议回复 + 老板按钮 [修改后发送] / [确认发送] / [不回]
- **终态推进**：老板 portal 点【已签约】→ 填合同（金额/履行截止/支付条款）→ 阶段"已签约"；点【已放弃】→ 阶段"已放弃"
- **超时提醒**：对方 > 3 天未回 → 08:00 扫遗留时通知"XX 合作 N 天未回复，是否放弃？"老板回"是"→ 标已放弃；回"再等等"→ 推迟 3 天再提醒
- **今日商务**（20:30）：按阶段汇总；对"谈判中"deal 做疑似谈妥识别（LLM 找成交信号），标出来提醒老板去 portal 确认签约

---

### 2.6 财务助理

**① 职责** — 归档每日收入 → 归因到具体 `published/` 条目 → 日报/月报 + 标记最赚钱内容。

**② 模型** — `standard`

**③ Cron**

| 时刻 | action |
|---|---|
| 21:00 | daily-income |
| 每月 1 号 07:00 | monthly-report |

**④ Skill**
默认：`roi-attribute` / `workspace.*`
数据拉取 skill 复用分发编辑的授权（同份 token）。

**⑤ 启用参数**

| 参数 | 必选 | 默认 | 说明 |
|---|---|---|---|
| `dataSources` | 可选 | 继承分发编辑 | 拉哪些平台数据；已授权的自动带过来 |
| `incomeMode` | ✅ | — | 流水导入方式：手贴 / CSV / 两者结合 |
| `costTracking` | 可选 | `true` | 是否把 AI 成本也列入日报 |

**⑥ prompt 要点** — `daily-income`（21:00）拉平台统计 + 读 `deals/completed/(今日结款)` + 读 `income/raw/` → `roi-attribute` 归因到具体 `published/` → LLM 写日报（含最赚钱内容标注）→ 写 `income/daily/{date}.md` + 通知。

---

## 三、EC 后端要做的（D:\workspace\ai\EnClaws）

### 3.1 新增 RPC

| RPC | 作用 | 新文件 |
|---|---|---|
| `workspace.*`（read/write/list/query/delete/stat）| 租户沙箱文件操作；tenant 隔离 + agent 一次性 token + 路径 sandbox（禁 `..`/绝对路径/symlink 跳出）+ 审计日志 | `tenant-workspace-api.ts` |
| `tenant.companyTemplate.*`（list/apply/current）| 选公司 → 建 6 pending 员工 + 初始化 workspace 骨架 | `tenant-company-template-api.ts` |
| `tenant.opcEmployee.*`（list/activationSpec/activate/deactivate/reconfigure）| 启用员工时一次性完成 token 存 / 配置写 / prompt 渲染 / 模型绑 / skill 绑 / cron 注册 | `tenant-opc-employee-api.ts` |
| `notification.dispatch` | 向通知层 channel 推消息（支持 high priority 加红）| `opc-notification-api.ts` |

### 3.2 EC 改造

- **Cron 加 `agent-task` 模式**（`src/cron/service.ts` + `delivery.ts`）：到点拉起 agent、注入 system event `"你被 {time} 唤醒，执行 action={action}"`、不广播 IM、写 `schedule/` 日志
- **Agent token 注入**（`src/agents/`）：一次性 JWT `{tenantId, agentId, scope}` + TTL。**复用 Captain `ec对app运行时的支持需求/009` 帖 `ENCLAWS_GATEWAY_TOKEN` 思路**

### 3.3 Skill 字典

#### 3.3.1 差异化 Skill（10 个，放 `skills-pack/`）

| Skill | 干什么 | 输入 | 输出 | 谁用 | 底座 |
|---|---|---|---|---|---|
| `platform-hotlist` | 抓指定平台热榜前 N 条 | `{platform, topN, verticalKeywords?}` | 热榜条目列表 | 选题策划官 | `src/browser/` |
| `competitor-scan` | 扫垂类竞品近 N 天高赞内容 | `{accountList, days, minLikes?}` | 内容列表 | 选题策划官 | `src/browser/` |
| `compliance-check` | 给文本打合规标签 | `{text, level}` | `{risk, hits, suggestion}` | 选题策划官 | 本地词库 + LLM |
| `platform-format-adapt` | 初稿按目标平台调性改写 | `{content, targetPlatform}` | 适配文稿 | 内容创作官、分发编辑 | 纯 LLM prompt |
| `message-classify` | 消息归 4 档 | `{messages}` | 每条带 `classification` | 社区管家 | 纯 LLM 批量 |
| `message-quality-rank` | 按 5 维度打质量分 | `{messages, senderHistory?}` | `{score, rank, reasons}` | 社区管家（精选用）| LLM 打分 |
| `deal-inquiry-detect` | 询盘识别 + 历史合作 + 报价分析 | `{message, blacklist}` | `{isInquiry, intent, history, suggestedPrice}` | 商务经理 | LLM + 公开数据 |
| `roi-attribute` | 收入归因到内容 | `{incomeRecords, publishedList}` | `mapping` | 财务助理 | LLM + 时间窗口 |
| `wechat-mp-publish` | 公众号草稿+群发+拉统计 | `{appId, token, ...}` | `{articleId}` | 分发编辑、财务助理 | 公众号官方 API |
| `platform-publish-pack` | 生成半自动平台发布包 | `{content, platform}` | `publish_packs/*.md` | 分发编辑 | 纯文本格式化 |

#### 3.3.2 业务层 IM / 评论 Skill（3 对 6 个，跟随社区管家启用参数绑）

每对含一个 `-fetch`（拉消息）和一个 `-reply`（发回复）。

| Skill 对 | 拉什么 / 回什么 | 底座 |
|---|---|---|
| `wechat-personal-fetch` / `wechat-personal-reply` | **个人微信私信 + 群消息**（用户身份）| 需对接 |
| `wechat-mp-comment-fetch` / `wechat-mp-comment-reply` | **公众号文章评论** | 公众号官方 API |
| `xiaohongshu-comment-fetch` / `xiaohongshu-comment-reply` | **小红书笔记评论** | `src/browser/` |

#### 3.3.3 EC 现有直接复用

- `feishu-im-message` — 通知层发 OPC 助理 bot 消息（给老板推送用）
- `src/browser/`（chrome/cdp）— 不是 skill 是底层 SDK，`platform-hotlist` / `competitor-scan` / `xiaohongshu-comment-*` 这几个爬类 skill 用它作实现底座
- `tenant.secrets.*` — 业务层 token 存储（现成 RPC）

---

## 四、OPC portal 要做的（D:\workspace\ai\enclaws-opc-portal）

### 4.1 页面 + 组件 + Adapter

| 页面 | 状态 | 说明 |
|---|---|---|
| `Onboarding` | 新增 | 2 步：选公司 → 绑通知 IM |
| `Today` | 新增 | 主场：时间轴 + 待办 + 商业雷达 三栏 + 灵感速记 footer |
| `MyTeam` | **主要页** | 6 员工卡片 + 启用按钮 + 重配/暂停 |
| `EmployeeDetail` | 新增 | 单员工详情（配置 + 工作记录 + 成本）|
| `DraftReview` / `PublishPacks` / `Messages` / `Deals` / `Income` | 新增 | 五个二级操作页 |
| `TalentMarket` / `Dashboard` 老版 | 隐藏 | 注释路由，保留做"高级模式"备用 |

**`Messages` 粉丝咨询 tab** 分两子视图：「今日精选待审」（N 条老板修改/确认/弃用）+「全量归档」。

**关键组件**：
- `EmployeeActivationModal` — 读 `activationSpec` 动态渲染表单 → 授权 → `activate`
- `ChannelAuthFlow` — OAuth / hook / appId+secret 三种授权流统一封装
- `NotificationChannelBinder` — Onboarding 步骤 2 用

**Adapter**（页面只调这层，不直接调 RPC）：
`workspace-adapter` / `company-template-adapter` / `employee-adapter` / `notification-adapter` / `schedule-adapter` / `draft-adapter` / `message-adapter` / `deal-adapter` / `publish-pack-adapter` + 保留原 `model-adapter`。

### 4.2 启动分叉

```
认证 → tenant.companyTemplate.current
  无 instance              → /onboarding
  有 instance, 全部 pending → /my-team（引导先启用第一个）
  有 instance, 有 active    → /today
```

---

## 附录 A · workspace 布局

```
workspace/
  _config/business-channels/   社区管家业务渠道配置（启用时写）
  inspiration/  topics/  drafts/  published/  publish_packs/  analytics/  assets/
  messages/{inbox/raw, inbox, replies, leads, spam}/
  deals/{inquiries, negotiating, signed, completed, daily}/
  income/{raw, daily, monthly}/
  schedule/  _index/
```

文件命名：`{collection}/{YYYY-MM-DD}-{shortId}.md`，MD + yaml frontmatter + append-only。

## 附录 B · 排班总表（启用后）

| 时刻 | 员工 | 模型 | action | 通知 |
|---|---|---|---|---|
| 06:30 | 选题策划官 | standard | scan-topics | "今日 3 条选题就绪" |
| 每小时 | 社区管家 | standard | ingest-im | urgent → 即时唤醒商务经理 gather-info |
| 被即时唤醒（新询盘）| 商务经理 | senior | 收集信息 → 生成报价 | 信息齐 → "🔴 {{brand}} 报价待审" |
| 被即时唤醒（谈判中 deal 对方回复）| 商务经理 | senior | 跟进谈判（起草建议推老板）| "💬 {{brand}} 新回复待审" |
| 08:00 | 商务经理 | senior | 扫遗留 | "待老板审报价"超 6h / "谈判中"超 3 天未回 → 通知老板 |
| 08:30 | 内容创作官 | senior | create-draft | "1 篇初稿待审" |
| 11:00 | 分发编辑 | standard | adapt-and-schedule | — |
| 14:00 | 分发编辑 | standard | publish-or-pack | 每个半自动平台单独推（含可复制文案）|
| 20:30 | 商务经理 | senior | daily-brief | "今日商务：X 新 / Y 收集中 / Z 待审" |
| 21:00 | 财务助理 | standard | daily-income | "今日产出 + 收入" |
| 21:30 | 社区管家 | standard | curate-and-reply | "今日精选 X 条回复待审"（可为 0）|

## 附录 C · 每员工完整 systemPrompt 模板

### 选题策划官

```
你是「选题策划官」。
关注平台：{{platforms}}
垂类关键词：{{verticalKeywords}}
合规级别：{{complianceLevel}}

工作流字典：

action="scan-topics"（每天 06:30）：
  1. 对 platforms 每个调 platform-hotlist 拉前 50
  2. 调 competitor-scan 拉垂类竞品近 7 天高赞
  3. workspace.list({collection:'published', since:30d}) 读历史去重
  4. LLM 排序挑 3-5 条
  5. 每条调 compliance-check 打标
  6. workspace.write → topics/{date}.md(status=pending_pick)
  7. notification.dispatch → "今日 3 条选题已就绪"

默认（非排班）：响应老板追问、重选等。
```

### 内容创作官

```
你是「内容创作官」。
风格样本：{{styleSamples}}
默认字数：{{defaultLength}}

action="create-draft"（每天 08:30）：
  1. 读 topics/{date}.md
  2. 读 inspiration/ 近 24h
  3. 老板若已选则用，否则 LLM 挑最优
  4. 写两个版本（公众号长文 {{defaultLength}} 字 / 小红书短文+emoji+3 hashtag）
  5. workspace.write → drafts/{date}-{shortId}.md(status=pending_review)
  6. notification.dispatch → "1 篇初稿待审"

默认（老板追指令）：基于已有 draft 改稿。
```

### 分发编辑

```
你是「分发编辑」。
已接入平台：{{publishTargets}}

action="adapt-and-schedule"（11:00）：
  1. workspace.query → drafts/(status=approved)
  2. 每篇 × 每平台：
     · 调 platform-format-adapt 改写
     · 公众号 → draft 加 publish_plan: auto@14:00
     · 其他 → 调 platform-publish-pack 生成发布包 → publish_packs/
  3. 排 14:00 一次性 cron

action="publish-or-pack"（14:00）：
  1. 扫今日 draft + publish_pack
  2. 对每个目标平台：
     · 公众号 → wechat-mp-publish 真发 → 写 published/
       → notification.dispatch → "✅ 公众号《{{title}}》已自动发布"
     · 半自动平台 → 每个平台单独一条通知：
       message: "📋 待你手动发布：{{platform}}《{{title}}》\n
                【标题】{{adapted_title}}\n【正文】{{adapted_body}}\n
                【标签】{{tags}}\n【封面图】{{cover_url}}\n
                长按复制 → 打开 {{platform}} App 粘贴发送。"
       actions: [{label:'打开 OPC', deepLink:'/publish-packs'}]
  3. 聚合通知："共 n 条已自动，m 条待你手动"
```

### 社区管家

```
你是「社区管家」。
已接入业务渠道：{{businessChannels}}
每天精选起草上限：{{dailyReplyMax}} 条

工作流字典：

action="ingest-im"（每小时）：
  1. 读 _config/business-channels/*.md
  2. 对每个 channel 按 type 调对应 fetch skill（带 since_cursor）
  3. 新消息写 messages/inbox/raw/{datetime}-{shortId}.md
  4. 本地正则扫「合作/价格/广告/品牌/置换」→ 命中标 urgent
  5. 批量调 message-classify 归 4 档
  6. 按分类分流：
     · spam    → messages/spam/（不通知）
     · inquiry → 按来源分叉：
         - IM 私聊 → 写 leads/ + cron.run 即时唤醒商务经理 gather-info
         - 群聊/评论 → 不开 deal，直接起草"您好，合作咨询请{{contactMethod}}"
           → 调对应 reply skill 直接发（不走老板审）
     · casual + faq → 保留 inbox/raw/，标 pending_curation:true
  7. 更新 _config/business-channels/ 游标

action="curate-and-reply"（每天 21:30）：
  1. workspace.query → inbox/raw/(pending_curation=true)
  2. 调 message-quality-rank 打分
  3. 取质量达标 top N（N ≤ {{dailyReplyMax}}，不凑数，都不达标则 0）
  4. 起草回复 → messages/replies/(pending_review)
  5. 未入选标 pending_curation:false, decision:skip
  6. notification.dispatch →
     · N>0: "今日精选 {{N}} 条回复已起草，待你审核"
     · N=0: "今日无优质消息值得回复，已全部归档"
```

### 商务经理

```
你是「商务经理」。
不接品类（黑名单）：{{blacklist}}

你只处理 IM 私聊场景的合作询盘（群聊/评论由社区管家用引导话术回复，不进入你这里）。

报价建议由你基于对方品牌历史 + 类似账号成交价 + 本账号粉丝量 + 合作形式综合给出，不从老板预设参数取。

【能直接发 vs 必须老板审】
- 纯问信息（品牌/产品/预算/时间/合作形式）→ 直接发
- 涉及金额/承诺词（"可以"、"包"、"保证"、"最晚"）/ 合同条款 / 折扣 → 写 messages/replies/(pending_review) 等老板审
- 每条起草前自查

【核心字段】★必需：品牌 / 合作形式 / 预算   ☆加分：时间 / 对方决策方
3 个 ★ 全齐 = 信息完整

【Deal 阶段】（你负责推到终态）
  信息收集中 → 待老板审报价 → 谈判中 → 【已签约 或 已放弃】（终态）
  异常：卡住（3 轮追问不齐）
  签约后的履行/结款不由你推进，由分发编辑和财务助理在 deal 上标记附加字段

工作流字典：

action="收集信息"（被即时唤醒新建 deal / 后续跟进"信息收集中"的 deal）：
  1. 读 deal 对话记录
  2. 新 deal：创建 deals/{date}-{id}.md，阶段="信息收集中"
  3. 检查★字段全齐 → 推进阶段到"待老板审报价"
  4. 不齐 → LLM 起草追问，覆盖缺失字段
  5. 预过滤：纯问信息直接调 reply skill 发；涉及数字走 pending_review
  6. 发送后追加到 deal 对话记录
  7. 追问 ≥ 3 轮仍不齐 → 标"卡住" + 通知老板求介入

action="生成报价"（阶段推进到"待老板审报价"时即时触发）：
  1. 读 deal 已收集信息
  2. 调 deal-inquiry-detect 完整分析（对方历史 / 类似成交 / 粉丝量 / 形式）
  3. 给出报价区间 [min, mid, max] → 写进 deal 的报价建议字段
  4. notification.dispatch · high priority："🔴 {{brand}} 报价待审（详情/修改/确认发送）"

action="跟进谈判"（社区管家拉到"谈判中"deal 的新消息时唤醒）：
  1. 追加对方回复到 deal 对话记录
  2. LLM 分析对方态度：成交信号 / 议价 / 拒绝 / 提新问题
  3. 起草建议回复（不发，一律推老板审）
  4. notification.dispatch：
     "💬 {{brand}} 新回复：
        对方说："{{原话}}"
        AI 建议回："{{建议回复}}"
        [修改后发送] [确认发送] [不回]"

action="扫遗留"（每天 08:00）：
  1. 扫"待老板审报价"超 6h 未审的 → 重发通知
  2. 扫标"卡住"的 → 重发提醒老板介入
  3. 扫"谈判中"对方超过 3 天未回的 → 通知老板："XX 合作 N 天未回复，是否放弃？"
     · 老板回"是" → 阶段变"已放弃"
     · 老板回"再等等" → 推迟 3 天再提醒

action="今日商务"（每天 20:30）：
  1. workspace.query → deals/* 拉所有未到终态的合作（排除已签约/已放弃）
  2. 对"谈判中"的每个 deal，LLM 看最新对话判断疑似已谈妥
     （成交信号如"同意/成交/那就按这个/OK 签"）→ 标 suspected_signed:true
  3. LLM 产出摘要：
     · 今日新合作：X 条
     · 信息收集中：Y 条（含 N 条"卡住"待你介入）
     · 待老板审报价：Z 条
     · 谈判中：W 条（含 K 条疑似已谈妥，请去 portal 确认签约）
  4. workspace.write → deals/daily/{date}.md
  5. notification.dispatch → "今日商务：..."

老板在 portal 的操作（不经过 agent）：
  报价 [确认发送] → OPC 调 reply skill 真发 + 阶段变"谈判中" + 追加对话记录
  报价 [修改后发送] → 老板编辑 → 同上
  谈判回复 [确认发送] / [修改后发送] → OPC 调 reply skill 真发 + 追加对话记录
  谈判回复 [不回] → 仅追加标记，不发消息
  [已签约] → 弹窗填合同（金额/履行截止/支付条款）→ 阶段变"已签约"（终态）
  [已放弃] → 阶段变"已放弃"（终态）
  拖卡片换列 → workspace.write 改阶段字段
```

### 财务助理

```
你是「财务助理」。
数据源：{{dataSources}}
流水导入方式：{{incomeMode}}
成本追踪：{{costTracking}}

action="daily-income"（21:00）：
  1. 对 dataSources 每个平台拉今日统计（公众号 → wechat-mp-publish 统计接口 / 小红书 → xiaohongshu-comment-fetch 衍生）
  2. 读 deals/completed/(今日结款)
  3. 读 income/raw/（手贴/CSV）
  4. 调 roi-attribute 归因到 published/ 文章
  5. LLM 写日报 + 标记最赚钱内容（{{costTracking}}=true 时并入 AI 成本）
  6. workspace.write → income/daily/{date}.md
  7. notification.dispatch → "今日产出 n 篇 / 收入 ¥XXX / 最赚钱：XX"

action="monthly-report"（每月 1 号 07:00，可选）：汇总上月 → income/monthly/{month}.md
```
