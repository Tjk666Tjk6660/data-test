---
type: act
author: lishuai
created: '2026-04-29T12:02:48+08:00'
index_state: indexed
---
## 一、需求与现状

### 1. 需求一句话

统一「发布新 matter 第一篇文档」和「发布回复文档」的内容创建逻辑：

- 默认鼓励 AI 协作；
- 允许跳过 AI 直接输入；
- 未经 AI 协作的内容在发布前弹质量提示，提供「立即发布」与「进行 AI 讨论」两个出口。

### 2. 原文目标 → 方案对应

| 原文目标 | 方案对应 |
|----------|---------|
| 默认支持并鼓励用户先和 AI 助手讨论，再生成草稿 | NewMatter 默认进入 6 步**引导式对话流**（左聊天 + 右预览）；reply 侧 AIPane 入口保留现状 |
| 允许用户跳过 AI 助手，直接输入并发布内容 | 顶部"跳过引导，直接写"按钮一键切到 classic form；质量门**不阻塞**，"立即发布"永远可点 |
| 对未经当前 AI 助手协作生成的内容，在发布前进行明确质量提示 | 新增 `body_source` 标签 + 公用 `PublishQualityConfirmDialog`，发布前按需弹窗 |
| 允许用户在发布前，将当前草稿转交给 AI 助手继续讨论和整理 | 弹窗按钮"进行 AI 讨论" → 把 `{title, category, docType, mentions, body}` 全量 bridge 到引导流并自动起草 |

### 3. 关于"当前 AI 助手讨论后输出"的判定语义

需求原文的检测条件是：「不是由**当前 AI 助手和用户讨论后输出**的」。这里隐含两个语义：

1. **由 AI 输出**：内容来自 AI，而不是用户手打。
2. **当前 + 讨论后**：在草稿对应的 AI 会话里，由用户**主动触发**生成。

方案里这两条对应到现有机制：

- AI 受 system prompt 约束（[server/ai/prompts.py](../../server/ai/prompts.py)）**不会自发输出 `<draft>`**，必须用户点"生成草稿"按钮、注入 `[[GENERATE_REPLY_DRAFT]]` 才会输出 → 满足"讨论后触发"。
- 草稿与 AIPane 会话用 `threadKey` 一对一绑定 → 满足"当前会话"。
- 判定**只看 body 内容来源**（与最近一次 AI 产出的相似度），**不绑定对话是否还存活**——用户清空对话不影响判定，避免误伤。

### 4. 当前状态盘点

| 路径 | AI 入口 | 草稿持久化 | 发布前质量门 |
|------|--------|----------|-------------|
| 回复文档（reply / result / insight / verify） | 有，[AIPane](../../web/src/components/AIPane.tsx) + `<draft>` 协议 | 走 `useDraftAutosave` | **无** |
| 新 matter 第一篇文档 | **无**（[NewMatter.tsx](../../web/src/pages/NewMatter.tsx) 仅有 Textarea，AI 仅用于生成 summary） | 走 `useDraftAutosave` | **无** |
| 评论（comment） | 无 | 无 | **不在本次范围**（见 §四） |

两条不一致需要在本次抹平：**新 matter 缺 AI 入口**、**两条路径都缺质量门**。

---

## 二、核心设计

### 1. 两条路径流程图

NewMatter 与 reply 路径的 AI 协作形态不同（前者是 6 步引导流，后者是 AIPane），但**都共用**同一套 body_source 状态机 + 质量门。流程图按场景而非组件画。

**路径 A：AI 协作发布**

```
NewMatter（默认引导流）            reply（AIPane）
        │                               │
        │                               │
6 步引导对话                       和 AI 自由讨论
topic→type→category→             点【生成草稿】注入
title→mentions→drafting          [[GENERATE_REPLY_DRAFT]]
        │                               │
        ↓                               ↓
AI 流式 <draft> + <summary>      AI 返回 <draft> + <summary>
        │                               │
        ↓                               ↓
落入引导流右侧预览面板             落入草稿卡片
body_source = "ai"               body_source = "ai"
snapshot = <draft> 内容          snapshot = <draft> 内容
        │                               │
        ↓                               ↓
review 阶段可继续与 AI 讨论        在草稿界面继续编辑
（chat composer 让 AI 重出       （小改 < 50% 仍判 ai；
覆盖旧版；保持多轮 history）      大改 → manual）
        │                               │
        └───────────┬───────────────────┘
                    ↓
        点【立即发布】→ body_source == "ai"：直接发，不弹窗
```

**路径 B：直接输入发布**

