---
type: think
author: zhangbo
created: '2026-04-21T17:43:34+08:00'
index_state: indexed
---
# EC定时任务逻辑整理改造

> 日期：2026-04-21

## 一、问题：定时任务的用户面整体对小白用户不可用

EnClaws 的定时任务页面（`ui/src/ui/views/cron.ts`——含列表 L351-L565 + 表单 L696-L1260；租户端另一份 `ui/src/ui/views/tenant/tenant-cron.ts`）是按工程师思维一字排开的。能跑，但**非技术用户基本跑不下去**。

这个 proposal 覆盖三块：**新增/编辑表单**、**任务总览**、以及一个**被前两者共同卡住的数据模型缺口**——后者不先补，前面的体验优化做一半就落地不了。

---

## 二、新增/编辑表单的痛点

### 2.1 "收件人"只接受 `ou_xxx`，而且**我们根本没那张可选的联系人表**

投递方式选"发布摘要"后，**收件人**这一栏写着"+1555... 或聊天 ID"，实际接受的是：

- 聊天 id（chat id）
- 手机号
- 用户 open id（`ou_xxxxxxxxxxxx`）

**表面问题**：小白用户从哪知道自己的 open id？飞书客户端不露这个字段。现状只有一个 `datalist` 后端补全，取决于后端有没有把历史值塞进去。

**更深的问题**：即便想做一个群/联系人 Picker，**当前数据层也不支持**——详见 §四。没有"这个 bot 在哪些群"、"这个租户下有哪些用户可达"的数据，Picker 列不出任何东西。

### 2.2 "会话 / 主会话 / 隔离会话" 在 SaaS 场景下语义糊掉了

表单里有个 select：

- `主会话`（help: "主会话发布系统事件"）
- `隔离会话`（help: "隔离会话运行独立的代理轮次"）

"系统事件" / "代理轮次"是**实现词**，不是用户词。

更麻烦的是和"执行内容"字段**语义重复**：

| 字段 | 选项 |
|------|------|
| 会话 | 主会话 / 隔离会话 |
| 执行内容 | 发布消息到主时间线 / 运行助手任务（隔离） |

这两个选择**不是独立的**：
- `主会话` + `运行助手任务（隔离）` 逻辑上冲突
- `隔离会话` + `发布消息到主时间线` 也一样别扭

系统其实只要问一件事：**"你是想到点让助手干点活，还是到点发条消息提醒？"** 当前把它拆成两个下拉框是泄漏实现的结果。

