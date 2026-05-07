---
type: think
author: liuyu
created: '2026-05-06T17:12:27+08:00'
index_state: indexed
---
# 后端并发架构改造与 AI 交互降级 · 设计方案

> **关联**：matter「Pivot 改版升级意见收集」/ act `006_dengke_act_3b91fd.md`
> **状态**：设计草稿，待评审
> **来源反馈**：[004_terry.tao_think](discussions/Pivot/Pivot改版升级意见收集/004_terry.tao_think_76ea16.md)（AI 摘要阻塞 1–2 分钟无提示）；[006_dengke_act](discussions/Pivot/Pivot改版升级意见收集/006_dengke_act_3b91fd.md)（升级为 act，要求统一排查重构后台并发架构并补齐交互提示）

---

## 摘要

Terry 反馈"提交需求文档时 AI 摘要阻塞约 1–2 分钟、无任何提示"是后台并发架构问题在某个高峰时刻的显形，并非孤立 bug。根因有二：**进程级 `write_lock` 把本地写盘和 git 网络往返绑死成一坨**、**AI 流式接口缺乏端到端超时/心跳/取消语义**。本方案分三层修——**A: Write Pipeline**（剥离 git 网络出请求路径）、**B: AI Runner**（统一 AI 调用契约 + SSE 心跳）、**C: JobRunner**（异步任务承载基础设施），并在前端配套 AI 交互五态状态机。同时记录 git 可替换性与 SaaS 多租户扩展性的接口位，**仅留口子，不做提前实施**。

---

## 1. 背景与问题定位

### 1.1 用户反馈

- **Terry @ 11:50**：在提交发布需求文档时，AI 生成摘要功能阻塞约 1–2 分钟，停留在当前操作页面无任何提示，但生成了对应的草稿；再次操作后提交成功；后续尝试复现未果。
- **Dengke 多次反馈**：Web 端 AI 助手进行回复内容分析时，偶发长时间超时和报错，重试后又正常。

### 1.2 根因定位

#### 根因一：SSE 读超时硬编码 120s，前端无任何感知

`server/ai/client.py:60`：

```python
async with httpx.AsyncClient(timeout=httpx.Timeout(10.0, read=120.0)) as client:
```

`read=120.0` 几乎就是 Terry "一两分钟" 的上限值。OpenRouter / Claude 偶发慢回包时，连接会一直占着；前端 `web/src/components/AIPane.tsx` 只有装饰性的"正在对齐颗粒度…"轮播，`Dashboard.tsx` 的 `AbortController` **只用来响应"用户主动点停止"**——没有 inactivity timer、没有"AI 响应较慢，是否重试"的兜底分支。流式静默 → 用户只能看着 ▌ 闪。

#### 根因二：全进程一把写锁，且锁内做 git 网络往返

`server/workspace.py:41` + `server/workspace.py:103-130`：

```python
self.write_lock = threading.Lock()       # 进程级单锁

@contextmanager
def write_session(...):
    with self.write_lock:                  # 全局序列化
        pull(str(self.path))               # ① git pull --rebase  (网络)
        yield                              # ② 落盘 (本地)
        changed = commit(...)              # ③ git commit (本地)
        if changed:
            push(str(self.path))           # ④ git push  (网络, 重试 ×3)
```

凡是写盘必走这把锁：`publish.py` 里的 `publish_proposal` / `publish_reply` / `publish_matter_create` / `publish_matter_append` / `publish_matter_comment` / `publish_matter_owner_change`，以及 `matters.py` 改 visibility，全部排队。每次锁的持有时间 ≈ `pull RTT + commit + push RTT × 重试`，其中 `git_ops.py` 的 `push` 在 rejected 时会进 `time.sleep(1 + attempt)` × 3 的串行退避——一旦遇到非 fast-forward，单次写至少 3–5 秒被独占。

更糟的是 `subprocess.run` 是阻塞调用，FastAPI 的 sync route 跑在 anyio threadpool（默认 40 槽）；threadpool 满了就让新请求**整个排队**，包括读请求。Terry 提交的时间正是午饭点（11:50），是典型午高峰窗口。

#### 双重叠加

NewMatter 里的"生成草稿"是 `AIPane.tsx` 走 `api/ai.py` `chat_matter`。Terry 实际上踩中了"AI 流读超时 + 提交时拿不到 write_lock" 两层阻塞中的某一层（或两层叠加），**前端在两层都没有显式的"超时/重试"语义**——这就是 Dengke 总结"既是体验问题，也可能是架构瓶颈"的来源。

---

## 2. 并发场景全量扫描

