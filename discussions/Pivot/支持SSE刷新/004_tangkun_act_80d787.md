---
type: act
author: tangkun
created: '2026-04-28T19:49:09+08:00'
index_state: indexed
---
# Matter 实时刷新机制 · 技术设计方案V2

| | |
|---|---|
| 状态 | 已实施（feat/pivot-matter） |
| 关联文档 | [`matter-sse-realtime-plan.md`](./matter-sse-realtime-plan.md)（实施计划）<br>[`003_tangkun_reply_1b1276.md`](./003_tangkun_reply_1b1276.md)（v1 设计） |
| 涉及组件 | `server/events.py` · `server/api/matters_events.py` · `web/src/events/*` · `Dashboard.tsx` · `MatterDetailPane.tsx` |

---

## 一、需求背景

### 1.1 当前系统现状

Pivot Web 是纯请求 / 响应模型，多人协作时：

- A 在浏览器创建 matter / 追加文件 / 加评论
- B 的浏览器**完全感知不到**，需手动刷新页面才能看到

具体现状：

- **服务端**早就在写入路径 emit 了三类领域事件（`server/events.py` + `publish.py:405/510/598`）：
  - `TOPIC_MATTER_CREATED`
  - `TOPIC_FILE_APPENDED`
  - `TOPIC_COMMENT_APPENDED`
- **未读数**由后端按文件名字典序计算（`server/inbox.py:217-241`），只要前端重 fetch `/api/matters` 就自动正确
- **详情页**已有 `visibilitychange` 兜底逻辑（`MatterDetailPane.tsx:269-275`），但仅限单页面、不全局

总结：**"事件源"全齐，缺的只是从服务端到浏览器的实时通道**。

### 1.2 版本变更原因

v1（[`003_tangkun_reply_1b1276.md`](./003_tangkun_reply_1b1276.md)）整体可跑，但有两处选型把简单事情做复杂了：

**用文件 path 当事件主键**

v1 的事件长这样：
```
{ "path": "discussions/<category>/<matter_id>/<filename>", "timestamp": ... }
```

前端拿到这个 path，要先 split 拆成 category / matter_id / filename，再去判断是新文件还是新评论；然后服务端为了让"只追加一张卡"成立，还得新加一个 `GET /api/matters/{matter_id}/files/{filename}` 单卡接口。前后端都得围着这个 path 转一圈。

**v2 直接换成 `matter_id`**：事件只说"哪个 matter 变了"，前端直接拿这个 ID 调既有的 `GET /api/matters/{id}`，列表页调 `GET /api/matters`，单卡接口不用了，path 解析也不用了。

**用 Streamable HTTP（fetch + ReadableStream）替代 SSE**

v1 选 Streamable HTTP 的唯一动机：fetch 能塞 `Authorization` header，所以 PAT 客户端也能订阅。问题是：

- 用 `EventSource` 浏览器就免费给你**自动重连、`Last-Event-ID` 断点续传、心跳保活**；

**v2 直接用 SSE / EventSource**：浏览器原生支持，调试链路（DevTools → EventStream tab）现成，服务端在 `server/api/ai.py` 里已经用过 `StreamingResponse(media_type="text/event-stream")`，依赖齐技术栈熟。

**其他顺手做的事**

- v1 把所有事件归一为 `matter_change` 一个 kind，业务语义全推给前端从 diff 里反推。v2 拆成 `matter.created` / `matter.updated`，看名字就知道走哪条分支。
- v1 自己造了一套"客户端记 `lastFetchAt`、详情页拉取不更新它"的去重规则。v2 直接 300ms debounce 合并，少一堆边界条件。
- v1 列了"事件来了之后要做的 7 件事"（fetchMatters + fetchInbox + 条件 fetchMatter + 按钮可见性矩阵 + 草稿合法性 + 高亮 + 红点）。v2 砍到 3 件。
- v1 写了"刷新结果没变化时页面不应闪烁"，但没说怎么做。v2 给了具体机制：`sameMatters` 指纹比较 + 稳定的 React `key` + 静默路径不动 `isLoading` + debounce。

---

## 二、目标

### 2.1 功能目标

