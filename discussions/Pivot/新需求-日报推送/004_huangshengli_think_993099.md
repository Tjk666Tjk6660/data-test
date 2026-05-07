---
type: think
author: huangshengli
created: '2026-04-28T15:39:26+08:00'
index_state: indexed
---
# Pivot 团队日报 · 产品设计初步构想

## 一、项目背景

Pivot 系统已经把团队的工作活动结构化为 matter 模型(每件事一条时间线 + 多类型文件 + 6 状态机),代码协作天然在 git 仓库里有完整记录。**两套数据已经是结构化事实,不需要人重复口头同步**。

## 二、项目目标

把这两套已结构化的工作事实,每天自动整理成一份团队日报,推送到飞书群。

日报要呈现的内容:

| 维度 | 回答什么 |
|---|---|
| 团队整体 | 推进了哪些 matter、产出了多少代码、哪些事完成了 |
| 个人 | 谁做了什么、产出形态(think / act / verify / result / insight)、协作密度 |
| 异常 | 谁今天没产出、哪些 commit 归属不明 |
| 评价 | AI 给团队和个人的 5 星评分 + 4 维度子分 + 一句话点评 |

**MVP 定位**:

本次实施是 **MVP**——先跑出一个能用的版本,推到生产收集真实使用反馈,再决定后续优化方向。**不是**一份方方面面都考虑完备的成品产品。

具体含义:

- 评分维度、卡片格式、推送时机这些都先按当前设计跑一周/两周,根据接收方实际反馈再调
- 文档里标出来的若干 trade-off(见 §六)都是有意保留的简化,**不要**在 MVP 阶段就预先优化
- 后续迭代方向(见 §七)是参考,不是承诺;真实需求要从生产反馈里来,不是评审会议上想象出来的

## 三、工作区间设计

日报脚本运行在 Pivot 生产服务器本机,大部分读取动作是本地文件系统或本地 SQLite。**唯一需要网络的是**:(1) 拉代码仓库最新 commit history、(2) 调 AI 模型评分、(3) 推飞书卡片。

### 3.1 资源访问表

| 资源 | 位置 | 访问方式 | 读 / 写 |
|---|---|---|---|
| 代码仓库 (team-pivot-web) | **独立** mirror workspace,默认 `/opt/team-pivot-web/var/code-mirror/team-pivot-web/` (见 §3.3) | `git fetch --all --prune` 拉最新 ref → `git log --all` 跑历史 | 只读 (不 push) |
| 数据仓库 matter index | `/opt/team-pivot-web/var/git/<repo>/index/*.index.yaml` | 文件系统 glob + yaml 解析 | 只读 |
| 用户表 | `/opt/team-pivot-web/var/data.db` (SQLite) | URI `mode=ro` 只读连接 | 只读 |
| 系统配置 | `data.db` 的 `settings` 表 (日报相关键见 §4.7) | 同上,只读 | 只读 |
| AI 模型 | OpenAI 兼容端点 (URL / key 来自 settings) | HTTPS POST | 只读(发请求) |
| 飞书 IM | 飞书 API (token 来自 `FeishuTokenManager`) | HTTPS POST 卡片消息 | 写(发群消息) |
| 日报运行日志 | `/opt/team-pivot-web/var/log/daily-report.log` | systemd 重定向 stdout/stderr | 写(append) |

### 3.2 进程隔离

- 日报脚本是**独立一次性进程**(通过 systemd timer 调起),与 Pivot 主服务(`team-pivot-web.service`)在同一台机器上但不共用进程。
- 主服务挂掉不影响日报触发;日报脚本崩掉不会拖累主服务。
- 数据仓库 git 工作树是**只读使用**(脚本不会写数据仓库 / 不会触发任何 commit),与主服务的 `Workspace.write_session` 写路径完全无锁竞争。
- 用户表用 SQLite URI `mode=ro` 打开,**无法**通过这个连接发起任何 DDL/DML(参考迁移脚本 `_ReadOnlyUserView` 同一安全模式)。

### 3.3 代码仓库 mirror 工作区间设计

为日报功能单独维护一份**代码仓库 mirror**,只用于读 commit history,与部署流程完全解耦。