按"并发模式 × 阻塞资源 × 现有控制"对全仓扫描：

| # | 模块 | 入口 | 并发模型 | 阻塞资源 | 现有控制 | 风险 |
|---|---|---|---|---|---|---|
| 1 | Workspace 写盘 | `workspace.write_session` | 进程内一把 `threading.Lock` | git pull/push 网络 + 本地 fs | 无超时；push 重试 ×3 串行退避 | 🔴 高 |
| 2 | Matter 发布族 | `publish_matter_*` | 委托 #1 | 同上 | 无 | 🔴 高 |
| 3 | AI 聊天 SSE | `POST /api/ai/matters/{id}/chat` | FastAPI async + httpx async stream | OpenRouter 网络 | `Timeout(connect=10, read=120)` 写死；无心跳 | 🔴 高 |
| 4 | AI 工具循环 | `chat_matter` 内 `_MAX_TOOL_TURNS=8` | 串行循环，每轮内 SSE | 同 #3，叠乘 8 倍 | 仅最大轮数；单轮无超时 | 🟠 中 |
| 5 | Scoring 队列 | `ScoringQueue` + `ScoringWorker._run` | 单 asyncio task 串行；`asyncio.to_thread` 卸载 sync AI | OpenRouter | `KEY_TIMEOUT_SECONDS` 默认 120s | 🟢 低 |
| 6 | Daily-report runner | `daily_report.runner.run_daily_report` | sync；内部 `asyncio.run + wait_for(90s)` | OpenRouter；广播飞书 | oneshot 90s | 🟢 低 |
| 7 | AI oneshot 同步包装 | `ai.oneshot.generate_text` | `asyncio.run`：禁止从 async 上下文调用 | — | wait_for | 🟡 隐患 |
| 8 | Recovery / 启动校验 | `Workspace.recover` | 启动时持锁；含 commit/push | git 网络 | 无 | 🟠 中 |
| 9 | Relevance scanner | `schedule_hourly_scan` | 周期任务 | 索引文件 + DB | 间隔可配 | 🟢 低 |
| 10 | 前端 AI 流消费 | `Dashboard.sendMessage` | `AbortController` 仅响应用户 stop | 浏览器 SSE | 无 inactivity timer / 无重试 | 🔴 高 |
| 11 | AIPane 单流互斥 | `activeStream` + `blockedByOtherThread` | UI 全局单流 | UI 状态 | 已有 | 🟢 低 |
| 12 | 多 worker 进程并发 | uvicorn `--workers N` | #1 的锁不跨进程 | git push 非 ff | git_ops.push 重试 | 🟠 中 |

**结论**：风险约 80% 集中在两个地方——**进程级 `write_lock` 把"本地写盘"和"git 网络"绑成一坨**、**AI 流式接口缺少端到端超时/心跳/重试语义**。这两条同时也是 Terry/Dengke 反馈的两个症状的根因。

---

## 3. 总体设计原则

### 3.1 不动点（本方案不动的东西）

- Pivot "代码即数据"的 Git/Markdown 落盘契约
- 单实例部署 + SQLite
- SSE 协议向后兼容（只新增字段/事件）
- AI 厂商抽象继续走 OpenAI 兼容协议

### 3.2 要动的

- 把 git 网络从请求生命周期里剥离
- 给所有涉及外部 I/O 的链路加上 **deadline / heartbeat / cancel / retry / 降级** 五件套
- 前端 AI 流补齐"等待—缓慢提示—超时—失败兜底—一键重试"的全状态机

---

## 4. 方案 A：Write Pipeline（**P0**）

### 4.1 解决什么

写盘 = 拿锁 → pull → 写文件 → commit → push → 放锁。响应时间 = 用户感受。

### 4.2 改造结构

拆成两段，请求路径里**只做本地写**，git push 异步化：

```
[请求生命周期]                          [后台 GitWorker]
  acquire fs_lock                         pop GitPushJob
  (lazy) git pull --rebase                git push (退避 + 失败重排队)
  写 .md / .index.yaml                    记录 last_push_at / last_error
  git add + commit                        失败超阈值 → admin 红条告警
  release fs_lock
  enqueue(GitPushJob) ─────────────────────┘
  HTTP 200 返回
```

### 4.3 关键决策

| 项 | 决策 | 理由 |
|---|---|---|
| `pull` 频率 | lazy：启动时 + 距上次 > N 秒 + webhook 触发 | 当前每次写都 pull 是最大浪费 |
| `push` 模式 | 后台 outbox（fire-and-forget） | 用户对"远端 backup"无等待诉求 |
| `outbox` 落地 | SQLite 表 `git_push_outbox` | 重启可重放，与 `recovery.py` 整合 |
| `fs_lock` 类型 | 仍是 `threading.Lock`（单实例） | 多实例时切文件锁/advisory lock |
| 写失败处理 | 不回滚 commit，只重试 push | 本地是真相源，远端是 backup |