| # | 场景 | 期望表现 |
|---|---|---|
| 1 | 双 Tab：A 新建 matter | B 列表 1s 内出现新条目，无 skeleton 闪现 |
| 2 | A 在 matter X 回帖，B 在列表 | B 看到 X 的 `unread_count + 1`，无整页闪烁 |
| 3 | A 回帖时 B 停留在 X 详情页 | B 看到新帖追加底部，滚动位置不变 |
| 4 | DevTools 关网 30s 再开 | 重连后自动补刷一次 |
| 5 | iOS 锁屏 5min 解锁 / BFCache 前进后退 | 回前台 refetch 一次 |
| 6 | 飞书 WebView 后台冻结再解冻 | visibility + 重连去抖后 refetch 一次 |
| 7 | 同 matter 1s 内 5 条事件 | 只触发一次 refetch（Network 验证） |
| 8 | 数据无变化时（重复标记已读等） | `<MatterRow>` 不 commit（React DevTools Profiler 验证） |

### 2.2 非目标（明确**不做**）

- ❌ 不引入 WebSocket
- ❌ 不引入在线用户列表 / 房间 / 已送达回执
- ❌ 不改 `server/inbox.py` 的未读计算
- ❌ 事件不携带业务正文（只发"哪个 matter 变了"）
- ❌ 不做 PAT 客户端实时推送（PAT 场景仍走主动轮询 REST）
- ❌ 不引入跨进程消息队列（单进程 in-process bus 已够；多副本部署再桥接 Redis pub/sub）

---

## 三、技术方案

### 3.1 ⭐ 协议选型：为什么 SSE，不 Streamable HTTP

这是本设计**最关键的决策**，单独成节。

#### 选 SSE 的理由

| 维度 | EventSource (SSE) | fetch + ReadableStream (Streamable HTTP) |
|---|---|---|
| 浏览器原生支持 | ✅ W3C 标准 API | ✅（fetch + getReader） |
| **自动重连** | ✅ **EventSource 内置** | ❌ 自己 setTimeout + 指数退避 |
| **`Last-Event-ID` 重放** | ✅ **EventSource 重连时自动带上请求头** | ❌ 自己存、自己塞 |
| 心跳保活 | ✅ EventSource 自动忽略 `:` 注释帧（`: ping\n\n`） | ⚠️ 需自己解析帧/丢弃 |
| 自定义请求头 | ❌ 不能加 | ✅ 能加任意 header |
| HTTP method | 仅 GET | 任意 method |
| 是否支持双向 | 单向（服务端 → 客户端） | 单向 |
| 调试体验 | DevTools 有 **EventStream** tab，结构化展示 | 只能看 raw bytes |
| 实施代码量 | 服务端 ~150 行，前端 ~80 行 | 服务端类似，**前端 ~300+ 行**（自己实现重连/重放/心跳） |

**只有当 `自定义请求头` 或 `非 GET 方法` 是硬需求时，才必须选 Streamable HTTP**。本期需求：

- 鉴权：浏览器 cookie 自动带 → SSE 够用
- 初始化参数：无 → GET 即可
- → SSE 完胜

v1 选 Streamable HTTP 的唯一理由是"PAT 客户端要加 `Authorization` header"。但：
- PAT 客户端**不是本期目标用户**

#### 已有的 SSE 案例

服务端在 `server/api/ai.py:309 / :504` AI chat 端点已经用过 `StreamingResponse(media_type="text/event-stream")`——技术栈熟、依赖齐、调试链路通。

#### 结论

**选 SSE，将 PAT 客户端的实时通道留作未来按需扩展。**

### 3.2 事件协议

端点：`GET /api/matters/events`，鉴权沿用 `current_user`（cookie session）。

```
:connected           # 首帧：连接建立标记

event: matter.created
id: 1714110123456-1
data: {"matter_id":"abc","reason":"created","actor":"alice","at":"2026-04-25T10:22:03+08:00"}

event: matter.updated
id: 1714110125001-7
data: {"matter_id":"abc","reason":"file_appended","actor":"bob","at":"2026-04-25T10:22:05+08:00"}

event: matter.updated
id: 1714110127500-12
data: {"matter_id":"abc","reason":"comment_appended","actor":"carol","at":"2026-04-25T10:22:07+08:00"}

: ping     # 每 25s 一帧（注释帧），穿透代理空闲超时
```