```
/opt/team-pivot-web/
├── var/
│   ├── data.db                          ← Pivot SQLite
│   ├── git/<repo>/                      ← Pivot 数据 workspace (matter index)
│   ├── code-mirror/
│   │   └── team-pivot-web/              ← ★ 代码仓库 mirror (新增)
│   │       └── .git/                    ← 完整 git history
│   └── log/daily-report.log
└── ...                                  ← 部署的应用代码 (无 .git)
```

**setup**(部署管理员一次性手工做):

```bash
# 用一个 readonly PAT (scope: contents:read on team-pivot-web)
mkdir -p /opt/team-pivot-web/var/code-mirror
cd /opt/team-pivot-web/var/code-mirror
git clone https://<readonly_token>@github.com/<org>/team-pivot-web.git
# .git/config 里就嵌入了 token,后续 fetch 自动用,日报脚本不参与认证
```

**日报每次运行时**:

```bash
git -C /opt/team-pivot-web/var/code-mirror/team-pivot-web fetch --all --prune
git -C /opt/team-pivot-web/var/code-mirror/team-pivot-web log --all --since=... --until=...
```

只 `fetch` 不 `pull`/`checkout`——日报只关心 ref history,**不需要更新 working tree**。fetch 得到的是所有分支(包含 `--prune` 清掉远端已删的 stale 分支)。

**fetch 失败的容错**:

- 网络故障 / token 过期 → 日报继续,基于上次 fetch 拿到的 commit history 做评估,卡片明显标 `⚠️ 代码仓库未刷新(原因),commits 数据可能滞后`
- 不让 fetch 失败导致日报整体停发——管理员通过卡片提示就能看到异常并修复

**path 是否可配**:

通过 SQLite settings 的 `daily_report.code_repo_dir` 配置,默认 `/opt/team-pivot-web/var/code-mirror/team-pivot-web/`。允许调整路径(比如运维想放别的盘)。

**security**:

- mirror 用**只读** PAT(scope `contents:read` on team-pivot-web),与 pivot-database 的 write_token 是不同的 token。即便日报脚本被攻陷,这个 token 也不能 push / 改任何代码
- mirror 不暴露给主服务、不暴露给 web 用户,只有日报脚本读它

## 四、工作原理

### 4.1 总流程

```
[09:30 systemd timer 触发]
       │
       ▼
┌──────────────────────────────────┐
│ 1. bootstrap                      │
│    读 .env / 打开 SQLite (ro)     │
│    定位 workspace + 代码 mirror   │
└──────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────┐
│ 2. git -C <code_mirror> fetch    │
│    --all --prune (拉最新 ref)    │
│    失败 → 容错继续,卡片标提示     │
└──────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────┐         ┌──────────────────────────┐
│ 3a. 扫 matter index 文件           │         │ 3b. 跑 git log --all      │
│    解析 timeline 事件 + comments   │         │    24h 窗口内 commits     │
│    按 created_at 过滤到 24h 窗口  │         │    含 shortstat (+/-行数) │
└──────────────────────────────────┘         └──────────────────────────┘
       │                                              │
       │                                              ▼
       │                                ┌──────────────────────────┐
       │                                │ 3c. commit author 8 级匹配│
       │                                │    → users.pinyin 或 桶  │
       │                                └──────────────────────────┘
       └──────────────────┬───────────────────┘
                          ▼
            ┌──────────────────────────────┐
            │ 4. 按 pinyin 聚合              │
            │    UserActivity[] + 团队总览  │
            └──────────────────────────────┘
                          │
                          ▼
            ┌──────────────────────────────┐
            │ 5. 喂 AI: 精简结构化 JSON      │
            │    AI 返回评分 + 总结 + 高光  │
            │    AI 失败 → 降级,纯统计       │
            └──────────────────────────────┘
                          │
                          ▼
            ┌──────────────────────────────┐
            │ 6. 渲染飞书卡片 dict          │
            │    复用 _card_shell 框架      │
            └──────────────────────────────┘
                          │
                          ▼
            ┌──────────────────────────────┐
            │ 7. 广播到所有飞书群            │
            │    (FeishuNotifier._broadcast)│
            └──────────────────────────────┘
```

### 4.2 数据仓库读取(matter index)

**扫描**:

```python
for index_path in (workspace / "index").glob("*.index.yaml"):
    data = read_matter_index(index_path)        # dict
    for item in data.get("timeline") or []:
        # 每个 item 是 timeline 上的一篇文件
        ...
```

**每个 timeline item 关注的字段**:

| 字段 | 用途 |
|---|---|
| `file` | 文件相对路径(也是该项在 timeline 上的唯一身份) |
| `created_at` | 创建时间(ISO 字符串,带时区);**用于窗口过滤** |
| `creator` | 创建人(pinyin 或 open_id 兜底);用于按用户归属 |
| `owner` | 责任人(pinyin 或 open_id);act 类型经常 `creator != owner` |
| `type` | `think / act / verify / result / insight` |
| `summary` | 一行摘要(给 AI 评分时作为内容描述) |
| `status_change` | `{from, to}`(若该文件触发了 matter 状态变更);用于"推进结果"维度 |
| `verifications` | verify 类型的字段:`[{target, judgement, comment}]`;用于"协作密度" + "推进结果" |
| `verifications_received` | act 上由 verify 反向写入;辅助计数 |
| `outcome` | result 类型的字段(`finished / cancelled`) |
| `comments[]` | 评论流;**每条 comment 单独按 created_at 过滤窗口**(评论可能远晚于文件创建) |

**过滤规则**:

```
event ∈ window  iff  event.created_at ∈ [window.since, window.until)

对每个 timeline item:
  - 若 item.created_at ∈ window:item 整条进 events
  - 若 item.created_at ∉ window:仍要扫 item.comments[],
    把 c.created_at ∈ window 的评论作为"独立事件"附在 item 上
    (这样别人在我老 matter 上留的评论也能算今日活动)
```

**`creator/owner == "ou_xxx"`**(未注册联系人,见 `publish.py::_resolve_owner_for_index`)的事件,计入团队总览统计但**不归属到任何已注册用户**——日报里的"个人列表"只覆盖 `users` 表里的注册用户。

### 4.3 代码仓库读取(git log)

**命令**(在 §3.3 定义的 code mirror workspace 里跑,**不是** `/opt/team-pivot-web` 本身):

```bash
# 起手先 fetch 拉最新所有分支(含 --prune 清 stale remote)
git -C /opt/team-pivot-web/var/code-mirror/team-pivot-web fetch --all --prune

# 然后取 24h 窗口内所有分支的 commits
git -C /opt/team-pivot-web/var/code-mirror/team-pivot-web log --all \
    --since='2026-04-26T09:30:00+08:00' \
    --until='2026-04-27T09:30:00+08:00' \
    --pretty=format:'%H|%ae|%an|%aI|%s' \
    --shortstat \
    --date-order
```

`--all` 是关键:扫**所有分支**,不只 main。开发者经常在 feature 分支上开发,日报必须能看到。

**每个 commit 提取的字段**:

| 字段 | 用途 |
|---|---|
| `sha` (前 7 位) | 唯一标识,日志 / 调试用 |
| `author_name` | 用户归属候选(本地 git config 的 user.name) |
| `author_email` | 用户归属候选(本地 git config 的 user.email) |
| `committed_at` | 时间(ISO),验证落在窗口内 |
| `subject` | 提交描述(给 AI 评分时作为产出内容) |
| `files_changed` / `insertions` / `deletions` | 体量度量(给 AI 看"产出量") |
| `branches` | 该 commit 命中哪些分支(`--all` 决定) |

**用户归属(commit author email/name → users.pinyin)**:

代码仓库的 commit author 由每个研发**本地** `git config` 决定,可能是中文名、pinyin、github_username,邮箱也任意(公司邮箱、私邮、GitHub noreply)。日报通过**8 级精确反查**匹配:

| 优先级 | 匹配规则 |
|---|---|
| 1 | settings 里 `daily_report.commit_author_overrides` 的人工映射 |
| 2 | GitHub noreply 邮箱 `\d+\+(name)@users.noreply.github.com` → `users.github_username` |
| 3 | `commit.author_name == users.pinyin` 精确 |
| 4 | `commit.author_name == users.github_username` 精确(忽略大小写) |
| 5 | `commit.author_name == users.name` 精确(中文名) |
| 6 | email 前缀 (`@` 之前) `== users.pinyin` |
| 7 | email 前缀 `== users.github_username` |
| 8 | 都未命中 → 进 `unattributed_commits` 桶 |