```
NewMatter classic form                reply（CreateFileDialog）
（"跳过引导，直接写"）
        │                                   │
        │                                   │
直接在表单手打正文                     直接在 textarea 手打
body_source = "manual"               body_source = "manual"
        │                                   │
        ↓                                   ↓
点【创建 Matter】 ←──── 共用质量门 ────→ 点【发布】
                       触发条件相同
        │                                   │
   ┌────┴────┬─────────────────┬───────────┴────┐
   ↓         ↓                 ↓                ↓
立即发布   进行 AI 讨论         取消             加入 AI 队列*
   │         │                 │                │
   ↓         ↓                 ↓                ↓
原 publish  NewMatter:        关弹窗          AI 流被其他
落盘 manual 全量 snapshot     （回到编辑器）   thread 占用时
            bridge → 引导流                    的 fallback
            自动起草（路径 A）                 文案，仅 reply
            
            reply:
            ai.setInput(body)
            打开/聚焦 AIPane
            不自动 send
```

`*` "加入 AI 队列" 仅在 reply 路径出现——NewMatter 没有 AI 流并发竞争，没这层。

### 2. `body_source` ：内容来源标签

新增字段，写入 draft `matter_payload`、并随发布落到 post frontmatter：

```ts
body_source: "ai" | "manual"
body_source_snapshot?: string  // 上次 AI 产出的 body 原文（仅草稿层用，不进 frontmatter）
```

#### 状态机：**单向降级**，永不升级

需求原文的关键语义是"由 AI 助手和用户**讨论后输出**"——必须经过 AI 协议显式输出，而不是"碰巧和某次 AI 草稿很像"。所以 `body_source` 是单向状态机：

```
                 AI 通过 <draft> 协议写入
   manual  ──────────────────────────────►  ai
              （唯一升级通道：显式 AI 输出）
              
   ai      ──────────────────────────────►  manual
              （唯一降级通道：用户编辑且与 snapshot 偏离）

   manual  ──────────────────────────────►  manual
              （用户怎么改、粘贴什么，都还是 manual）
```

具体规则：