#### 字段定义

| 字段 | 类型 | 含义 |
|---|---|---|
| `matter_id` | string | 哪个 matter 变了 |
| `reason` | `"created" \| "file_appended" \| "comment_appended"` | 变更类型，前端按此分流 |
| `actor` | string | 触发用户的 pinyin（仅日志/调试用，UI 不必显示） |
| `at` | string (ISO 8601) | 服务端 emit 时刻，原样透传 `Event.at` |

#### Topic 映射

```
TOPIC_MATTER_CREATED   → event=matter.created   reason=created
TOPIC_FILE_APPENDED    → event=matter.updated   reason=file_appended
TOPIC_COMMENT_APPENDED → event=matter.updated   reason=comment_appended
```

**不广播的 topic**（`server/api/matters_events.py:_TOPIC_MAP` 故意不收录）：

- `TOPIC_STATUS_CHANGED` / `TOPIC_RESULT_CREATED` —— 这两类总是和一次 file 写入捆绑触发，对应的 `matter.updated` 已经发过；独立转发只会让前端去重一次。
- `POST /api/matters/{id}/read`、`/favorite` —— 本端动作，自己点的、自己刷新，不发给其他人。

#### 关键约束

- 事件**不携带正文**——拿到 `matter_id` 后由前端走 `/api/matters` 或 `/api/matters/{id}` 取真相
- `reason` 字段允许前端**按场景分流**，不需要再 diff 反推
- ID 格式 `<ms-ts>-<seq>`（毫秒时间戳 - 进程内自增序号），保证全局单调、避免同毫秒碰撞

### 3.3 服务端实现

事件流端点 `/api/matters/events` 落在 `server/api/matters_events.py`，由 `server/app.py` 注册到 FastAPI 应用。它本身不再埋任何 emit 点：事件源是已有的 `server/events.py` 进程内 pub/sub，`publish.py` 里 matter 创建 / 文件追加 / 评论追加三条写入路径早就 emit 了对应领域事件，每条都带 `matter_id`、`actor`、`at` 三个字段，本期没有改动 publish 层。

每条客户端 SSE 连接的生命周期分四段：

- **进入**。Handler 立刻向事件总线注册一个回调，并为这条连接分配一条私有 `asyncio.Queue` 作 backlog 缓冲，上限 256；同时记下 `Last-Event-ID` 请求头（可能为空）。
- **入队**。总线回调收到领域事件后，先用一张 topic 映射表把 internal topic 翻成 SSE 的 `event` 名加 reason；映射表里没有的 topic（如 `matter.status_changed` / `matter.result_created`）直接忽略——它们总是和 file 写入捆绑触发，对应的 `matter.updated` 已经发过。命中映射的事件构造成一帧 SSE，先 append 到进程级 ring buffer（容量 512）作为重放窗口，再通过 loop 的 `call_soon_threadsafe` 安全投递到本连接的 queue。绕一道 loop 调度的原因：事件总线的调用方不一定在事件循环线程里，跨线程直接操作 queue 会撞 race。
- **回放 + 实时**。生成器先吐一条 `:connected` 注释帧标记连接建立；如果带了 `Last-Event-ID`，再线性扫一遍 ring 把那个 id 之后错过的帧依次 yield 出去；之后转入正常监听。监听循环每次 await queue 取下一帧，最长等 25 秒就超时一次，超时就吐一条 `: ping` 注释帧（穿透代理空闲超时）；同一循环里检测客户端是否已断，断了就跳出。
- **退出**。`finally` 调 unsubscribe 清理订阅。Queue 溢出（慢订阅者堆积超过 256 帧）只 warning + 丢帧，由客户端的 resume 路径全量兜底，不阻塞 emit 侧。