**不做模糊 / substring 匹配**——错把 A 的 commit 算到 B 头上比"未识别"更糟。未识别桶在卡片末尾显示总数(不暴露邮箱),管理员可在系统设置里补 override 映射,下次日报就能正确归属。

### 4.4 按用户聚合

把 4.2 / 4.3 拿到的事件按 pinyin 聚合,形成每人一个 `UserActivity`:

```
UserActivity {
  pinyin
  display_name              ← users.name 优先,fallback pinyin
  file_creates              ← creator == 自己 的 timeline items
  file_owns                 ← owner == 自己 (但 creator 是别人) 的 items (派给我的活)
  verifications_given       ← 自己作为 verify owner 给的 judgements
  status_changes_triggered  ← 自己创建的、带 status_change 的文件
  comments_given            ← 自己写的评论(自己文件 + 别人文件)
  mentions_received         ← 自己被 @ 的次数
  commits                   ← attribution 到自己的 git commits
}
```

`TeamSummary` 是同时段全员活动的汇总(总文件数、总 commits、状态推进次数、涉及 matter 数等)。

### 4.5 评分逻辑

#### 输入给 AI 的精简 JSON

不喂 raw timeline(token 会爆),只喂经过聚合的精简结构:

```json
{
  "window": {
    "since": "2026-04-26T09:30:00+08:00",
    "until": "2026-04-27T09:30:00+08:00"
  },
  "team_stats": {
    "matters_touched": 7,
    "files": 23,
    "commits": 41,
    "status_changes": 3,
    "comments": 18,
    "active_users": 6,
    "inactive_users": 2,
    "unattributed_commits": 3
  },
  "per_user": [
    {
      "pinyin": "dengke",
      "name": "邓柯",
      "files_created": [
        {"matter_id": "auth-redesign", "type": "act", "summary": "推进登录回跳与 cookie 处理"},
        {"matter_id": "auth-redesign", "type": "result", "summary": "auth-redesign 收口,可接受"}
      ],
      "files_owned_by_me": [],
      "verifications_given": [
        {"matter_id": "mobile-ui", "judgement": "passed", "comment": "主链路通过"}
      ],
      "status_changes": ["auth-redesign: executing → finished"],
      "comments_given": 4,
      "mentions_received": 2,
      "commits": [
        {"subject": "feat(auth): 接 cookie 持久化", "stats": "+120/-30 5 files"},
        {"subject": "fix(auth): 修边界回跳", "stats": "+18/-8 2 files"}
      ]
    }
  ]
}
```

#### 给 AI 的完整提示词

System prompt:

```
你是 Pivot 团队日报评分助手。基于下面 24 小时的工作活动数据,
为每个成员和团队整体给出客观评价。

评分规则:
- 5 分制,允许 0.5 步进 (1.0 / 1.5 / 2.0 / ... / 5.0)
- 4 个维度独立打分,然后给出总分 (可不等于平均):
  · output    = 产出量    (写了多少 file / commit,体量是否充足)
  · progress  = 推进结果  (status_change、verify passed、result 文件等"事项闭环"信号)
  · blocker   = 阻塞情况  (5=完全无阻塞;有 paused 或长时间未推进则扣分)
  · collab    = 协作密度  (mention、quote 引用、verify 给他人的判断)

输出约束(严格):
- 没有任何活动的成员不要进 per_user
- 严格输出下面 JSON 格式,不要 markdown 包裹,不要任何解释文字

{
  "team": {
    "score": 4.0,
    "sub_scores": {"output": 4.0, "progress": 3.5, "blocker": 4.5, "collab": 4.0},
    "summary": "一句话总结(<=40 字)"
  },
  "per_user": [
    {
      "pinyin": "dengke",
      "score": 4.5,
      "sub_scores": {"output": 5.0, "progress": 4.0, "blocker": 5.0, "collab": 4.0},
      "summary": "1-2 句简评(<=60 字)",
      "highlights": ["完成 auth-redesign 收口", "verify 了 liuyu 的 act"]
    }
  ]
}
```

