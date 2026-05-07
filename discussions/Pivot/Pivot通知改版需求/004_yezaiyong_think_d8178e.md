---
type: think
author: yezaiyong
created: '2026-04-23T15:30:48+08:00'
index_state: indexed
---
# Pivot 通知样式迁移 + Mention 结构化 · 实施设计（v2）

> **本版定位**：范围收缩后的实施方案。summary 相关能力**统一延期**至底层 API 重构完成后。
> **目标完成日**：2026-04-24（半个工作日内可搞定，工作量远小于 v1）

---

## 0. 前置说明

### 0.1 为什么有这份 v2

v1 的原方案（`2026-04-23-summary-notify-migration.md`）包含大量 summary 相关条目（生成、校验、MD/index 双写、客户端拦截）。因**底层 API 即将重大重构**，新设计规范对 summary 的位置改了口径：

- summary 不再写入 MD 文件
- 仅保留在 index 中
- 由新 API 统一要求双写

原方案的 summary 相关实现几周后就要被新 API 重做，**本期不再投入**。本版只保留"通知样式迁移 + mention 结构化"两项。

### 0.2 一句话范围

- **改什么**：服务端飞书通知生成逻辑 + Web 的 post 级深链
- **不改什么**：summary 相关一切（发布 API、客户端提示词、强制升级拦截等）、VSCode 客户端

### 0.3 基线分支（重要）

**不使用**之前 v1 遗留的 feature 分支（`feature/summary-notify-migration` / `feature/summary-notify-v0.0.5`）。那两个分支带了大量 summary 相关改动，本期不该打包一起合。

从 `main` 重新拉新分支：

| 仓库 | 新分支名 |
|---|---|
| `team-pivot-web` | `feature/notify-mention-migration-v2` |
| `vscode-team-pivot` | **不建分支**（本期不改） |

v1 的 feature 分支建议**归档或删除**（下文 §10 有操作清单）。

---

## 0.5 开工前读这些文件

| # | 文件 | 用途 |
|---|---|---|
| 1 | 本文档 | 总览 + 任务清单 |
| 2 | `2026-04-23-summary-notify-migration.md` §附录 C | 原始需求文本（历史） |
| 3 | `teamDocs/appv2/tools/notify.py` L411-L560 | 老版飞书卡片 7 字段版式，本次对标 |
| 4 | `team-pivot-web/server/notify.py` (main 分支) | 新版通知代码，要改 |
| 5 | `team-pivot-web/server/publish.py` (main 分支) | notifier 调用点，可能需要适配新签名 |
| 6 | `team-pivot-web/web/src/pages/ThreadDetailPane.tsx` (main 分支) | 要加 post 级深链 |

---

## 1. 涉及仓库

| 仓库 | 角色 | 本期改动量 |
|---|---|---|
| `team-pivot-web` | 服务端 + Web 前端 | 服务端中等，Web 仅 1 项 |
| `vscode-team-pivot` | VSCode 扩展 | **本期不改** |
| `teamDocs` | 老版 appv2（只读对标） | 不动 |

---

## 2. 核心需求摘要

### 2.1 通知样式迁移（对齐老版 appv2）

- **信息层次 / 版式**：7 字段 info 块（项目 / 主题 / 操作 / 文件 / 时间，加上 @mention 提及块和说明块）
- **移除**：原始文件全文预览、服务端版本号
- **保留**：@ 提及块、说明块、「查看 Web」按钮、其他老版有价值的信息层次（如讨论目录折叠面板）

> 注：**不再展示"摘要"字段**。原 v1 设计里 info 块有 6 或 7 行（含摘要），本版去掉摘要行，其余保持紧凑。

### 2.2 Mention 通知结构化（按新版语义重新设计）

mention 通知（群广播 + 私聊 DM 两种形态）必须明确展示：

1. **圈了谁**：@ 提及块
2. **哪个 thread**：主题行
3. **哪条 post**：帖子文件名行
4. **mention 内容**：说明行
5. **直达该 post 的链接**：卡片按钮 URL 使用 post 级深链（带 `?post=<anchor>#post-<anchor>`），**点击后要落到具体那条 post**，不是落到 thread 顶

---

## 3. 现状快照（以 main 分支为准）

### 3.1 服务端 `main` 分支现状