帧 ID 是"毫秒时间戳-进程级自增序号"形式，序号来自模块级单调计数器，避免同毫秒多事件碰撞、保证全局有序。客户端的 `Last-Event-ID` 拿这个串原样回传，服务端线性扫 ring 找到匹配位置后回放后续帧；ring 上限 512 已经覆盖绝大部分断线场景，超出窗口的事件丢失由 resume 全量刷新兜底。

响应头侧加了三项兼容性配置：禁缓存（`Cache-Control: no-cache`）、长连接保持（`Connection: keep-alive`）、关闭 Nginx 输出缓冲（`X-Accel-Buffering: no`）；前两项给浏览器和反代用，最后一项确保每帧立即 flush 给客户端。

### 3.4 客户端实现

新增三个文件全部放在 `web/src/events/` 下：

- `types.ts` —— `MatterEvent` 联合类型，包含两条 wire 事件（`matter.created` / `matter.updated`）加一条本地合成事件（`resume`），订阅方按 `evt.type` 分流。
- `MatterEventsProvider.tsx` —— 根 Provider 与 `useMatterEvents` 订阅 hook。
- `scheduleRefresh.ts` —— 按字符串 key 把短时间内的多次刷新合并成一次的 debounce 工具，默认 300ms。

整个 App 路由树包在 `App.tsx` 顶层的 `<MatterEventsProvider>` 内，全局共用一条 SSE 长连接和一个 listeners 集合，所有页面订阅都从同一个 dispatch 路径 fanout——避免了重复连接和重复拉数据。

Provider 内部三件事：

- **维持 SSE 连接**。用浏览器原生 `EventSource` 连 `/api/matters/events`（cookie session），分别给 `matter.created` 和 `matter.updated` 两个 event 名各注册一个监听器，收到一帧就解析 JSON 后 dispatch 给所有订阅 listener。listeners 存在 ref 而非 state，订阅 / 取消不会触发 Provider 自身 re-render，dispatch 引用保持稳定。
- **区分首连与重连**。监听 `EventSource` 的 open 事件，本地维持一个 `hasOpenedOnce` 标记。第一次 open 视为初次挂载（页面已经各自首屏 fetch 过一遍），仅置标、不发任何信号；第二次起的 open（即网络恢复后的自动重连）才 dispatch `resume`，避免双拉。
- **可见性兜底**。同时挂三类浏览器事件：`document.visibilitychange` 变为 visible、`window.focus`、`window.pageshow` 且 `persisted=true`（对应 iOS BFCache 恢复）。任一触发都 dispatch `resume`。这样移动端 WebView 冻结解冻、Tab 切回前台、SSE 还没及时重连等场景全部能被兜住。`resume` 是订阅侧"全量兜底刷新"的统一入口。

订阅侧两类用法，关键区别在过滤粒度：

- **列表页 `Dashboard`**。Handler 接到任何带 `matter_id` 的事件，或 `resume` 信号，都用 key `matters-list` 调 `scheduleRefresh` —— 300ms 内多事件被合并成一次 `fetchMatters()`。回包后用指纹比较函数 `sameMatters` 判定是否真有变化：字段不变就把同一引用回设给 state，React 整个列表完全不 commit；只有真正变化的行才 re-render（依赖稳定的列表 key）。指纹覆盖的字段是 id、`updated_at`、`unread_count`、`file_count`、`favorite`、`current_status`、`title` 七项。
- **详情页 `MatterDetailPane`**。Handler 多一层过滤——`resume` 全放行，wire 事件只有 `evt.matter_id` 等于当前打开的 matter 才放行；其它 matter 的事件直接忽略，避免误刷本页。debounce key 用 `detail:<matter_id>`，与列表页完全正交，两边各自合并。命中后先用 `sameDetail` 指纹比较的方式静默刷详情，然后**仅当页面真在前台**才补一次"标已读 + 侧栏 reload"——后台 tab 不动，避免被偷偷清零未读。

旧的本地 visibility 监听（原 `MatterDetailPane.tsx:269-275`）已删除，统一由 Provider 全局接管，订阅入口从两条收敛到一条。

### 3.5 防闪烁机制（明确化）

v1 写了"不闪烁"但没给方法。本设计明确四条：