更本质的：`sessionTarget` 是**实现层字段**——`main` = "塞进一个已存在的 sessionKey"，`isolated` = "新开一次性会话 + `forceNew:true` + reaper 清理"（[timer.ts:849-931](D:/workspace/ai/EnClaws/src/cron/service/timer.ts#L849-L931) / [isolated-agent/run.ts](D:/workspace/ai/EnClaws/src/cron/isolated-agent/run.ts)）。用户心里想的是"到点发提醒" vs "到点跑任务"，这两件事和"场景 + 收件人"两个字段是一一对应的，`sessionTarget` 能完全推导出来——**不该暴露给用户**，给默认值即可。

SaaS 场景下"主会话"这个概念本身也需要再审——租户 A 眼里的"主会话"到底是哪个会话？当前解析路径（sessionKey → last-active channel 兜底）对"个人用户 ↔ bot 私聊"这种朴素场景直觉性还行，对租户管理员代表组织配的任务就完全对不上。

### 2.3 "已启用" 没有任何说明

默认勾上，放在"代理 ID"右边。新用户看到的疑问：

- 不勾会怎样？草稿？未来启用？
- 时间到了但"已启用"= off，历史上算跑没跑？

一行 help text 就能解决的事，当前完全没有。

### 2.4 "高级" 是杂物抽屉

展开"高级"后并列塞 **10+ 个字段**，彼此条件耦合、作用域各异：

| 字段 | 作用范围 | 默认 |
|------|---------|------|
| 运行后删除 | 全部 | **✓ 勾上** |
| 清除代理覆盖 | 全部 | ✗ |
| 精确时间（无抖动） | 仅 cron 调度 | ✗ |
| 抖动窗口 + 抖动单位 | 仅 cron 且未勾精确 | 空 |
| 模型 | 仅助手任务 | 空 |
| 思考（thinking） | 仅助手任务 | 空 |
| Failure alerts（硬编码英文） | 仅助手任务 | inherit |
| ├─ Alert after / Cooldown / Channel / Alert to | Failure=custom 时 | 混合 |
| 尽力投递 | 投递不为 none 时 | ✗ |

具体坑：

1. **"运行后删除" 默认勾上**——普通用户建个"每天 7 点提醒开会"，保存完下次来找不到（看字面意思以为一次性的会被清理）。默认值 + 命名组合非常容易误伤
2. **Failure alerts 整块硬编码英文**（"Failure alerts" / "Alert after" / "Cooldown (seconds)" / "Alert channel" / "Alert to" / "Inherit global setting" / "Disable for this job" / "Custom per-job settings"），i18n 漏了一大块
3. **"清除代理覆盖"**——不知道"覆盖"从哪来的用户完全理解不了
4. **"抖动窗口 / 抖动单位"**——字面看不出干什么用

### 2.5 其他同级痛点

- **执行内容**："发布消息到主时间线"对小白是黑话
- **唤醒模式**："立即" vs "下次心跳"——"心跳"是实现词
- **cron 表达式**没有下次触发预览
- **超时（秒）**：留空用网关默认，没告诉用户网关默认是多少

---

## 三、任务总览页的痛点

### 3.1 "已禁用"不区分"用户禁用"和"失败自动禁用"

用户视图（`ui/src/ui/views/cron.ts`）里，enabled/disabled 就是一个红/绿 chip——用户主动关掉的任务，和连续失败被系统自动 disable 的任务，在列表上**视觉完全一样**。

租户管理员视图（`ui/src/ui/views/tenant/tenant-cron.ts` L211）已显式标了 `(auto-disabled)` + 连续失败次数——但用户视图这一侧没移植过去。**一侧做了的能力，另一侧漏掉**。

### 3.2 cron 表达式直接暴露 raw 字符串

列表里 `Cron 0 7 * * * (America/Los_Angeles)` —— 对会写 cron 的人没问题；对业务用户——像系统错误。配合表单里没有下次触发预览，用户永远没法从界面确认任务会不会真的每天 7 点跑。

### 3.3 "Skipped" 状态不解释语义

`lastStatus` 有 `OK` / `Error` / `Skipped`。"跳过"什么意思？被锁挡住？前一次还没跑完？点进去也没解释。

### 3.4 空状态不引导

列表为空显示 `"No matching jobs."`——没 CTA、没示例。

### 3.5 无批量操作

没多选、没批量启用/禁用/删除。清理陈旧任务只能一条条点。

### 3.6 列表 + 表单共页，状态不清

左右分栏：选任务切到编辑模式。副作用：

- 编辑到一半切另一个任务——**未保存的改动直接被覆盖**，没 confirm
- 没有独立的"新增路由" / "编辑路由"
- 表单状态持久化边界对用户不可见

### 3.7 用户视图和租户管理员视图是**两份独立实现**

整个系统是 SaaS 多租户，不存在"单租户模式"。但 cron 的两个面向（**用户看自己的任务** / **租户管理员看全租户任务**）在代码里分成了 `ui/src/ui/views/cron.ts` 和 `ui/src/ui/views/tenant/tenant-cron.ts` 两份独立实现，功能不对齐：

| 能力 | 用户视图 | 租户管理员视图 |
|------|---------|-------------|
| 显示 `(auto-disabled)` | ✗ | ✓ |
| 连续失败次数 | ✗ | ✓ |
| Edit / Clone / Run / History | ✓ | ✗（仅 Disable + Remove） |
| agent 维度过滤 | ✗ | ✓ |

同一个 bug 要改两遍、一侧加了新能力另一侧长期落下、双份测试。

### 3.8 Run log 里会露出内部 `jobId`

`renderRun` 里 `entry.jobName ?? entry.jobId`——job 被删后 run 还在，fallback 直接显示内部 id 字符串。

---

## 四、底层数据模型缺口（阻塞 §2.1 的根因）

### 4.1 现在存了什么

| 层面 | 现状 |
|------|------|
| `users` 表 | `src/db/sqlite/schema-sql.ts`：`id / tenant_id / email / password_hash / open_ids (JSON 数组) / union_id / channel_id / display_name / role / status ...`——一张表同时承载控制台用户和飞书用户，但身份字段按来源分布：控制台登录用户 `password_hash` 非空、`open_ids = []`、`union_id = NULL`；飞书用户 `password_hash = NULL`、`open_ids` 有值 |
| OAuth token | `StoredUAToken`（`extensions/openclaw-lark/src/core/token-store.ts` L37-46）：键 `{appId}:{userOpenId}`，存 access/refresh token 及作用域 |
| 消息上下文 | `MessageContext`（`extensions/openclaw-lark/src/messaging/types.ts` L155-185）：含 `chatId` / `chatType` / `senderId` / `appId`——但**每条消息临时构造，不落库** |
| 聊天信息 | `chat-info-cache.ts`：1 小时 LRU，按需调 `im.chat.get`——**进程内缓存，不落库** |
| Bot 成员变化 | `event-handlers.ts` L276-289 `handleBotMembershipEvent()`——**只打日志，什么都没做** |
| 隐式账号关联 | `src/db/models/user.ts` L210-250：飞书消息到达时若 `union_id` 匹配已有用户，会把 `open_id` append 进 `open_ids` 数组——但**没有主动绑定 UI，控制台用户在没收到过飞书消息前 `open_ids` 一直是空** |

### 4.2 缺什么

**关系维度（真的缺）**：

- **"这个 bot 在哪些群里"**——完全没表。租户管理员想在 Picker 里选个群发通知，我们列不出选项
- **"这个群里有哪些成员"**——完全没表。想在群里挑"@ 某人"也列不出
- **"bot ↔ chat 活跃时间"**——无法做"最近 30 天没人说话的群"这种 stale 清理

**身份打通（更大的坑）**：

- 控制台用户在首次被飞书消息触达前 **`open_ids = []`**——意味着租户管理员登录进来建 cron 任务时，系统根本不知道"我"在飞书是谁
- 没有主动"绑定飞书账号"的 UI 入口；现有的隐式 append 只在"飞书 → 控制台"这个方向上起作用，而 cron 的场景是"控制台 → 飞书"

这直接影响 §6.2 Picker 里的"发给我自己"——控制台用户大概率**没有可用的 open_id**，这个快捷入口对他们直接断掉。

### 4.3 `CronJob.createdBy` 只存 console user id

`src/cron/types.ts` L127：`createdBy?: { userId: string; displayName?: string }`——任务**只记控制台用户 id，不记 open_id**。这是合理的（cron 是控制台功能），但也意味着从 cron 任务反解"创建者的飞书身份"必须经过 `users.open_ids`，而这恰恰是可能为空的那一列。

### 4.4 这些信息其实每条消息都经过

关键路径：`extensions/openclaw-lark/src/messaging/inbound/dispatch-context.ts` L73-145 `buildDispatchContext()`。在 sessionKey 构造那一刻（L97-105），**全部字段都在手里**：

| 字段 | 来源 |
|------|------|
| `app_id` | `account.appId` |
| `account_id` | `account.accountId` |
| `chat_id` | `ctx.chatId` |
| `chat_type` | `ctx.chatType`（`p2p` / `group`） |
| `user_open_id` | `ctx.senderId` |
| `message_id` | `ctx.messageId` |
| `thread_id` | `ctx.threadId`（可选） |

sessionKey 本身的格式是 `agent:<agentId>:<channel>:<peerKind>:<peerId>`（`src/routing/session-key.ts` L115-162），group 形态下 `peerId` = `<chatId>:sender:<senderOpenId>`，direct 形态下 `peerId` = `<senderOpenId>`。这些信息一条不少地流过，但**没人接**。

### 4.5 建议补的数据模型

最小可用两张**关系表**（不重建已有 `users`，只补空白）：

**① `bot_chats`**——bot ↔ 群/DM 的成员关系（新增）

| 字段 | 说明 |
|------|------|
| `tenant_id` | 租户 |
| `app_id` | bot |
| `chat_id` | 群或 DM 标识 |
| `chat_type` | `p2p` / `group` |
| `chat_title` | 群名（按需从 `chat-info-cache` 回填） |
| `state` | `active` / `left` / `removed` |
| `first_seen_at` / `last_seen_at` | 活跃时间 |
| `joined_via` | `bot_added_event` / `inbound_message` / `backfill` |

**② `chat_members`**——chat ↔ user 的成员关系（新增）

| 字段 | 说明 |
|------|------|
| `tenant_id` | 租户 |
| `chat_id` | 群或 DM |
| `user_open_id` | 用户 open_id |
| `user_id` | 关联到 `users.id`（可空——只有绑定过才有） |
| `first_seen_at` / `last_seen_at` | 活跃时间 |

**"bot 能触达哪些用户"** 不需要新表，用 `chat_members JOIN bot_chats ON chat_id` 就能拿到。旧的 `users.open_ids[]` 继续作为用户的飞书身份集合，不动。

### 4.6 回填通道（多层，不能只靠事件）

查了代码后结论是：**单纯监听事件不够**——`im.chat.member.bot.deleted_v1` 事件已订阅（`extensions/openclaw-lark/src/channel/monitor.ts:106-107`），scope 也够（`im:chat:read` / `im:chat:update` 在 `tool-scopes.ts:350-371` 的 `REQUIRED_APP_SCOPES` 里），但**至少四种情况会让 EC 不知道 bot 已被踢**：

1. **EC 进程当时挂着**——飞书的事件投递无重试、无 replay；进程起来时事件已过期
2. **`im.chat.member.user.deleted_v1` 根本没订阅**——用户退群完全不 tracked，`chat_members` 会长期 stale
3. **发消息失败时不回查**——`sendTextLark` / `sendCardLark` 拿到 4xx "bot not in chat" 错误只打日志，`chat-info-cache` 也不会因此 invalidate
4. **冷启动没有基线**——新接入 bot 时，之前就在的群没有任何触发事件，除非有人说话否则我们一直不知道这个 bot 在哪些群

因此回填必须是**多层**组合：

| 层 | 时机 | 实现 | 覆盖的失败模式 |
|----|------|------|-------------|
| **A. 被动 upsert（主力）** | 每条入站消息 | 在 `buildDispatchContext` sessionKey 构造后（L105 之后），upsert `bot_chats` + `chat_members` | 日常活跃流量 |
| **B. 事件订阅** | bot 加/退群、用户加/退群 | 把 `handleBotMembershipEvent`（L276-289）从"打日志"改成 upsert `bot_chats.state`；**新加**订阅 `im.chat.member.user.deleted_v1` 处理 `chat_members` | 实时群成员变化 |
| **C. 发送失败反查** | `sendTextLark` / `sendCardLark` 返回 4xx 特定错误码 | 在送达层 catch 到 "bot not in chat" 类错误时，标记 `bot_chats.state='removed'`，失效 `chat-info-cache` 对应条目 | EC 离线期间的事件丢失 |
| **D. 启动期 reconciliation** | 进程启动、WebSocket 重连后一次 | 调 `im.chat.list` 对齐 bot 实际在的群列表；多出来的标 `removed`，少了的补进来 | 冷启动基线、长时间离线 |
| **E. 一次性 backfill** | 本 proposal 落地时 | 扫现有 session/cron job 的 sessionKey 反解填表；`chat-info-cache` 命中过的回填 `chat_title` | 存量历史数据 |

**注意**：A + B 是"保持新鲜"，C + D 是"修错"。缺少 C/D 的话，离线期间的状态漂移会一直累积——这也是为什么原方案"改一行 handler 就够"是错的。

### 4.7 `im.chat.member.user.deleted_v1` 事件 —— 目前完全漏监听

（原 §4.7 内容前置说明）`extensions/openclaw-lark/src/channel/monitor.ts` 只订阅了 bot 的加/退群事件（L106-107），**用户加/退群事件一个没订阅**。这意味着 `chat_members` 只能由被动 upsert（层 A）刷新——一个用户从群里退了，我们永远不知道。

修法：在 `monitor.ts` 的 handler map 里加 `im.chat.member.user.deleted_v1` 和 `im.chat.member.user.added_v1`，路由到 `handleChatUserMembershipEvent`（新写）去 upsert `chat_members`。Scope 已经覆盖，不需要重新申请。

### 4.8 "控制台用户如何关联飞书身份"——相邻问题，但不阻塞本 proposal

上面两张表解决了 **bot/chat/user 维度**的空白，但**没解决控制台用户 ↔ 飞书身份**的绑定空白。目前唯一的绑定路径是"飞书消息到达 → 按 `union_id` 匹配 → append `open_id`"——单向且被动。

好消息是：**"发给我自己"这个高频场景不依赖它**——直接走 `session=main` 投递到控制台聊天框即可（详见 §6.2），控制台用户登录进来天然能看到。

仍然会用到身份绑定的场景（本 proposal 范围之外，列出供参考）：

- 用户希望"控制台 + 飞书 DM 两路都收到"（本 proposal 在 Picker 里留个可选 checkbox，`open_ids` 非空时才亮）
- 跨渠道 @me 通知、面向本人的告警投递（例如 §6.7 自动禁用告警）
- 租户管理员主动把某个"组织里的同事"指到某条 cron 任务——需要从飞书组织架构反查 open_id

这些可统一由后续一个"控制台绑定飞书账号"的 OAuth 流解决（复用现有 OAuth token 基础设施）。**本 proposal 明确不做**，只在 Picker 里留好条件展示的占位，不让缺失的绑定挡住主路径。

---

## 五、目标

1. 让**完全不懂 "agent / session / heartbeat / cron expression"** 的业务用户，不问工程师自助完成"每天早上 9 点让助手发会议提醒到我们群"这种场景
2. 总览页一眼看出任务健康度（哪些挂了、哪些被系统自动关了）
3. 补上 bot ↔ chat ↔ user 的持久化关系，让"选谁发、发到哪个群"有真数据源
4. 两个视图（用户视图 / 租户管理员视图）功能对齐，不再两份实现
5. 不牺牲高级用户

---

## 六、方案

### 6.1 【前置】补上 bot ↔ chat ↔ user 数据模型

按 §4.4 / §4.5 落地：

- 在 `buildDispatchContext` 的 sessionKey 构造点之后加 upsert 钩子
- 把 `handleBotMembershipEvent()` 从 no-op 改成事件驱动 upsert
- 一次性 backfill 脚本扫历史 sessionKey

**这是 §6.2 Picker 的硬前置**——否则 Picker 只能拿 `datalist` 历史值，和现状没区别。

### 6.2 表单：收件人 Picker

基于 §6.1 建的表，把"收件人" `<input>` 改成 Picker：

- **群/频道**：`bot_chats WHERE tenant_id = ? AND state = 'active'` 按 `last_seen_at` 排序
- **联系人**：`chat_members JOIN bot_chats` 取"本租户下 bot 可触达的 open_id"，再 LEFT JOIN `users` 拿 `display_name`
- **"发给我自己"**：**默认置顶且对所有控制台用户可用**——映射到 `session=main` + payload=systemEvent，消息投递到用户自己的控制台聊天框（main timeline）。**不依赖飞书绑定**，因为消息的落地位置就是控制台里的那个会话，用户登录进来天然能看到。若用户另外希望"也在飞书 DM 里收到一份"，才需要飞书身份绑定——作为可选增强，放在按钮旁一个"同时在飞书 DM 我"的小 checkbox，`users.open_ids` 非空时才亮
- 支持搜索（群名、昵称）
- 底部保留"高级：手动输入 chat id / phone / open id"，能力不丢
- 多 bot 租户按 `app_id` 分组显示（让用户清楚"这条消息会由哪个 bot 发"）

**前置依赖清晰化**：Picker 本身上线 = `bot_chats` + `chat_members` 两表落地（§6.1），**不依赖**控制台 ↔ 飞书绑定。"发给我自己"这条核心快捷入口也**不依赖**——走控制台聊天框路径，控制台用户无缝可用；Feishu DM 双通道只是可选增强。

### 6.3 表单：用"场景模式"替代一字排开

进入页面先问：**"你想定时做什么？"** 回到 `payload × session × delivery` 三轴组合，覆盖实际用户意图至少需要 **5 个场景卡**（原先三个合并掉了"助手后台任务"和"助手 + webhook"两类真实需求）：

| 场景卡 | 用户看到 | 典型用例 | 底层映射 |
|-------|---------|---------|---------|
| ① **定时提醒** | "到点发一条消息给某人/某群/我自己" | "每天 7 点提醒我们群开会" | payload=systemEvent, delivery=announce, recipient=? |
| ② **定时助手汇报** | "到点让助手跑任务，完成后把结果发出来" | "每天早上助手汇总昨天团队动态发到群" | payload=agentTurn, session=isolated, delivery=announce |
| ③ **定时助手后台任务** | "到点让助手跑任务，做完就行，不用通知" | "每天凌晨清理/刷新索引/跑维护脚本" | payload=agentTurn, session=isolated, delivery=none |
| ④ **定时 webhook 推送** | "到点 POST 一条固定内容到外部 URL" | "每小时给外部监控打一次心跳" | payload=systemEvent, delivery=webhook |
| ⑤ **定时助手 → webhook** | "到点让助手生成结果，POST 到外部系统" | "每天助手分析数据后推到 BI 系统" | payload=agentTurn, session=isolated, delivery=webhook |

选完场景只显示该场景需要的字段。当前"会话 / 执行内容 / 结果投递"三个重复选择折叠成场景。

**`sessionTarget` 完全不展示**——按上表从场景自动推。用户永远看不见"主会话 / 隔离会话"这两个词。后端/高级调试需要时仍可通过 API 直填，但 UI 层不提供 override（场景 ≠ 枚举选项的概念，没有"场景=高级"这一档留给暴露 sessionTarget）。

**一次性任务不是独立场景**——"10 分钟后提醒我一次"是 ① 配 `schedule=at` + `运行后删除=✓`，场景不变，只是周期改一次性。

**场景卡集合要不要就这 5 个**：上面的划分是按 form 字段的组合穷举出来的实际意义上不同的 case。但哪些该作为主卡片、哪些收进"高级 / 其他"，最好用真实用户行为数据验证（目前看 ①②④ 应该是绝大部分流量，③⑤ 偏工程场景）。这个集合在 §八 列为待讨论。

### 6.4 表单：给每个字段配人话

| 不懂的词 | 改成 |
|---------|------|
| 主会话 / 隔离会话 | **不暴露**，由场景自动推（①④→main / ②③⑤→isolated） |
| 系统事件 | "一条消息" |
| 代理轮次 | "一次助手任务" |
| 立即 / 下次心跳 | "马上触发" / "等下一个周期"（收进高级） |
| 心跳（heartbeat） | 不暴露给用户 |
| 代理覆盖 | "忽略本任务的助手设置，用网关默认助手" |
| jitter / 抖动 | "多个任务错峰执行，避免同一秒一起跑" |

特别修：

- "已启用"加 help："关闭后不触发，但任务保留"
- "运行后删除" 默认 **✓ 改成 ✗**，文案改成"这是一次性任务，执行完自动清理"
- Failure alerts 整块补 i18n，改放到"进阶 → 失败告警"子折叠

### 6.5 表单：高级字段三组子折叠

```
▸ 高级
   ▸ 执行控制     （超时、模型、思考、清除代理覆盖）
   ▸ 调度微调     （精确时间、抖动窗口）
   ▸ 失败与告警    （failure alerts + 尽力投递）
```

### 6.6 表单：cron 表达式加"下次触发"预览

填完实时显示：

```
下次触发：2026-04-22 07:00:00 (Asia/Shanghai)
之后 3 次：04-23 07:00 / 04-24 07:00 / 04-25 07:00
```

croner 本来就能算 next occurrence，零成本。

### 6.7 总览页：禁用状态显式三态

| 状态 | 展示 |
|------|------|
| 正常 | 绿 "已启用" |
| 用户禁用 | 灰 "已停用" |
| 失败自动禁用 | **红 "自动停用（连续失败 N 次）"** + 点击看详情 |

自动禁用还应首页 banner 告警——当前用户进总览才知道任务被关了。

### 6.8 总览页：cron 表达式人话化

列表里 `Cron 0 7 * * *` 改成：

```
每天 07:00（Asia/Shanghai）
原表达式：0 7 * * *      ← hover 可见
```

### 6.9 总览页：Skipped 加解释、空状态加 CTA、最小批量操作

- `Skipped` pill hover / 点击显示原因（上次还在跑 / 被锁挡住 / 其他）
- 空状态：`"还没有定时任务。试试这些常见场景："` + §6.3 三场景卡
- 多选 + 批量禁用/删除（只覆盖"清理一批陈旧任务"场景）

### 6.10 总览页：编辑未保存切换拦截 + 路由分离

切任务前检测 dirty state 弹 confirm；长期考虑把编辑拆成独立路由 / 弹窗。

### 6.11 总览页：合并两份实现

`cron.ts` + `tenant/tenant-cron.ts` 收敛为**一个 list 组件 + 能力白名单**：

```ts
<CronList
  mode="self" | "tenant-admin"
  actions={['edit', 'run', 'history', 'disable', 'remove']}
  columns={['name', 'schedule', 'agent', 'state', 'autoDisabled', ...]}
/>
```

### 6.12 "主会话" 语义在 SaaS 下的歧义

前提：UI 层已不暴露 `sessionTarget`（§6.3 / §6.4），这里讨论的是**后端推定逻辑**本身——场景 ①（定时提醒）自动推 `main` 时，"塞进哪个已存在 sessionKey"怎么确定？

- **方案 A**：跟当前收件人字段走——Picker 选的群/用户对应哪个 sessionKey 就是哪个（`agent:<agentId>:<channel>:<peerKind>:<peerId>` 按 §4.4 组装）。自然、可解释
- **方案 B**：租户管理员建的任务 fallback 到"租户默认通知频道"（复用 LLM Provider proposal §5.2 "租户通知频道"一等公民）——只在 Picker 留空或"发给租户默认频道"时触发

倾向 A 为主，B 作为 fallback——收件人非空时永远用收件人对应的 sessionKey，别再走 last-active 兜底那套歧义解析。

---

## 七、实施路线

| 阶段 | 内容 | 工作量 |
|-----|------|-------|
| **P0（前置）** | `bot_chats` + `chat_members` 表结构 + 迁移脚本 | 小 |
| **P0（前置）** | `buildDispatchContext` 被动 upsert 钩子（§4.6 层 A） | 小 |
| **P0（前置）** | `handleBotMembershipEvent` 从 no-op 改为 upsert + 新订阅 `im.chat.member.user.{added,deleted}_v1`（§4.6 层 B / §4.7） | 中 |
| **P0（前置）** | 历史 sessionKey 一次性 backfill 脚本（§4.6 层 E） | 小 |
| P1 | 送达层 4xx "bot not in chat" 反查标 `removed`（§4.6 层 C） | 小 |
| P1 | 启动期 / WebSocket 重连后调 `im.chat.list` 做 reconciliation（§4.6 层 D） | 中 |
| P0 | 收件人 Picker 前端（§6.2） | 中 |
| P0 | 用户视图补 `(auto-disabled)` + 连续失败次数（§6.7） | 小 |
| P0 | cron 表达式人话化（§6.8） | 小 |
| P0 | "已启用" / "运行后删除" 文案 + 默认值（§6.4） | 小 |
| P0 | Failure alerts 补 i18n（§6.4） | 小 |
| P0 | 场景卡入口 + 字段可见性收敛（§6.3） | 中 |
| P0 | Cron 下次触发预览（§6.6） | 小 |
| P0 | 空状态 CTA + 编辑未保存拦截（§6.9 / §6.10） | 小 |
| P1 | 联系人 Picker 接入 `im.user.get` 补 display_name（§6.2） | 小 |
| P1 | 高级字段三组子折叠（§6.5） | 小 |
| P1 | Skipped 原因解释、最小批量操作（§6.9） | 小 |
| P1 | 唤醒模式 / 心跳改人话或收进高级（§6.4） | 小 |
| P1 | `主会话` 语义决策落地（§6.12） | 取决于选型 |
| P2 | 合并两份 list 实现（§6.11） | 大 |
| P2 | "最近使用收件人"本地持久化 + 自动禁用首页 banner | 小 |

P0 完成后，"每天 7 点提醒我们群开会"从"需要翻工程师问"降到"两分钟自助完成"，且总览页能一眼看出任务健康度。

---

## 八、待讨论问题

1. **`bot_chats` / `chat_members` 是新建表还是扩进既有 schema**：倾向新建关系表——`users.open_ids[]` 已承担"用户身份集合"语义，不适合再叠"成员关系"；但 DB 审美意见可以讨论
2. **控制台 ↔ 飞书绑定 UX 是否在本 proposal 范围内**：我倾向**放到后续 proposal**——本轮靠"发给我自己 = 落到控制台聊天框"绕过了绑定依赖，绑定只是"想同时飞书 DM"的可选增强。但绑定对其他功能（跨渠道告警、反查组织成员）仍是共同前置，需要确认拆分可接受
3. **"主会话" 保留还是禁用**：倾向禁用（§6.12 方案 B），涉及存量用户迁移，需产品拍板
4. **飞书组织架构 API 的接入深度**：Picker 只用我们自己积累的 `chat_members`（被动收集，冷启动期空）还是主动拉组织架构全量？主动拉要额外授权范围
5. **backfill 脚本的运行时机**：部署即跑？还是迁移任务里手动触发？历史 sessionKey 数量可能大
6. **"场景卡"是 P0 必做还是可选**：不做场景卡字段隐藏只能靠 select 联动，信息架构上仍是工程师思维；做意味着表单组件级改造
7. **场景卡具体收多少个、哪些进主卡片、哪些收进"其他"**：当前 §6.3 列了 5 个是按 form 字段组合穷举的（①提醒 / ②助手汇报 / ③助手后台 / ④webhook / ⑤助手→webhook）。理论上还能叠"一次性 vs 重复"等维度，但那些是调度类型而不是场景。希望用真实用户流量数据验证主卡集合——有没有现成后台数据可看（现有 cron job 的 payload/delivery 组合分布）
8. **"运行后删除" 默认值翻转**：行为变更，需确认有没有脚本依赖"建了就跑一次"的隐含约定
9. **合并两份 list 实现的时机**：现在上 P2 牵动较大，是先把用户视图对齐到租户管理员视图的能力（短期冗余）再合并，还是 P2 一次性做
10. **"自动停用"的恢复路径**：用户点"重新启用"直接恢复？还是强制先看一眼失败历史
11. **cron 人话化的 i18n 策略**：cronstrue 多语言颗粒度粗；要不要自己维护常见 pattern 的 zh-CN 映射
12. **批量操作的权限边界**：租户管理员能批量删除租户下所有任务吗？还是只能删本人创建的
13. **bot 被踢出群后 ① 检测层优先级**：§4.6 列了 A-E 五层，MVP 够不够？倾向 A + B + E（被动 upsert + 事件订阅 + 一次性 backfill）先上，C（发送失败反查）作为 P1，D（启动期 reconciliation）看冷启动场景频率再定；但"离线期间状态漂移"在生产环境多严重没数据，判断可能偏乐观
14. **bot 被踢出群后 ② 下游响应**：检测到 `state='removed'` 后，Picker 立即隐藏是共识；已经 reference 这个 chat 的 cron job 是自动禁用（对齐"失败自动禁用"视觉，§6.7 红标）还是保留任务只 disable 投递？涉及用户体感——任务消失 vs 任务保留但不 work，哪种更不反直觉

（proposal 结束）