| 能力 | 位置 | 现状 |
|---|---|---|
| `POST /api/threads` / `/{cat}/{slug}/posts` | `server/api/discussions.py` | ✅ 存在，不要求 summary |
| 飞书卡片 4 个 builder | `server/notify.py` | ⚠️ **卡片版式是"作者 + 200 字正文截断"，信息层次远少于老版 appv2** |
| `notify_standalone_mention` 路由 | `server/api/discussions.py` | ✅ 有，但下游 build_standalone_mention_card 信息不够（缺帖子名 / 相关内容摘录） |
| `_thread_url` 深链 | `server/notify.py` | ⚠️ 只能深链到 thread 顶，没有 post 级锚点 |
| 讨论目录折叠面板 | — | ❌ 没有 |
| 版本号行 | `server/notify.py::_build_card` | ✅ 本来就没有（这条差异 main 已满足） |
| 原文全文 | `server/notify.py::_build_card` | ⚠️ 现在是截断 body 前 200 字作为卡片正文，**算半个"原文预览"要去掉** |

### 3.2 Web 前端 `main` 分支现状

| 能力 | 位置 | 现状 |
|---|---|---|
| 错误 toast 展示服务端 detail | 用 sonner | ✅ 已有 |
| post 级深链 `?post=xxx` + scrollIntoView | `web/src/pages/ThreadDetailPane.tsx` | ❌ **没有**，必须加 |
| 帖子卡片 DOM id=post-xxx | 同上 | ❌ 没有 |

### 3.3 VSCode `main` 分支现状

| 能力 | 现状 |
|---|---|
| UpdateManager 版本自升级 | ✅ 已运行 |
| 通用错误 toast | ✅ 已有 |
| 发布 API 调用 | ✅ 不依赖 summary，和服务端契约吻合 |

**结论：VSCode 本期不动。**

---

## 4. 设计决策

### 4.1 卡片版式（6 字段 info 块，无摘要行）

```
<at 提及块（若有）>
**说明**：{mention_comments}（若有）
**项目**：{category}
**主题**：{thread_slug}
**操作**：{author_name} {action_text}
**文件**：{filename}
**时间**：{update_time}
```

**action_text 映射**：

| 事件 | action_text |
|---|---|
| 新讨论 | `发起了新讨论` |
| 新回复 | `发布了新回复` |
| 状态变更 | `{from_state} → {to_state}` |
| 独立 mention | `在 {kind} 中 @ 提及` |

**行内拼接规则**：
- `@ 提及块` + `说明` + 6 字段 **合并为一段**，行与行用 `<br>` 连接（Feishu schema 2.0 `markdown` tag 对单 `\n` 仍按段落间距渲染，必须用 `<br>` 才紧凑）
- 其他模块（比如讨论目录折叠面板）单独起一段（`\n\n` 分隔）

### 4.2 Mention 深链 URL 格式

```
https://<web_base_url>/auth/entry?next=/t/{category}/{slug}?post={anchor}#post-{anchor}
```

- `anchor` = `filename` 去掉 `.md` 后缀
- `?post=` query 参数给 Web 路由读（决定默认锚定哪条 post）
- `#post-{anchor}` hash 给浏览器自动滚动

Web 侧 `ThreadDetailPane` 在 data 加载完成后 `useEffect` 读 URL 的 `post` 参数，`document.getElementById('post-xxx').scrollIntoView({ behavior: "smooth" })`。

### 4.3 Mention 卡片细节

**群广播（`build_standalone_mention_card`）**：

```
<at user_id="..." ...>
**{author_name}** 提及了以上成员
**项目**：{category}
**主题**：{thread_title}
**帖子**：{target_filename}
**说明**：{mention_comments}
**时间**：{update_time}

**相关内容**：{post_excerpt_truncated_150}  ← 独立段落
```

按钮文案 `查看该帖子`，URL 用 post 级深链。

**私聊 DM（`build_mention_dm_card`）**：

```
**{author_name}** 在「{thread_title}」的 {kind} 中 @ 了你
**帖子**：{target_filename}
**说明**：{mention_comments}

**相关内容**：{post_excerpt_truncated_150}  ← 独立段落
```

按钮文案 `去查看`，URL 用 post 级深链。

### 4.4 讨论目录折叠面板（对齐老版 appv2 §附录 A）

新版通知卡片附带一个 `collapsible_panel`，展开后列出 thread 里所有 post：

```
📄 **[001_alice_proposal_xxx.md]** · alice · 2026-04-23
📄 **[002_bob_reply_yyy.md]** · bob · 2026-04-23
🔷 **[003_carol_reply_zzz.md]** · carol · 2026-04-23    ← 当前 post，蓝色高亮
📄 **[004_alice_reply_www.md]** · alice · 2026-04-23
```