1. **指纹比较 + 引用复用**：`setMatters(prev => sameMatters(prev, next) ? prev : next)` —— React 看到 `prev === next` 不 re-render
2. **稳定 React key**：列表用 `matter.id`，时间轴用 `post.filename`，避免 React 误判 mount/unmount
3. **静默路径不动 `isLoading`**：避免 skeleton 闪现
4. **300ms 去抖**：`scheduleRefresh(key, callback)` 同一 key 在 300ms 内只 fire 一次，合并多事件

### 3.6 兜底机制（漏事件防护）

| 漏事件场景 | 检测 | 动作 |
|---|---|---|
| Tab 切回前台 / 窗口拿回焦点 | `visibilitychange` / `focus` | Provider 派发 `resume` |
| iOS BFCache 恢复 | `pageshow` 且 `event.persisted === true` | Provider 派发 `resume` |
| SSE 断后重连 / 飞书 WebView 解冻 | `EventSource.onopen` 第二次起 | Provider 派发 `resume` |

订阅方收到 `resume` = 全量 refetch（`Dashboard` 刷列表，`MatterDetailPane` 刷当前 matter 详情）。**`resume` 跟 `matter.updated` 走同一个 `scheduleRefresh` 通道**，去抖合并，不会重复请求。

---

## 四、时序图

### 4.1 正常推送链路（A 写入 → B 同步）

```
[A 浏览器]            [Server]                [事件总线]            [B 浏览器]
   │                     │                       │                      │
   │ POST /matters/{id}  │                       │                      │
   │ /files              │                       │                      │
   │────────────────────►│                       │                      │
   │                     │ publish_matter_append│                      │
   │                     │────► write to git    │                      │
   │                     │                       │                      │
   │                     │ emit(FILE_APPENDED,  │                      │
   │                     │      matter_id, ...) │                      │
   │                     │──────────────────────►│                      │
   │                     │                       │ on_event 回调       │
   │                     │                       │ 构造 SSE 帧          │
   │                     │                       │ 入 ring + put queue │
   │ 200 OK              │                       │                      │
   │◄────────────────────│                       │                      │
   │                     │                       │                      │
   │                     │  GET /matters/events  │                      │
   │                     │  (B 已建立的长连接)   │                      │
   │                     │                       │                      │
   │                     │  event: matter.updated│                      │
   │                     │  data: {matter_id,    │                      │
   │                     │    reason="file_      │                      │
   │                     │    appended", ...}    │                      │
   │                     │──────────────────────────────────────────────►│
   │                     │                       │                      │
   │                     │                       │  scheduleRefresh    │
   │                     │                       │  (debounce 300ms)   │
   │                     │                       │                      │
   │                     │◄─ GET /matters ──────────────────────────────│
   │                     │   (B 静默 refetch)    │                      │
   │                     │                       │                      │
   │                     │ 200 + 列表 JSON       │                      │
   │                     │──────────────────────────────────────────────►│
   │                     │                       │   sameMatters 比较  │
   │                     │                       │   有变 → setMatters │
   │                     │                       │   无变 → 跳过       │
```

### 4.2 断线重连 + Last-Event-ID 重放

```
[B 浏览器]                                                        [Server]
   │                                                                  │
   │  EventSource 已连接,接收到 id="1714110125001-7" 后断网          │
   │     ✗                                                            │
   │                                                                  │
   │  期间服务端继续 emit 5 条事件 → ring buffer:                    │
   │     [.., id=1714110125001-7, id=...8, id=...9, ..., id=...12]   │
   │                                                                  │
   │  网络恢复,EventSource 自动重连                                  │
   │  请求头自动带:                                                   │
   │     GET /api/matters/events                                      │
   │     Last-Event-ID: 1714110125001-7  ────────────────────────────►│
   │                                                                  │
   │                                       服务端从 ring 找到该 id    │
   │                                       回放后续 5 条帧            │
   │                                       然后转入实时模式           │
   │                                                                  │
   │  ◄───── replay 5 帧 ─────                                        │
   │  ◄───── 实时帧持续推 ─────                                       │
   │                                                                  │
   │  EventSource.onopen 触发(第二次),Provider 派发 resume          │
   │  → scheduleRefresh 触发 fetchMatters() 全量兜底                  │
```