User prompt: 上面 4.5.1 那段精简 JSON 直接作为消息内容。

#### AI 期望响应

```json
{
  "team": {
    "score": 4.0,
    "sub_scores": {"output": 4.0, "progress": 3.5, "blocker": 4.5, "collab": 4.0},
    "summary": "全员推进有节奏,auth-redesign 收口稳,collab 略低需关注"
  },
  "per_user": [
    {
      "pinyin": "dengke",
      "score": 4.5,
      "sub_scores": {"output": 4.5, "progress": 5.0, "blocker": 5.0, "collab": 4.0},
      "summary": "完成 auth-redesign 收口,verify 通过组员 2 篇 act",
      "highlights": ["auth-redesign · finished", "verify ×2", "commits ×8"]
    },
    {
      "pinyin": "liuyu",
      "score": 4.0,
      "sub_scores": {"output": 4.5, "progress": 3.5, "blocker": 4.5, "collab": 3.5},
      "summary": "主推 mention 系统 3 篇 act 顺利落地",
      "highlights": ["mention-system · act ×3", "commits ×12", "mentions ×5"]
    }
  ]
}
```

#### 解析与降级

```
正常路径:解析 JSON → 校验 schema(score 在 [1.0, 5.0]、4 个 sub_scores 齐全、per_user 字段完整)
       → 打包成 ScoringResult.status = "ai"

降级触发:任一条件命中 →
       - AI 接口超时 / 错误
       - 输出非 JSON / JSON 缺字段 / score 越界

降级行为:
       - team.score = 3.0 中性
       - per_user.score 按 (file_creates + commits) 数量给粗分:
           0~3 项 → 2.5 / 4~7 项 → 3.5 / 8+ 项 → 4.5
       - sub_scores 全置 null
       - 卡片明显标注 "⚠️ AI 评分缺失,以下仅展示统计"
       - ScoringResult.status = "fallback"
```

降级**不会让脚本失败退出**——日报照发,只是评分块降为统计块。这样保证日报稳定送达,AI 异常仅以"卡片提示"形式暴露。

### 4.6 卡片渲染与发送

- 卡片框架复用 `notify.py::_card_shell`(标题 + markdown + 按钮)
- markdown 块顺序:窗口 → 团队总览统计 → 团队评分 + AI 总结 → 个人评分(降序)→ 0 活动用户 → 未识别 commits 计数
- 发送复用 `notify.py::FeishuNotifier._broadcast`,自动枚举 bot 所在的所有飞书群
- AI 模式 vs 降级模式用不同 header 颜色区分(`blue` vs `wathet`)

## 五、输出结果示例

### 5.1 正常情况(飞书群卡片)