### 4.4 兼容性

对调用方完全透明：`Workspace.write_session(...)` 签名不变，内部切换到 `_write_session_local + GitOutbox.enqueue`。

### 4.5 可视化对比

```
现状：
  [用户点发布] ─HTTP─[拿锁]─[pull 5s]─[写盘]─[commit]─[push 10s]─[放锁]──→ 返回
              ↑ 用户必须等完整 15s+ ↑

改造后：
  [用户点发布] ─HTTP─[拿锁]─[写盘]─[commit]─[放锁]──→ 返回 (200ms)
                                              │
                                              └→ 入 outbox → GitWorker 后台 push
```

---

## 5. 方案 B：AI Runner（**P0**）

### 5.1 解决什么

`server/ai/client.py:60` 写死 `read=120.0`；前端 `AIPane.tsx` 只有装饰性轮播提示，没有 inactivity / 心跳 / 自动取消 / 重试入口。

### 5.2 统一 AI 调用契约

抽出 `server/ai/runner.py`，所有 AI 调用入口（chat_matter / oneshot / scoring / daily_report）走同一套：

```python
@dataclass
class AICallSpec:
    purpose: Literal["chat", "summary", "scoring", "daily_report"]
    soft_timeout_s: float        # 触发 slow 提示
    hard_timeout_s: float        # 触发取消 + 失败
    heartbeat_interval_s: float  # SSE 心跳事件间隔
    retry: RetryPolicy           # max_attempts, backoff, retry_on
    cancel_token: CancelToken
```

### 5.3 SSE 协议扩展

新增事件（不破坏现有字段）：
- `data: {heartbeat: {since_last_token_ms: 11800}}`
- `data: {error: {code, retryable, message}}`

终止 sentinel `data: [DONE]` 保留。

### 5.4 关键决策

| 项 | 决策 |
|---|---|
| 超时分桶 | chat 60s soft / 180s hard；summary 20s/60s；scoring 沿用 120s |
| 心跳频率 | 10s 无 token 即发一条 |
| 重试策略 | retryable=true 时前端给"重试"按钮，**不自动重试** |
| 取消传播 | 前端 abort / hard timeout → 后端 `await resp.aclose()` 立即释放上游 |
| 错误归一化 | 统一 `AIError(code, retryable, user_message_zh)` |

---

## 6. 方案 C：JobRunner 任务编排（**P1**）

### 6.1 定位

不是新功能层，而是 A 和 B 的承载基础设施。把目前散落在请求线程里的长任务统一搬上来，避免后续每加一个异步任务就再造轮子。

### 6.2 三种 UX 模式（任务路由表）

| 任务 | UX 模式 | 用户感受 | 实现 |
|---|---|---|---|
| Git push（A 的 worker） | ① fire-and-forget | 无感 | outbox + 后台 worker |
| Publish 写盘 / commit | ② sync wait | 转圈，毫秒级返回 | 直接走 fs_lock，**不入池** |
| AI chat / summary | ③ 流式 SSE | 实时 token | 池层管限流取消，协议层不变 |
| Scoring 触发 | ① fire-and-forget | 无感 | 现状沿用 |
| Daily-report 手动重跑 | ③ 轮询 + 进度 | 显式进度页 | 新增 |
| 改 visibility / 设置 | ② sync wait | 转圈，毫秒级返回 | 不入池 |

### 6.3 实现骨架

沿用 `server/scoring/worker.py` 已经验证的 `ScoringQueue + ScoringWorker` 模式抽通用版：

```python
class JobRunner:
    """通用作业池：每类任务一个 worker，跨 worker 并行，单 worker 内部串行。"""
    workers: dict[str, asyncio.Task]      # "git_push" / "ai_summary" / ...
    semaphores: dict[str, asyncio.Semaphore]   # 每类的并发度 + 限流
    outbox: SQLite                         # 持久化 + 重启重放
```

### 6.4 约束

- 每个 worker 类内部串行（OpenRouter 限流友好；与 scoring 现状一致）
- 跨类并行（git push 不会被 scoring 拖住）
- 持久化作业仅限 fire-and-forget 类（重启可重放）；流式 SSE 类不持久化（用户在则在，断了重发）
- 每类 worker 的并发度从 settings 可配

### 6.5 配套机制