> 注：本期**不展示各 post 的摘要**（原老版每条下面会跟一段 summary）。因为 summary 能力整块延期，这里就只展示 文件名 + 作者 + 日期。新 API 上线后再补摘要渲染。

**样式定义**：
- 当前 post：`<font color='blue'>🔷 **[filename.md]({post_url})** · author · date</font>`
- 其他 post：`📄 **[filename.md]({post_url})** · author · date`
- 面板 header：`📂 讨论目录（N 篇帖子）`

### 4.5 去掉原文 / 版本号

- `_build_card` 里 `_truncate(body, 200)` 那行**删除**（裁掉"截断正文"那 200 字）
- 版本号行 main 分支本来就没有，无需动

### 4.6 Notifier 签名调整

`Notifier.notify_new_thread` / `notify_new_reply` 的参数从 `body: str` 改为 `filename: str`（给卡片展示 **文件** 行用）。本期**不传 summary**（summary 能力延期）。

```python
class Notifier(Protocol):
    def notify_new_thread(self, *, category, slug, title, author_name,
                          filename,  # NEW
                          mention_open_ids=None,
                          mention_comments=None) -> None: ...

    def notify_new_reply(self, *, category, slug, thread_title, author_name,
                         filename,  # NEW
                         mention_open_ids=None,
                         mention_comments=None) -> None: ...
```

`notify_standalone_mention` 保持现有 `post_excerpt` 参数不变。

`FeishuNotifier.__init__` 新增可选 `workspace` 参数（用来给讨论目录折叠面板读全部 post 列表）。

### 4.7 错误码

本期**没有 summary 校验**，服务端发布 API 不新增 422 用例。保留现有 400 / 404 行为。**不**引入 426 Upgrade Required（那是 summary 强制升级拦截的用例，本期延期）。

---

## 5. 任务清单

### 5.1 服务端（`team-pivot-web`）

| Task | 文件 | 说明 |
|---|---|---|
| **S1** 6 字段卡片 `_build_card_6fields` | `server/notify.py` | 新 helper 函数，info 块用 `<br>` 连接；包含 @ 提及 / 说明 / 项目 / 主题 / 操作 / 文件 / 时间，**无摘要行**。替换现有 `_build_card` |
| **S2** `build_thread_card` / `build_reply_card` 适配新版 | `server/notify.py` | 调用 `_build_card_6fields`；header 改为 `新讨论：{title}` / `{author} 回复：{title}` |
| **S3** `build_standalone_mention_card` 重写 | `server/notify.py` | 按 §4.3 群广播版式；带 post 级深链按钮 |
| **S4** `build_mention_dm_card` 重写 | `server/notify.py` | 按 §4.3 DM 版式；带 post 级深链按钮 |
| **S5** 新增 `_post_url(category, slug, filename)` | `server/notify.py` | 生成 `/auth/entry?next=/t/.../?post=xxx#post-xxx` |
| **S6** 新增 `build_thread_directory` + `collapsible_panel` 拼装 | `server/notify.py` | 按 §4.4 样式；FeishuNotifier 构造传入 workspace 供其读 thread |
| **S7** `FeishuNotifier` 构造支持 workspace 参数 | `server/notify.py` | 可选 `workspace: Workspace \| None = None`，None 时跳过讨论目录（用于测试） |
| **S8** `Notifier` Protocol 签名调整 | `server/notify.py` | 按 §4.6 改，`body` → `filename` |
| **S9** `publish.py` 的 notifier 调用点适配新签名 | `server/publish.py` | `publish_proposal` / `publish_reply` 调用处把 `body=body` 改成 `filename=filename` |
| **S10** `server/app.py` 构造 FeishuNotifier 时传 workspace | `server/app.py` | 把 workspace 初始化提前到 notifier 之前 |

### 5.2 Web 前端（`team-pivot-web/web`）

| Task | 文件 | 说明 |
|---|---|---|
| **W1** post 级深链 scrollIntoView | `web/src/pages/ThreadDetailPane.tsx` | 每条 post 外层包 `<div id="post-{anchor}">`；`useEffect` 读 URL `post` query 参数，scrollIntoView |

### 5.3 VSCode

**本期不动。**

---

## 6. 测试与验收

### 6.1 单元测试（pytest）