- **唯一升级路径**：[handleUseDraftAsReply](../../web/src/pages/MatterDetailPane.tsx#L585) 接收到 AI 的 `<draft>` 时，set `body_source = "ai"` + `snapshot = <draft>` 内容。
- **降级判定**（仅当当前是 `"ai"` 时执行）：用户编辑后用 LCS 与 snapshot 比，`sim < 0.5` 翻为 `"manual"`，`sim ≥ 0.5` 保持 `"ai"`。
- **`"manual"` 状态下**：用户做任何编辑（包括粘贴一大段恰好与某次 AI 草稿高度相似的外部文本）**都不会升级回 `"ai"`**。这是防"粘贴绕过质量门"的关键。
- **空 body**：发布按钮 disabled，不进入判定。

#### 相似度算法（仅用于 ai → manual 降级）

```
sim = LCS(current_body, snapshot) / max(len(current_body), len(snapshot))
```

- `sim ≥ 0.5` → 保持 `"ai"`
- `sim < 0.5` → 降级 `"manual"`
- `len(current_body) < len(snapshot) * 0.3` → 直接降级 `"manual"`（防"AI 写一大段、用户删剩两行新写"）

#### 性能与存储

- **LCS 只在用户点【发布】按钮时算一次**，不在 onChange 里实时算——避免长文本（>5000 字）的卡顿。
- **snapshot 存全文**，但天然受 Textarea `maxLength={50000}` 约束，单条草稿 payload 增量可控；不做截断（截断会丢精度）。
- autosave 已有 debounce（参考 [useDraftAutosave](../../web/src/hooks/useDraftAutosave.ts)），snapshot 字段共享同一节流，不会额外抖动写入。

### 3. 发布前质量门

新增公用组件 `PublishQualityConfirmDialog`，由一个 hook 包装：

```ts
const confirmPublishQuality = (body: string, source: "ai" | "manual") =>
  Promise<"go" | "send_to_ai" | "cancel">;
```

触发条件：用户点发布且 `source === "manual"`。

弹窗文案直接使用需求文档原文：

> 检测到这篇内容不是由当前 AI 助手和你讨论后输出的。
>
> 请确认这篇文档能够代表你的真实想法，并且足够清晰可读、有思考沉淀、有结构化表达，以便持续提高团队讨论质量。
>
> 对于低质量输入，后期我们会使用 AI 工具进行检测和评价。

按钮（同一组组件按路径有不同行为）：

- **立即发布** → 关闭弹窗，继续原 publish 链路（包括 `body_source: "manual"` 落盘）。
- **进行 AI 讨论** → 行为按入口分支：
  - reply 路径：关闭弹窗 + 打开/聚焦 AIPane + `ai.setInput(threadKey, currentBody)`，**不自动 send**，让用户追加说明再发；
  - NewMatter classic form 路径：关闭弹窗 + 把 `{title, category, docType, mentions, body}` 全量 snapshot bridge 到 guided flow，guided flow 自动起草（详见 §二.4.B）。
- **取消** → 关闭弹窗回到编辑器，不变更状态。

#### 边界处理

| 场景 | 行为 |
|------|------|
| reply：AI 流被其他 thread 占用（`blockedByOtherThread`） | 弹窗里"进行 AI 讨论"按钮文案动态切换为**"加入 AI 队列"**，副本提示"内容已填入输入框，等当前对话「xxx」完成后再发送"。点击后仍执行 setInput + open，但**不显示 streaming hint**——明确告诉用户当前是排队态 |
| 用户清空 AIPane 对话后又微改发布 | 仍按 body 与 snapshot 比对判定，不绑定对话存活 |
| reply：AI 没聊一句直接点"生成草稿" | 算 AI 协作（用户主动触发，AI 看了上下文） |
| 用户拒绝弹窗（取消） | 关弹窗回到编辑器，**不阻塞**；用户随时可再点发布；体现原文"不会因为 AI 流程而失去直接发布的自由" |
| 紧急内容需要直接发 | 路径 B → 立即发布，一次确认即可，不需要先走 AI |

### 4. NewMatter 的 AI 协作流

NewMatter 拆成**两形态**，由 [NewMatter.tsx](../../web/src/pages/NewMatter.tsx) 顶层做 mode 切换：

- `NewMatterGuidedFlow`（默认）—— AICraft 风格的 6 步引导对话，左侧聊天气泡 + 右侧实时构建的预览面板
- `NewMatterClassicForm`（备选）—— 传统表单，从顶部"跳过引导，直接写"按钮进入；表单内**不内嵌** AIPane

体现需求文档"默认鼓励 AI 协作"原则——默认入口是 AI 引导，但允许 1 键退出到纯手写。

#### A. guided flow 的 6 步流程

| 步骤 | 用户输入 | UI 形态 | AI 角色 |
|------|---------|--------|---------|
| 1. topic | 一句话描述要讨论什么 | Textarea，Ctrl+Enter 提交 | 不参与 |
| 2. type | 选 think 或 act | 单选卡片 | 不参与 |
| 3. category | 选既有种类或新建 | select + 新建输入 | 不参与 |
| 4. title | 主题句 | 基于 topic 自动建议，可改 | 不参与 |
| 5. mentions | 圈人（可跳过） | MentionField | 不参与 |
| 6. AI 起草 | (无输入) | 流式预览面板 | **唯一发声**——基于上述 5 步信息流式输出 `<draft type="...">` + `<summary>` |
| 7. review | 微调 / 继续讨论 | 右侧编辑 + 左侧 chat composer | 多轮修订（详见 §D） |

每步的用户输入都作为 user bubble 写入聊天历史；步骤切换由 AI bubble 自动衔接（"好的。这篇你倾向于 think 还是 act？"等）。

#### B. 形态切换与 bridge

```
NewMatterGuidedFlow ⇄ NewMatterClassicForm
        │                       │
   "跳过引导，直接写" ──────────→  classic form
        │                       │
        ←──── 切回 AI 引导 ──────  （手动按钮）
        ←──── 进行 AI 讨论 ──────  质量门 send_to_ai 分支
                                 + 全量 snapshot bridge
```

**bridge 的语义**：classic form 写到一半被质量门拦下来，用户选"进行 AI 讨论"——前端把 `{title, category, docType, mentions, body}` 整包传给 guided flow，guided flow 检测到 bridge 后跳过 5 步直接进 drafting；`buildDraftPrompt` 多注入一段"用户原始草稿"，要求 AI 保留事实、只重构表达。这样用户写到一半的内容**不会**因为切引导流而丢失。

#### C. AI 工具集

NewMatter 模式 AI **与 reply 路径完全一致**，都拿到 5 个工具：

| 工具 | 用途 |
|------|------|
| `list_thread_titles` | 列全部 matter / thread 标题 |
| `search_indexes` | 跨 matter 搜索 index |
| `read_thread_index` | 读某 thread 的 index |
| `read_matter_index` | 读某 matter 的 index |
| `read_post` | 读某篇 post 正文 |

理由：

- **产品一致性**：两条路径若工具集不一致，会形成"reply AI 比 NewMatter AI 强"的体感分裂。
- **真实价值**：用户新建 matter 时可能说"延续 #某 matter 讨论的方向"或"看看团队最近有没有在聊类似话题"——这些诉求自然落到 `search_indexes` / `read_matter_index` / `read_post` 上。
- **实现简化**：两条路径共用同一份 `AITools(...).specs()`，prompt 按 mode 分支即可。

**保留 mode 区分的部分**（在 prompt，不在工具）：
- 告诉模型"当前没有 matter 上下文，用户在新建 matter 阶段"；
- 收到 `[[GENERATE_REPLY_DRAFT]]` 时除了 `<draft>` / `<summary>`，多输出一个 `<title>` 标签（仅供参考，前端目前不消费——见 §F）。

#### D. review 阶段的多轮 AI 修订

drafting 完成后进入 review 阶段。右侧预览面板是初版草稿；底部出现一个 chat composer，用户可继续告诉 AI "再精简些"/"加一段 X 背景"等反馈。

实现要点：
- 维护 `aiHistory: ChatMessage[]`，把首轮起草的 user prompt + assistant response 完整存住；后续每次反馈追加一对 user + assistant 入 history
- 每轮 AI 重新输出 `<draft>` + `<summary>`，**整体覆盖** `data.body` / `data.summary`——不做 patch / merge
- 修订期间预览面板显示 streaming，chat composer 禁用直到完成
- 保留 multi-turn 让用户能渐进打磨，但任何一版都是完整的 `<draft>`，避免"半成品被发布"

按钮：
- "立即发布" —— 当前预览即最终版，body_source = "ai"，**不弹质量门**（来自 AI 的 `<draft>` 协议输出）
- "返回上一步"（仅非 bridge 入口） —— 回 mentions 步骤，重新经历 drafting

#### E. 预览面板交互（拖动 + 全屏）

md+ 屏：

- 中间有竖向 drag handle，左右宽度可拖（预览面板范围 320–880px）；用 CSS 变量 `--preview-w` 串到 Tailwind arbitrary value 实现，narrow 屏不渲染 handle
- 预览面板右上角 Maximize2/Minimize2 toggle：全屏 = `fixed inset-0 z-50` overlay，最小化回侧栏

#### F. 各字段由谁决定

| 字段 | 由谁决定 | 备注 |
|------|---------|------|
| `topic` | guided 第 1 步用户输入 | bridge 时取 classic form 的 body |
| `doc_type` | guided 第 2 步用户选 / classic 表单单选 | AI 不动 |
| `category` | guided 第 3 步用户选 / classic 表单 | AI 不动 |
| `title` | guided 第 4 步用户输入（基于 topic 自动建议）/ classic 表单输入 | AI 在 prompt 输出的 `<title>` 标记为"仅供参考"，前端不消费——避免抢用户的命名 |
| `mentions` | guided 第 5 步 / classic 表单 | AI 不动 |
| `body` | guided 第 6 步 AI 起草 + 第 7 步可继续修订 / classic 表单手打 | 触发质量门的核心字段 |
| `summary` | 路径 A：AI 在 `<draft>` 之外同时输出 `<summary>`；classic + 立即发布：发布时再调一次 AI 基于 body 生成 | classic form 没有 AI 渠道直接给 summary，只能后补 |
| `status` | 系统默认（`planning`） | AI 不参与；状态推进走 status_change 流 |

#### G. 草稿持久化策略

| 形态 | 草稿持久化 |
|------|----------|
| guided flow | **不持久化**——是会话式的一次性引导。中途关页会丢；考虑过用本地 storage 但首版未做（用户体验上 bridge 出去到 classic form 再回来时也不需要恢复 step） |
| classic form | 沿用既有 [useDraftAutosave](../../web/src/hooks/useDraftAutosave.ts)，含 body_source / body_source_snapshot |
| reply（AIPane） | 既有不动 |

#### H. 窄屏响应式

guided flow：聊天 + 预览原本是左右两栏，narrow 改成上下堆叠（聊天在上，预览在下），跟着 Dashboard 的 `<main>` 自然滚动。无需"AIPane 引导"——guided flow 本身就是 AI 协作的可视化表达，第一屏就能看到 AI bubble。

classic form：单列布局，与既有 NewMatter 表单的 narrow 形态一致。

reply 路径（[MatterDetailPane.tsx](../../web/src/pages/MatterDetailPane.tsx)）的 AIPane 现有窄屏处理（叠加在 timeline 上的浮层）保留，不动。

质量门 `PublishQualityConfirmDialog` 用现成的 [Dialog](../../web/src/components/ui/dialog.tsx) 组件，自带窄屏全屏适配，无额外工作。

---

### 5. AI 失败兜底（跨路径）

AI 流可能在多个环节失败，统一处理原则：**body 不变、不触发质量门、Toast 提示并保留重试入口**。

| 失败场景 | 行为 |
|---------|------|
| AI 流式连接错误（网络断、5xx、API key 失效） | 已写入的部分文本舍弃；body 不变；Toast「AI 暂时不可用：xxx，请稍后重试」；AIPane composer 立即解锁可重发 |
| AI 输出超时（>120s 无 token） | 同上，Toast 改为「AI 响应超时，请重试」 |
| AI 返回空 `<draft>` 或缺标签（路径 A） | 不写入 body；Toast「AI 没有给出可用草稿，请补充更多上下文后再试」；保留 AIPane 现有对话 |
| AI 在 NewMatter 模式下未给 `<title>` | 仅缺 `<title>` 不视作失败：标题保持用户已输入或留空，body 和 summary 正常落 |
| 用户中途停止（如关 AIPane / 切 thread） | 复用现有 abort 逻辑；body 不变；不触发质量门；不需要 Toast |

关键约束：**任何 AI 失败都不应该让 `body_source` 翻 `"ai"`**——只有完整收到 `<draft>` 且解析成功，才执行升级。

### 6. 发布后状态流转（跨路径）

**Reply 路径**：
- publish 成功 → 现有 `afterWrite()` 流程不变（refresh matter detail + reload sidebar）；
- 该草稿对应的 `pendingCreate` / `pendingDraftId` / `pendingInitial` / `body_source` / `snapshot` 全部 reset；
- **AIPane 对话保留**——按 thread 维度延续，用户接着同一 matter 写下一条回复时上下文还在（这是现有体感，不动）。

**NewMatter 路径**：
- publish 成功 → 沿用现有 `navigate(\`/m/${matter_id}\`)` 跳到详情页；
- guided flow 是会话式 UI，发布即会话生命周期结束，不留持久状态；
- classic form 的 draft 在 publish 成功后被 `deleteDraft(draftId)` 清掉。

**遗留的 `__newmatter__:` 虚拟 threadKey 孤儿清理**：早期 AIPane-on-NewMatter 设计阶段曾在 AI store 里堆 `__newmatter__:${draftId}` 前缀的会话——guided flow 不再产生这种 key，但旧分支在用户本地 storage 里留下的孤儿仍需清。Dashboard 首次 mount 跑一次清理（详见 [Dashboard.tsx](../../web/src/pages/Dashboard.tsx) 的 effect），扫所有 `__newmatter__:` 前缀的 thread，对应的 draftId 已不在 `fetchDrafts()` 结果里就删掉。

**质量门点击"取消"或拒绝**：
- 不做任何状态变更；保留 classic form 的草稿、保留 AIPane 对话；用户可继续编辑或再次点发布。

---

### 7. 对需求文档"交给实现同学思考的问题"的回答

需求文档结尾留了 3 个待决问题，逐一回答：

**Q1. 新 matter 的 AI 助手是否复用回复文档的助手逻辑，还是需要单独针对 matter 创建设计提示词和字段结构？**

A：**框架与工具集都复用，仅 prompt 单独**。

- 复用：AIPane 组件、`useDashboard().ai` store、`<draft>` / `<summary>` 协议、草稿写入回调、`AITools(...).specs()` 全部 5 个工具（详见 §二.4.C）。
- 单独：[server/ai/prompts.py](../../server/ai/prompts.py) 新增 `new-matter` 模式 system prompt（无 matter 上下文 / 增加 `<title>` 协议）。
- 字段结构：复用现有 draft `matter_payload`，仅追加 `body_source` / `body_source_snapshot` 两个公用字段。

**Q2. AI 生成新 matter 草稿时，是否需要同时整理 matter 的结构化字段（标题、分类、摘要、状态等）？**

A：**只整理标题和摘要**，分类/类型/状态不动。详见 §二.3.F。

- 标题：AI 给建议（`<title>`），用户没手输才预填，否则只提示。
- 摘要：路径 A 复用 AI 的 `<summary>`，路径 B 沿用旧的"发布时再调一次"逻辑。
- 分类（category）：用户主导组织习惯，AI 没足够信号；不让 AI 选。
- 类型（doc_type）：用户决策点，AI 默认按 think 输出。
- 状态（status）：系统默认 `planning`，AI 不干涉；和现有 status_change 解耦。

**Q3. 草稿保存机制是否需要和普通回复草稿保持一致，避免用户在讨论和编辑过程中丢失内容？**

A：**完全一致**，沿用 [useDraftAutosave](../../web/src/hooks/useDraftAutosave.ts)。

- NewMatter 已经在用同一套 hook（[NewMatter.tsx:101-119](../../web/src/pages/NewMatter.tsx#L101)），不引入第二套机制。
- AIPane 会话用虚拟 `threadKey = __newmatter__:${draftId}` 与草稿绑定，刷新/换设备恢复时对话和 body 一起回。
- `body_source` / `body_source_snapshot` 写进 `matter_payload`，跟着草稿活，跨刷新仍能正确判定质量门。

---

## 三、后端改动

### 1. 发布 API：post frontmatter 加 `body_source`

涉及 6 个发布入口（前端调用，后端落盘）：

| 入口 | 后端 |
|------|------|
| 新 matter 第一篇 | `POST /api/matters` |
| reply（think / act / verify） | `POST /api/matters/{id}/files` |

**result / insight 不写**：v1 跳过这两类，落盘时不带 `body_source` 字段，前端也不在 publish 前判定（详见 §四）。

后端在 [server/posts.py](../../server/posts.py)（或 publish 落盘处）把 `body_source` 当作可选 frontmatter 字段写入 markdown 头：

```yaml
---
type: think
author: dengke
created: '2026-04-28T...'
body_source: ai          # 或 manual；缺省按 manual 处理（兼容存量帖子）
---
```

**v1 不做 enforcement**：后端不会因为 `body_source: manual` 拒绝发布。这个字段只是为后续"AI 检测低质量输入"留下历史数据。

### 2. AI prompts：新增 NewMatter 模式

[server/ai/prompts.py](../../server/ai/prompts.py) 加一份 system prompt 变体（**仅 prompt 走分支，工具集不变**——详见 §二.4.C）：

- 告诉模型当前没有 matter 上下文，用户在新建 matter 阶段；
- 工具集与 reply 路径完全一致（5 个全开），AI 可按需调用 `search_indexes` / `read_matter_index` / `read_post` 等参考已有 matter；
- 收到 `[[GENERATE_REPLY_DRAFT]]` 时输出 `<draft type="think">` 正文 + `<summary>` + `<title>`，三段都必须有。

[server/ai/client.py](../../server/ai/client.py) 的 streamAIChat 新增 `mode` 参数路由 prompt（沿用现有 `_new_matter_` 占位 matter_id 也可以，二选一）。

### 3. 不改的部分

- 草稿 API（`/api/drafts`）不动，`matter_payload` 是自由 JSON。
- 现有 reply 路径的 `<draft>` / `<summary>` 协议不动。
- MCP 工具不动（外部 AI 走 MCP 通道，本次不涉及）。

---

## 四、范围确认

### 1. 范围内：质量门 + `body_source` 落 frontmatter

| 类型 | 入口 | 备注 |
|------|------|------|
| **新 matter 第一篇文档** | NewMatter | 需求原文明示 |
| **think 回复** | CreateFileDialog（卡片底部「+think」） | 需求原文"回复文档"明示 |
| **act 回复** | CreateFileDialog（卡片底部「+act」） | 同上 |
| **verify 回复** | CreateFileDialog（act 卡片底部「+verify」） | 同属"回复文档"——是"回复 + 验收 act"，仍是文档型回复 |

### 2. 不在本次范围（v1 不接入质量门）

| 项 | 原因 |
|----|------|
| **result（生成 Result）** | 阶段性收尾文档（推进 executing → finished/cancelled），需求原文"回复文档"按字面不含；触发场景固定（matter 完成时），AI 协作价值低于普通回复；v1 跳过，仍可走原 publish 路径，不写 `body_source` 字段。**仍允许**用户用 AI 起草，但发布时不弹门。v1.x 视用户反馈再决定接入 |
| **insight（生成 Insight）** | 同上——复盘归档文档（推进 finished/cancelled → reviewed），不在原文"回复文档"字面范围 |
| 评论（comment）的质量门 | 评论体量小、节奏快，加门破坏讨论流；需求文档措辞是"发布**文档**"，评论不在文义内 |
| 后端按 `body_source` enforce 拒绝低质量发布 | 需求文档明确"后期我们会使用 AI 工具进行检测和评价"，v1 只埋数据 |
| 强制 / 强引导 NewMatter 阶段查重 | 工具是开的（与 reply 一致），AI 可自行判断要不要查；产品不刻意推动，也不专门做"已存在相似 matter"的弹窗提示，留给 v1.1 |
| AI 帮忙决定 category / doc_type | 留给用户选择，暂时为不做 |
| 独立移动端 app / H5 | 项目本身就只有一份 Web 代码，没有原生 app 也没有独立 H5；窄屏由响应式适配（见 §二.4.G） |
| 已存在 post 的 frontmatter 回填 | 存量帖子缺字段时按 `manual` 处理，不批量回填 |

---

## 五、改动文件清单

| 文件 | 变动 | 说明 |
|------|------|------|
| `web/src/components/PublishQualityConfirmDialog.tsx` | **新增** | 公用质量门弹窗（按钮文案"立即发布 / 进行 AI 讨论 / 取消"，blocked 态退化为"加入 AI 队列"） |
| `web/src/hooks/useConfirmPublishQuality.ts` | **新增** | hook 包装弹窗 + 三态 promise |
| `web/src/lib/bodySource.ts` | **新增** | LCS 相似度算法 + 单向状态机 API：`applyAIDraft()`（唯一升级入口）、`onUserEdit()`（按需降级）、`computeAtPublish()`（发布前最终判定） |
| `web/src/pages/NewMatterGuidedFlow.tsx` | **新增** | 6 步引导对话；左聊天 + 右预览；review 阶段 chat composer 多轮 AI 修订；预览面板拖动 / 全屏；接收 classic form bridge |
| [web/src/pages/NewMatter.tsx](../../web/src/pages/NewMatter.tsx) | 大改 | 顶层 mode 切换器：默认 `NewMatterGuidedFlow`，`NewMatterClassicForm` 备选；classic form 接入质量门 + body_source；send_to_ai 走全量 snapshot bridge |
| [web/src/components/AIPane.tsx](../../web/src/components/AIPane.tsx) | 中改 | 新增 `mode` prop（reply / new-matter）；后端协议保留两套 prompt；NewMatter 路径不再消费 AIPane（仅服务 reply） |
| [web/src/pages/MatterDetailPane.tsx](../../web/src/pages/MatterDetailPane.tsx) | 小改 | reply 路径 think/act/verify 接入质量门；result / insight 跳过；`handleUseDraftAsReply` 写入 `body_source = "ai"` 与 snapshot |
| [web/src/components/matter/CreateFileDialog.tsx](../../web/src/components/matter/CreateFileDialog.tsx) | 小改 | Textarea onChange 时调用 `onUserEdit()` 更新 `body_source`；publish handler 按 `context.type` 分支——只对 think/act/verify 触发质量门，result/insight 跳过 |
| [web/src/pages/Dashboard.tsx](../../web/src/pages/Dashboard.tsx) | 小改 | AI store 兼容旧分支留下的 `__newmatter__:` 前缀虚拟 threadKey；启动时做一次孤儿清理（详见 §二.6） |
| [web/src/api.ts](../../web/src/api.ts) | 小改 | Draft / NewFileIn 类型加可选 `body_source`；`streamAIChat` 加 `mode` 参数 |
| [server/ai/prompts.py](../../server/ai/prompts.py) | 小改 | 新增 NewMatter 模式 system prompt（含 `<title>` 协议） |
| [server/api/ai.py](../../server/api/ai.py) | 小改 | chat_matter 接收 `mode` 参数（`reply` / `new-matter`），按 mode 选 system prompt；工具集仍用 `AITools(...).specs()` 不变 |
| [server/api/matters.py](../../server/api/matters.py) | 小改 | `InitialFileIn` / `NewFileBody` 加可选 `body_source` |
| [server/publish.py](../../server/publish.py) | 小改 | publish 时把 `body_source` 写入 post frontmatter |

---

## 六、测试计划

### 1. 单元测试

- `bodySource.ts`：
  - LCS 相似度边界（0.5 阈值附近、空字符串、中文字符、长文本截断）；
  - **状态机单向约束**：从 `"manual"` 状态出发，无论怎么 setBody（包括粘贴一段与 snapshot 100% 相同的文本），结果仍是 `"manual"`；只有 `applyAIDraft()` 才能升级到 `"ai"`。
- 后端 frontmatter writer：`body_source` 缺省时不写、有值时写。

### 2. 集成 / e2e（手测清单）

| 场景 | 期望 |
|------|------|
| NewMatter：默认进页，进入 guided flow 第 1 步（话题输入） | √ |
| NewMatter guided：6 步走完 → AI 流式起草 → 进入 review，预览面板有 body + summary | √ |
| NewMatter guided：review 阶段 chat composer 给反馈 → AI 重新输出，覆盖旧版 body / summary；多轮 history 保留 | √ |
| NewMatter guided：review 阶段点"立即发布"→ 不弹门，post frontmatter `body_source: ai` | √ |
| NewMatter guided：review 期间在右侧预览微改 body（小改 < 0.5） → 仍是 ai；大改 → manual，发布时弹门 | √ |
| NewMatter guided：预览面板 md+ 拖动 handle 调整宽度（320–880px）；narrow 屏不渲染 handle | √ |
| NewMatter guided：预览面板 Maximize 全屏 / Minimize 回侧栏 | √ |
| NewMatter guided：错误回滚——AI 起草失败时回到 mentions 步骤，不丢前 4 步状态 | √ |
| NewMatter classic：顶部点"跳过引导，直接写"切到表单 | √ |
| NewMatter classic：纯手打正文 → 点创建 → 弹质量门 | √ |
| NewMatter classic：质量门点"立即发布" → 创建成功，post frontmatter `body_source: manual` | √ |
| NewMatter classic：质量门点"进行 AI 讨论" → 切到 guided flow，title/category/docType/mentions/body 全部带过去；AI 自动起草并把"用户原始草稿"作为参考 | √ |
| Reply：AI 生成草稿不改 → 发布 → 不弹门，frontmatter `body_source: ai` | √ |
| Reply：AI 生成后小改（相似度 ≥ 0.5）→ 发布 → 不弹门，frontmatter `body_source: ai` | √ |
| Reply：AI 生成后大改（相似度 < 0.5）→ 发布 → 弹门 | √ |
| Reply：纯手打 → 发布 → 弹门 | √ |
| Reply：刷新页面后草稿恢复，`body_source` 状态正确 | √ |
| Comment：发评论 → **不弹门**（防回归） | √ |
| **result / insight 发布 → 不弹门**（防回归）；落盘 frontmatter **不含** `body_source` 字段 | √ |
| Reply：AI 流被其他 thread 占用时点"进行 AI 讨论"→ 按钮文案变"加入 AI 队列" + setInput + open（不显 streaming hint） | √ |
| **粘贴绕过测试**：手动状态下把任意 AI 风格长文粘进 body → 仍为 manual → 仍弹门 | √ |
| AI 流式失败（断网模拟）→ body 不变、不弹门、Toast 重试入口可用 | √ |
| AI 返回空 `<draft>` → body 不变、不弹门、Toast 提示 | √ |
| NewMatter 发布成功后跳转详情页 | √ |
| 旧分支留下的 `__newmatter__:` 孤儿会话：Dashboard 首次 mount 自动清理 | √ |
| Reply 发布成功后 AIPane 对话保留（同 thread 下次回复仍有上下文） | √ |
| 质量门点取消 → 不变更状态 / 草稿保留 / 可再次点发布 | √ |

---

## 七、已确认决策清单

无待批准事项，开发可启动。

| 决策点 | 取值 | 落点 |
|--------|------|------|
| 相似度阈值 | **0.5**（仅用于 ai → manual 降级） | §二.2 |
| `body_source` 状态机 | **单向**：仅 AI `<draft>` 可升级；用户编辑只能降级 | §二.2 |
| 相似度计算时机 | **仅在点【发布】时算一次**，非 onChange 实时 | §二.2 |
| comment 是否纳入质量门 | **否**（豁免） | §四.2 |
| **result / insight 是否纳入质量门** | **否**（v1 跳过；落盘也不写 `body_source`；v1.x 视反馈再决定） | §四.2 |
| `body_source` 是否落 post frontmatter | **是**（仅 think/act/verify + 新 matter 第一篇） | §三.1 |
| NewMatter AI 工具集 | **与 reply 完全一致**（5 个全开） | §二.4.C |
| AI 是否决定 category / doc_type | **否**（用户手选） | §二.4.F |
| **`body_source: "ai"` 状态下编辑是否做视觉提示** | **否**（v1 不在编辑器加状态条；状态变化不抢用户视觉焦点，体感由"发布时不弹门"自然反馈） | — |
| 后端是否按 `body_source` enforce | **否**（v1 只埋数据） | §四.2 |
| **NewMatter 默认形态** | **6 步引导对话流（guided flow）**；用户可"跳过引导，直接写"切到 classic form | §二.4 |
| **classic → guided 的 send_to_ai 行为** | **全量 snapshot bridge**：title/category/docType/mentions/body 整包传过去，guided flow 跳过前 5 步直奔 drafting；`buildDraftPrompt` 注入"用户原始草稿"段要求 AI 保留事实 | §二.4.B |
| **review 阶段是否支持继续与 AI 讨论** | **支持**（多轮修订）；维护 `aiHistory: ChatMessage[]`，每轮 AI 重出 `<draft>` + `<summary>` 整体覆盖 | §二.4.D |
| **预览面板交互** | md+ 拖动调宽（320–880px）+ 全屏 toggle | §二.4.E |
| **质量门按钮文案** | "立即发布 / 进行 AI 讨论 / 取消"；blocked 态退化为"加入 AI 队列"（仅 reply） | §二.3 |