- **健康度门控**：连续 3 次 AI 失败 → 短熔断 30s，期间 AI 入口直接返回"AI 暂时不可用，可改为手写"
- **OpenRouter 限流**：内置 leaky-bucket，超额时进队列；前端给"AI 排队中（位置 N）"
- **写降级**：git push 连续失败 → 仅本地保存 + admin 红条提示，**绝不阻塞用户写作**

---

## 7. 三方案协作关系

```
请求层 (FastAPI handler)
  │
  ├─ 写盘类 ──→ 方案 A (write_session_local) ──→ enqueue ──┐
  │                                                          │
  ├─ AI 类  ──→ 方案 B (AICallSpec + SSE 心跳) ──→ enqueue ─┤
  │                                                          ↓
  └─ 短任务 ──→ 直接处理 (≤200ms)                    方案 C JobRunner
                                                              │
                                          ┌───────────────────┼───────────────┐
                                          ↓                   ↓               ↓
                                    git_push worker     ai_summary worker  scoring worker
                                    (① fire-and-forget) (③ 流式)         (① fire-and-forget)
```

- **A：性能根因修复**——把"用户必须等"的部分压到 ms 级
- **B：体验根因修复**——AI 慢/挂时用户一定有反馈和出口
- **C：基础设施**——A 和 B 落地的统一承载，未来扩展不重造轮子

---

## 8. 前端 AI 交互状态机

### 8.1 五态扩展

把 AIPane 的状态机从现在的 `loading | streaming | idle` 三态扩成五态：

```
idle → connecting (≤2s) → streaming → finished
                    ↘ slow (soft timeout 命中) ↘ stalled (hard timeout) → failed
                                                       ↘ user_aborted
```

### 8.2 落地清单

| # | 触发 | UI 表现 | 行为 |
|---|---|---|---|
| F1 | `connecting > 2s` | "正在连接 AI…"（替换原本的颗粒度轮播） | — |
| F2 | `slow`（12s 无 token） | 顶部黄色细 banner："AI 响应较慢，可继续等待或停止"，配"停止/重试"按钮 | 后端 heartbeat 事件驱动 |
| F3 | `stalled`（60s 无 token） | 自动 abort + 错误气泡："AI 没有响应，请重试" + 一键"重试" | 计时器 + AbortController |
| F4 | `failed` 含 retryable | 错误气泡显示厂商 code，按钮"用相同上下文重试一次" | 复用同一 history |
| F5 | 草稿生成失败 | 已有 `toast.warning` 保留；额外在 NewMatter 提示"AI 暂时不可用，可手写后再发布" | 保证 body_source 不会被误置为 ai |
| F6 | 写盘等待 > 3s | 发布按钮旁"提交中…"；超过 10s 转为"网络较慢…" | 与方案 A 配合后这里很少触发 |
| F7 | 全局异常 | 顶栏小红点 + 详情；不再依赖 toast.error 一条而过 | 跟现有 sessionExpired 一致 |

### 8.3 接口对接

前端 `Dashboard.tsx` 的 `streamAIChat` 增加：
- `softTimeoutMs` / `hardTimeoutMs` 两个 timer，token 到达即重置
- `onHeartbeat` 回调驱动 F2/F3 文案
- `retry()` 方法：保持 `historyForApi` 不变，重新打开 SSE

---

## 9. 验收指标

### 9.1 技术指标

- 发布类接口 P95 ≤ 300ms（不含 AI），P99 ≤ 1s
- AI chat 流无 token 间隔 P95 ≤ 5s，超 12s 必出现 slow banner，超 60s 必出现重试入口
- 同进程 100 并发写无死锁、无 push 堆积 > 30s
- 至少 2 类 worker 并行运行不互相干扰（git_push + ai_summary）
- 重启服务，outbox 内 pending push 可自动重放
- 模拟 git push 延 10s，用户感受 ≤ 1s

### 9.2 体验验收（直接对 Terry/Dengke 的反馈）

- 复现 Terry "提交时 AI 阻塞 1–2 分钟无提示"已被消除（要么不再阻塞，要么有清晰提示 + 重试）
- AI 助手回复偶发超时报错带"重试"入口，不需要用户复制重发
- AI 暂不可用时，写作主流程不受阻塞

---

## 10. 关于 git 的可替换性

### 10.1 立场

git 在 Pivot 当前架构里承担多重职责，但**没有任何一项是不可替代的**。方案 A 已将 git push 从写路径异步化，git 实际上已退化为"远端归档/备份"角色。**本方案不主动替换**，仅记录可替换性矩阵，便于未来按需切换。