### 4.3 防闪烁路径

```
[Server emit FILE_APPENDED for matter_id=X]
        │
        ▼
[B 浏览器收到 matter.updated 帧]
        │
        ▼
[scheduleRefresh("matters-list", refreshMattersSilently)] ──┐
                                                            │ 300ms 内多事件合并
        ▼                                                   │
[300ms debounce]    ◄───── 200ms 后又来一帧 ────────────────┘
        │
        ▼
[refreshMattersSilently() → fetchMatters()]
        │
        ▼
[next = await fetchMatters()]
        │
        ▼
[setMatters(prev => sameMatters(prev, next) ? prev : next)]
        │
        ├── prev === next (引用相同) ──► React 不 commit ──► 列表零闪烁 ✓
        │
        └── 数据真的变了 ──► React commit ──► 仅变化的 row 重渲染
                                              (因为 key={matter.id} 稳定)
```

---

## 五、实施分工

### 5.1 阶段拆解

```
Phase 0 ─┬─► Phase 1 (后端) ─┐
         └─► Phase 2 (前端骨架) ─┬─► Phase 3 (列表) ─┐
                                  └─► Phase 4 (详情) ─┴─► Phase 5 (联调)
```

Phase 1 与 2 可并行；Phase 3 与 4 可并行。

| Phase | 内容 | 负责人 | 工作量 |
|---|---|---|---|
| 0 | 切分支 `feat/matter-sse`；核对 `publish.py:405/510/598` emit payload 字段；缺 `matter_id`/`actor` 补上 | 后端 | 0.5 h |
| 1 | 新建 `server/api/matters_events.py`（端点 + ring buffer + Last-Event-ID 重放 + 心跳）；`app.py` 注册 router | 后端 | 0.5 d |
| 2 | 新建 `web/src/events/{types,MatterEventsProvider,scheduleRefresh}.ts(x)`；`App.tsx` 包 Provider | 前端 | 0.5 d |
| 3 | `Dashboard.tsx` 接入 `useMatterEvents` + `sameMatters` + `refreshMattersSilently` | 前端 | 0.5 d |
| 4 | `MatterDetailPane.tsx` 接入 + `sameDetail` + 删除旧 `:269-275` visibility 监听 | 前端 | 0.5 d |
| 5 | 桌面 / 移动 / 飞书 WebView 联调，验收标准全跑一遍 | 前后端 | 0.5 d |

**合计约 2.5 人日**。

### 5.2 交付清单

```
新增:
  server/api/matters_events.py
  web/src/events/types.ts
  web/src/events/MatterEventsProvider.tsx
  web/src/events/scheduleRefresh.ts

修改:
  server/app.py                              # +1 行注册 router
  web/src/App.tsx                            # +1 层 Provider
  web/src/pages/Dashboard.tsx                # 接入 + sameMatters + refreshMattersSilently
  web/src/pages/MatterDetailPane.tsx         # 接入 + sameDetail + 删除旧 visibility 监听
  server/publish.py                          # 仅当 Phase 0 发现 emit 字段缺失才改
```

### 5.3 风险与回滚

| 风险 | 对策 |
|---|---|
| 反代缓冲 SSE | Headers `X-Accel-Buffering: no`；Phase 5 在生产同款反代验证 |
| ring buffer 512 条不够 | resume 兜底覆盖；监控 `Last-Event-ID` 命中率，必要时 N→1024 |
| `same*()` 漏字段导致漏更新 | Phase 5 显式覆盖：未读 / favorite / 删除 case；指纹缺漏时退化为 `JSON.stringify` 兜底 |
| 飞书 WebView 不支持 EventSource | EventSource 是 W3C 标准，飞书内核（Chromium 108+）支持；如真不支持降级长轮询同一端点 |
| 多副本部署后单进程 ring 不够 | Phase 1 之后再桥接 Redis pub/sub；事件协议不变，仅订阅源换 |

**回滚**：删掉 `App.tsx` 里 `<MatterEventsProvider>` 包裹一行即关闭整个机制；后端 `/api/matters/events` 端点保留无副作用。

---