| # | 测试点 | 文件 |
|---|---|---|
| T1 | 新卡片 6 字段都在 | `server/tests/test_notify.py::test_thread_card_shape` |
| T2 | 新卡片不含原文（无 "**作者**：xxx" + 200 字 body） | `test_card_no_longer_contains_raw_body` |
| T3 | mention DM 卡片含 `**帖子**` / `**说明**` / `**相关内容**` | `test_mention_dm_card_structure` |
| T4 | post_url 带 `?post=` 和 `#post-` | `test_post_url_carries_anchor` |
| T5 | 讨论目录高亮当前 post（蓝色 🔷） | `test_build_thread_directory_highlights_current_post` |

### 6.2 手工验收清单

| # | 验收项 | 预期 |
|---|---|---|
| A1 | 发新帖，群里收到卡片 | 6 字段 info 块 + 讨论目录折叠面板，**无摘要行**、无原文预览 |
| A2 | 发回复，群里收到卡片 | 同上；header 带作者名（`{author} 回复：{thread}`） |
| A3 | 独立 mention，群里收到卡片 | 含帖子名 / 说明 / 相关内容 / 时间；按钮 deep-link 到 post |
| A4 | 独立 mention，被 @ 用户收到 DM | 含「谁 @ 你」lead line + 帖子 / 说明 / 相关内容 |
| A5 | 点击 mention 卡片按钮 | Web 登录后直接定位到被 @ 的那条 post（页面自动滚动，不只是打开 thread） |
| A6 | 讨论目录折叠面板点开 | 当前 post 蓝色 🔷 高亮，其他 📄，点 filename 跳对应 post |
| A7 | 当前页打开某 thread 然后手动在 URL 加 `?post=xxx` | 自动滚动到 xxx |

### 6.3 回归（不要打破已有）

| # | 回归项 | 预期 |
|---|---|---|
| R1 | 发布无 mention 的 new thread | 只广播群卡片，不发 DM |
| R2 | 状态变更卡片 | 继续正常（本期未动 `build_status_change_card`） |
| R3 | 收藏 / 取消收藏 thread | 不受影响 |
| R4 | VSCode 发布流程 | 不受影响（因为服务端 /api/threads 的契约没变） |

---

## 7. 风险与回滚

### 风险矩阵

| 风险 | 严重度 | 缓解 |
|---|---|---|
| 讨论目录读 workspace 失败时卡片整个挂掉 | 中 | `build_thread_directory` 内 try/except，失败时返 `("", 0)`，卡片降级为没有折叠面板 |
| Feishu schema 2.0 对 `<br>` 的渲染跨版本不一致 | 低 | 先 smoke test 看一眼，如果失效改用 `<br/>` 或 `lark_md` 标签 |
| Web 改动破坏 thread 详情页现有滚动行为 | 低 | `scrollIntoView` 只在 URL 有 `post` 参数时触发，无参数时不动 |
| Summary 能力延期后旧老版验收条目怎么办 | 低 | 在 v2 需求文档 §7 已删除相关条目；实施时按 v2 验收表为准 |

### 回滚

所有改动在单个 feature 分支上。如果生产翻车，直接在 main revert 掉合并提交即可（因为原先 main 分支就在运行，回到那个状态）。

---

## 8. 工作量预估

| 任务组 | 预计耗时 |
|---|---|
| S1-S8 服务端改造 | 2-3h |
| S9-S10 publish.py + app.py 适配 | 30min |
| W1 Web post 级深链 | 30min |
| 单测 T1-T5 | 1h |
| 手测 A1-A7 | 1h |
| 缓冲 | 1h |
| **合计** | **~6h（约一个整工作日的一半）** |

比 v1 小很多，因为去掉了：
- Pydantic summary 校验（S1/S2 v1）
- frontmatter 双写（S3/S4 v1）
- AI prompt 改造（C1 v1）
- 客户端本地校验（C2 v1）
- 客户端 API 类型 + 422/426 处理（C3 v1）
- DraftCard summary 预览（C4 v1）
- 版本号升 0.0.5 + release.json（C5 v1）
- 发版 / 强制升级流程（§4.6 v1）

---

## 9. 执行顺序

```
S5 (_post_url helper) → S1-S4 (卡片重写，依赖 _post_url)
                     ↘
S7 (Feishu workspace 构造) → S6 (讨论目录)
S8 (Protocol 签名) → S9 (publish.py 适配)
S10 (app.py 构造顺序)                      ↓
                                         服务端改完
                                             ↓
                                       重启 uvicorn + 冒烟
                                             ↓
                                          W1 Web 改
                                             ↓
                                         端到端联调
```