```
┌──────────────────────────────────────────────────────────────┐
│ 📊 团队日报 · 4-26                              [蓝色 header] │
├──────────────────────────────────────────────────────────────┤
│ 📅 覆盖窗口:2026-04-26 09:30 → 2026-04-27 09:30             │
│                                                              │
│ 📈 团队总览                                                  │
│ matter 事件 23 篇 · git commits 41 个                        │
│ 状态推进 3 次 · 评论 18 条                                   │
│ 涉及 matter 7 个 · 活跃成员 6 人                             │
│                                                              │
│ ⭐ 团队评分:★★★★☆ (4.0)                                    │
│ 产出 4.0 · 推进 3.5 · 阻塞 4.5 · 协作 4.0                    │
│ "全员推进有节奏,auth-redesign 收口稳,collab 略低需关注"      │
│                                                              │
│ 👥 个人评分                                                  │
│                                                              │
│ 邓柯 · ★★★★★ (4.5)                                         │
│   完成 auth-redesign 收口,verify 通过组员 2 篇 act           │
│   auth-redesign · finished / verify ×2 / commits ×8          │
│                                                              │
│ 刘昱 · ★★★★☆ (4.0)                                         │
│   主推 mention 系统,3 篇 act 顺利落地                        │
│   mention-system · act ×3 / commits ×12 / mentions ×5        │
│                                                              │
│ 唐昆 · ★★★☆☆ (3.5)                                         │
│   推进 mobile UI 改造,但 paused 1 件待恢复                   │
│   mobile-ui · think ×2 / commits ×9 / paused ×1              │
│                                                              │
│ 熊建平 · ★★★☆☆ (3.0)                                       │
│   主要在写 think,推进结果较少                                │
│   ai-tools · think ×4 / commits ×6                           │
│                                                              │
│ 😴 今日 0 活动:张博、佘耀君                                  │
│ ❓ 未识别 commits:3 个                                       │
│                                                              │
│ [📊 进入 Pivot]                                              │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 边界情况

#### 全员 0 活动

```
┌──────────────────────────────────────────────────────────────┐
│ 📊 团队日报 · 5-1                               [蓝色 header] │
├──────────────────────────────────────────────────────────────┤
│ 📅 覆盖窗口:2026-05-01 09:30 → 2026-05-02 09:30             │
│                                                              │
│ 😴 今日团队无活动(节假日 / 集中放空)                         │
│                                                              │
│ [📊 进入 Pivot]                                              │
└──────────────────────────────────────────────────────────────┘
```

#### AI 调用失败(降级)

```
┌──────────────────────────────────────────────────────────────┐
│ 📊 团队日报 · 4-26                            [浅蓝色,降级]  │
├──────────────────────────────────────────────────────────────┤
│ 📅 覆盖窗口:2026-04-26 09:30 → 2026-04-27 09:30             │
│ ⚠️ AI 评分缺失,以下仅展示统计                                │
│                                                              │
│ 📈 团队总览                                                  │
│ matter 事件 23 篇 · git commits 41 个 ...                    │
│                                                              │
│ 👥 个人活跃度(按事件 + commit 数排序,无 AI 评分)            │
│                                                              │
│ 邓柯 · 11 项 (matter 3 / commits 8)                          │
│ 刘昱 · 15 项 (matter 3 / commits 12)                         │
│ ...                                                          │
└──────────────────────────────────────────────────────────────┘
```

## 六、局限与已知 trade-off

| 局限 | 说明 |
|---|---|
| 初版不写回 matter | 不生成 insight 文件入库,只发飞书卡。后续视采纳度决定要不要把日报本身作为可追溯产物入 Pivot |
| commit ↔ matter 不归并 | matter 文件 vs git commit 在初版分两块独立呈现,**不做交叉去重**。同一份工作既有 act 又有 commit,各算各的——精确归并语义留迭代 |
| 不允许人为豁免成员 | "为什么 X 没在日报里"是更糟的问题。如果有 leave / 离职状态,应在 users 表层面建模,不在日报层 |
| AI 评分仅供参考 | 评分是 AI 主观判断,**不入考核**。仅用于"快速扫描异常"而非"打绩效" |
| 未识别 commits 出现在卡片 | 让管理员看见才能补 override 映射。仅展示**总数**,不暴露个人邮箱细节 |
| 工作日 / 周末不区分 | 初版周末也跑(可能多数日期是 0 活动卡)。视效果决定要不要加工作日过滤 |
| AI 评分维度无法定制 | 4 维度硬编码在 prompt 里。如需调整(增减维度、改权重),要改代码并重新发版 |

## 七、后续迭代方向

1. **日报本身入库**:把每天的日报作为 `insight` 文件写到"运营元 matter",可追溯、可被 AI 后续读取
2. **commit ↔ matter 归并**:精确识别同一工作的代码与 matter 文件,产出"完整闭环度"指标
3. **周报 / 月报**:基于日报数据再聚合
4. **评分人为校正**:管理员可在事后给某天的评分加批注 / 修正,记入历史
5. **个人推送**:每个用户额外收到自己的"个人日报"DM,作为自我反思工具
6. **阻塞预警**:连续 N 天 paused 的 matter 自动单独提示
7. **卡片格式优化**:加趋势图 / 雷达图(飞书卡片 schema 2.0 支持)

## 八、责任与上下文

- 本文档:`AI-docs/daily-report/product-design.md`(产品设计,评审入口)
- 实施计划:`AI-docs/daily-report/implementation-plan.md`(技术方案 + 实施路径)
- Pivot 产品本体:`AI-docs/pivot-product.md`
- 项目实现现状:`AI-docs/pivot-memo.md`
- 部署手册:`AI-docs/pivot-deploy.md`