### 10.2 可替换性矩阵

| git 当前职责 | 可选替代 | 触发切换的信号 |
|---|---|---|
| 文件存储 | 本地 fs / 对象存储（S3 / OSS / MinIO） | 引入图片附件 / 多实例部署 |
| 版本历史 | DB `audit_log` 表（actor + resource + diff JSONB + at） | 需要按用户/资源粒度查历史 |
| 远端同步 / 备份 | DB WAL + 对象存储版本控制 | 切 PG 时一并替换 |
| 用户 take-out | `GET /api/export` 出 ZIP（含 .md + manifest.json） | 用户反馈"不会用 git" |
| MCP / AI 上下文 | MCP 直接读 fs 或 DB（接口层不变） | git 切走时同步切换 |
| 崩溃恢复 | DB 事务 + outbox 重放 | 已在方案 A 中部分落地 |

### 10.3 前置确认事项

Pivot 是否对外承诺过"git 原生"——若有产品契约，则 git 留作可选导出方式而非主存储；若无，则切换路径打开。

---

## 11. SaaS 多租户扩展性

### 11.1 立场

本方案的目标是**让架构有能力支持 SaaS 多租户**，**不做提前设计、不做提前实施**。下表是为"能支持"必须埋好的接口位，仅此而已。具体的 SaaS 数据库选型、租户隔离机制、迁移路径等，等业务信号明确后再设计。

### 11.2 接口位

| # | 接口位 | 改造内容 | 未来切 SaaS 时的收益 |
|---|---|---|---|
| 1 | `tenant_id` 字段 | 所有 DB 表 + 内存缓存 key + audit log 都带 tenant_id；单租户期固定为 `"default"` | 不用回头加字段、改索引 |
| 2 | 存储抽象 | `Storage` 接口（read/write/list/delete）替代直接的 fs 调用 | 切对象存储零成本 |
| 3 | DB 适配层 | 所有 SQL 走 `server/db.py` 的 `Database` 类；SQL 方言贴近 ANSI，避免 SQLite 专属语法 | 切 PG 时只改适配层 |
| 4 | 命名锁接口 | `LockProvider.acquire(scope, resource_id)` 替代裸 `threading.Lock` | 切跨进程锁（PG advisory lock 等）零成本 |
| 5 | 多租户配置读取 | `settings.get(key, tenant_id=...)` 替代全局单例 | AI key、邮件等配置可按租户读 |

**约束**：这 5 项**只做接口抽象，不做实现切换**。SQLite、本地 fs、threading.Lock 等当前实现完全保留。

### 11.3 明确不做的事

- ❌ 不切 PG（保留 SQLite）
- ❌ 不做 RLS / Schema-per-tenant 等租户隔离实施
- ❌ 不做 SQLite → PG 迁移设计
- ❌ 不做对象存储接入（保留 fs 实现）
- ❌ 不做多实例部署 / 跨进程锁实施

以上几项均不在本方案范围内，需单独开贴做设计。

---

## 12. 本方案不涉及的内容

- ❌ 多实例部署的跨进程锁（仍单实例，注释留位）
- ❌ OpenRouter 余额/限流熔断的具体实施（C 方案中保留接口位）
- ❌ AI 聊天的多模型路由 / 模型切换 UI
- ❌ 历史 stuck 状态数据的迁移（一次性脚本另列）
- ❌ 工作流可配置化（think/act/verify/result/insight 语义保持代码硬编码）
- ❌ 完全去 git 化（参 §10）
- ❌ SaaS 数据库迁移（参 §11）
- ❌ 图片 / 附件支持（zhangbo 在 002 提到，作为另一条 act 单独处理）

---

## 13. 待评审的开放问题

1. **Pivot 是否对外承诺过"git 原生"** —— 决定 §10 的切换路径是否打开
2. `pull` 的 lazy 阈值定多久？建议 60s，但若 webhook 已上线可以更长
3. `git push` 失败超阈值的告警通道走哪里？admin 红条 vs 飞书 DM
4. SSE 心跳事件 type 字段如何命名最不冲突？建议 `heartbeat`
5. AI summary 在 NewMatter 模式下是否独立分桶超时（更短，比如 20s/60s）
6. JobRunner 各 worker 的默认并发度 / 限流配额初值

---

## 附：兼容性边界声明

- 现有 SSE 字段全部保留，仅新增字段/事件
- 现有 `Workspace.write_session(...)` 签名保留（内部实现切换）
- Scoring / Daily-report 的现状不动，未来纳入 JobRunner 时无破坏性改动
- MCP 协议不受影响（接口层不变）