先做 S5 / S7 / S8 这些"基础设施"类小改动，再做 S1-S4 / S6 的卡片主体，最后 S9 / S10 接线。

---

## 10. 旧 feature 分支处理

v1 留下了两个本地分支：

| 仓库 | 分支 | 建议 |
|---|---|---|
| `team-pivot-web` | `feature/summary-notify-migration` | **归档**：`git branch -m feature/summary-notify-migration archive/v1-summary-notify-migration-2026-04-23`；不删，以备将来 summary 能力回来时 cherry-pick |
| `vscode-team-pivot` | `feature/summary-notify-v0.0.5` | 同上，改名成 `archive/v1-summary-notify-v0.0.5-2026-04-23` |

不要在 v1 分支上继续加 commit；本期从 main 重新拉：

```bash
cd team-pivot-web
git checkout main
git pull --ff-only origin main
git checkout -b feature/notify-mention-migration-v2
```

`vscode-team-pivot` **不建分支**（本期不改）。

---

## 11. 追踪表（开工后填）

### 11.1 分支状态

| # | 项 | 状态 |
|---|---|---|
| B1 | v1 分支归档改名 | [ ] |
| B2 | 从 main 拉 `feature/notify-mention-migration-v2` | [ ] |
| B3 | 服务端 pytest 全绿 | [ ] |
| B4 | Web `npm run build` 成功 | [ ] |
| B5 | 合并到 main + push | [ ] |

### 11.2 服务端任务

| Task | 状态 | commit / 时间 |
|---|---|---|
| S1 6 字段卡片 | [ ] | |
| S2 build_thread_card / build_reply_card | [ ] | |
| S3 standalone mention card | [ ] | |
| S4 mention DM card | [ ] | |
| S5 _post_url | [ ] | |
| S6 讨论目录折叠面板 | [ ] | |
| S7 FeishuNotifier workspace 参数 | [ ] | |
| S8 Notifier Protocol 签名 | [ ] | |
| S9 publish.py 适配 | [ ] | |
| S10 app.py 构造顺序 | [ ] | |

### 11.3 Web 任务

| Task | 状态 | commit / 时间 |
|---|---|---|
| W1 post 级深链 scrollIntoView | [ ] | |

### 11.4 验证

| # | 项 | 状态 |
|---|---|---|
| D1 uvicorn 本地启动 | [ ] |
| D2 pytest 全绿 | [ ] |
| D3 §6.2 A1-A7 全通过 | [ ] |
| D4 §6.3 R1-R4 全通过 | [ ] |

---

## 附录 A — 对比 v1 / v2 任务表

| v1 原 Task | v2 是否保留 | 说明 |
|---|---|---|
| S1 Pydantic summary 字段 | ❌ | 延期 |
| S2 route 传 summary | ❌ | 延期 |
| S3 publish.py 写 summary 到 frontmatter | ❌ | 延期 |
| S4 index_files.py 写 summary | ❌ | 延期 |
| S5 Notifier Protocol 扩展 + _post_url | ✅ 保留（重编号为 v2.S5 / v2.S8） | |
| S6 7 字段卡片 | 🟡 改为 6 字段（去摘要行） | |
| S7 standalone mention + summary 回读 | 🟡 保留结构化，不含 summary | |
| C1 AI prompt 改造 | ❌ | 延期 |
| C2 客户端本地 summary 校验 | ❌ | 延期 |
| C3 422/426 + 类型 | ❌ | 延期（422 没用例了；426 不新增） |
| C4 DraftCard 显示 summary | ❌ | 延期 |
| C5 版本号 0.0.5 + release.json | ❌ | 延期（本期不发 vsix） |
| W1 post 级深链 | ✅ 保留（v2.W1） | |
| W2 thread 详情页展示 summary | ❌ | 延期 |
| W3 422 错误 toast 自测 | ❌ | 延期 |

---

## 附录 B — 下一步（给接手同学）

1. 读完本文档
2. 按 §10 归档 v1 分支
3. 从 `main` 拉 `feature/notify-mention-migration-v2`
4. 按 §9 顺序执行 S1-S10 + W1
5. 每完成一个 Task 在 §11 追踪表填 commit SHA
6. 全部做完 → 跑 §6.1 pytest → 跑 §6.2 / §6.3 手测 → 向用户确认合并

---

**文档版本**：v2.0
**撰写时间**：2026-04-23
**基于需求**：`2026-04-23-notify-migration-v2-requirement.md`（范围收缩版需求）
