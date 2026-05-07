---
type: think
author: yezaiyong
created: '2026-04-23T18:29:47+08:00'
index_state: indexed
---
# 外部 AI 接入 Pivot：MCP 方案设计

**作者**：叶在勇
**日期**：2026-04-23
**状态**：draft，待评审
**依据**：Pivot 新需求《支持 MCP 接入外部 AI》、`pivot-product.md`、`pivot-interface.md`
**前置依赖**：新 Matter API 落地（见 `2026-04-23-index-refactor-design.md`）

---

## 一、为什么要做这件事

Pivot 正在从旧的 `thread / reply` 模型切到新的 `matter / timeline` 模型。老 API 和建立在它上面的客户端（例如 vscode-team-pivot）后续会失效或需要重构。

与此同时，公司同事越来越多地在 Claude Code、Codex、ChatGPT 这些外部 AI 终端里工作。如果我们只做新 Web 和 Pivot 内置 AI，**不定义一套统一的外部 AI 接入方式**，各家客户端很可能各接一套，形成新的碎片化。

一句话：

> 我们需要一条**不绑定某家 AI 产品、基于新 Matter API、正式对外开放**的外部 AI 接入通道。

---

## 二、第一版目标

让任何支持 MCP 的 AI 客户端（Claude Code、Codex、ChatGPT 等）都能接入 Pivot，跑完**最小闭环**：

```
用户在 Pivot Web 复制上下文 URL
        ↓
粘到外部 AI
        ↓
AI 读上下文 → 与用户讨论 → 起草新文件
        ↓
用户审稿 → 用户点头
        ↓
AI 提交，文件出现在 Pivot matter timeline 上
```

**明确不做**：创建新 matter、跨 matter 读取、搜索、评论流、草稿列表等。等第一版跑通再看。

---

## 三、技术选型：为什么是 MCP，而不是 skill 或 CLI

结论：**MCP 为主，CLI 为辅，skill 不作为主方案**。

| 方案 | 适合 | 不适合作为主线的原因 |
|------|------|--------------------|
| **MCP** | AI 终端 | — |
| skill | 某个客户端内部 | 跟客户端强耦合，跨产品难复用，升级成本高 |
| CLI | 调试 / 脚本 / fallback | 面向人类命令行用户，不直接服务 AI 终端 |

MCP 的核心优势：**工具名、描述、参数、返回结构一定义清楚，AI 在运行时就能直接理解并调用**，不用用户另装 skill 来"教"它。这正好符合"跨 AI 客户端通用"的目标。

CLI 作为辅助壳层保留（未来需要时再做），skill 作为极少数客户端的增强层保留（需要时再做）。

---

## 四、整体架构

```
┌──────────────────────────────────────────────────────┐
│   外部 AI 客户端                                      │
│   （Claude Code / Codex / ChatGPT / ...）             │
└──────────────┬───────────────────────────────────────┘
               │
               │  MCP over HTTP + SSE
               │  Authorization: Bearer pvt_xxxxxxxx
               ▼
┌──────────────────────────────────────────────────────┐
│   pivot.enclaws.ai  （FastAPI 服务）                   │
│                                                      │
│   ┌────────────────────┐      ┌──────────────────┐   │
│   │  /mcp endpoint     │ ───▶ │  Matter API      │   │
│   │  （本次新增）        │      │  （已有设计草案）  │   │
│   │                    │ ◀─── │  /api/matters    │   │
│   │  - PAT 认证转发     │      │  /api/matters/.. │   │
│   │  - URL 解析         │      │  /files          │   │
│   │  - 草稿状态存储      │      └──────────────────┘   │
│   │  - 字段校验         │                ↓            │
│   └────────────────────┘      ┌──────────────────┐   │
│                               │ Git + index YAML │   │
│                               └──────────────────┘   │
└──────────────────────────────────────────────────────┘
```

### 三个关键决定

1. **远程部署**：MCP 服务挂在 pivot.enclaws.ai 现有服务里，不起独立进程。用户只要配一个 URL + PAT 就能用，无需本地安装。
2. **现有 PAT 认证**：沿用 Pivot 已有的 PAT（`pvt_` 前缀、90 天过期、Bearer 头认证），不引入新机制。
3. **纯协议包装层**：MCP 自己不碰文件、不碰 index YAML，一切写入都走 Matter API。

---

## 五、MCP 对外提供的 5 个工具

### 读（3 个）

| 工具 | 作用 |
|------|------|
| `resolve_context` | 把用户粘的 Pivot URL 解析成 matter + 文件定位 |
| `list_matters` | 列出当前用户可见的 matter，可按状态、负责人、关键字筛选 |
| `get_matter` | 返回某个 matter 的头部 + 完整 timeline + 所有文件正文 |

### 写（2 个，强制两步走）

| 工具 | 作用 |
|------|------|
| `prepare_draft` | AI 把要写的内容和结构化字段一起交给服务端，**只校验、存草稿，不写入**。返回草稿编号 |
| `submit_draft` | 用户点头后，AI 拿草稿编号调此工具，真正写入 Pivot |

### 为什么写必须两步

AI 客户端（Claude Code / Codex 等）在 AI 每次调用工具时都会**弹确认框**让用户同意。两步 = 两个确认框：

- 调 `prepare_draft` 时：用户看到草稿全貌，决定"改改"还是"可以"
- 调 `submit_draft` 时：用户再点一次头，真正写入

这就是天然的"先审稿、再点头"gate。用户不需要装 skill、不需要记指令——工具本身的形状强制了流程。

---

## 六、核心流程

### 用户视角（8 步）

1. 在 Pivot Web 看某个 matter 的某篇文件
2. 点页面上的"复制上下文 URL"按钮
3. 粘给 AI："帮我看看这个"
4. AI 自动识别 URL、读 matter → 简要介绍给用户
5. 用户和 AI 多轮讨论
6. 用户说"写一篇 verify 吧"
7. AI 起草 → 用户审稿 → 确认
8. 文件出现在 Pivot matter 的 timeline 上

### 时序图

```
 用户         外部 AI        MCP 服务      Matter API
  │             │              │              │
  │  粘 URL ──▶ │              │              │
  │             │─ resolve ──▶ │              │
  │             │              │─ 校验存在 ──▶ │
  │             │◀── matter ── │◀── OK ────── │
  │             │    信息                      │
  │             │                             │
  │◀─ AI 回复  ─│                             │
  │             │                             │
  │  讨论… ──▶  │─ get_matter▶ │─ 拉时间线 ──▶ │
  │             │◀──timeline ──│◀── 全文 ──── │
  │             │                             │
  │◀─讨论… ────│                             │
  │             │                             │
  │ 帮我起草 ──▶│─ prepare ──▶ │(校验+存草稿)   │
  │             │◀─ draft_id ──│              │
  │             │                             │
  │ 看草稿 ◀────│                             │
  │             │                             │
  │ 点头 ─────▶ │─ submit ───▶ │─ POST files▶ │
  │             │              │              │─ 写入 Git
  │             │              │◀── 成功 ──── │
  │             │◀── 成功 ──── │              │
  │◀── 完成 ──── │              │              │
```

---

## 七、实施分工

这件事分成两条线，**本次我们只做其中一条**。

| 事项 | 谁做 | 状态 |
|------|------|------|
| 新 Matter API（`GET /api/matters`、`POST /api/matters/{id}/files` 等） | Index 重构线 | 设计已完成，待实现 |
| MCP 端点 `/mcp` + 5 个工具 | **本次** | 待设计确认、实现 |
| 草稿状态存储（SQLite 表 + 12 小时 TTL） | **本次** | 待实现 |
| Pivot Web "复制上下文 URL" 按钮 | **本次** | 待实现 |

**关键依赖**：MCP 依赖 Matter API 先落地才能端到端联调。**设计可以并行**（本文档），**实现要错峰**。

具体依赖的 Matter API 接口清单（全部已列在 `pivot-interface.md` 末尾的"Matter API（设计草案）"中）：

- `GET /api/matters`
- `GET /api/matters/{matter_id}`
- `POST /api/matters/{matter_id}/files`

---

## 八、访问控制

一句话：**MCP 不做任何自己的权限判断，全部透传给 Matter API**。

- 用户的 PAT 能读某个 matter → MCP 就能让 AI 读
- PAT 不行 → MCP 返回 401/403 给 AI，AI 告知用户

MCP 层只做"认证头转发"，不在 MCP 这一层重新实现权限模型。

---

## 九、草稿的生命周期

- 草稿存在服务端 SQLite 的一张表里（不用 Redis，避免额外依赖）
- TTL **12 小时**——覆盖"开会去、晚点回来接着改"的异步场景
- 到期自动清理
- `submit_draft` 之前草稿被删 / 过期 → 返回错误让 AI 重新起草

**为什么不让 AI 把全量内容在 submit 时再发一次（stateless 设计）**：

那样 AI 有机会在"用户审稿"和"用户点头"之间偷改内容，用户看到的 ≠ 最后写入的。存服务端 = 用户审到什么就是什么。

---

## 十、V1 的边界

| 在 V1 里 | 不在 V1 里 |
|----------|-----------|
| 读 matter 列表 / 详情 / 所有文件正文 | 跨 matter 读取（refer 到别的 matter 的文件） |
| 起草 think / act / verify / result / insight | 新建 matter（暂时先到 Web 建） |
| 两步走写入 | CLI 壳层 |
| PAT 认证 | 搜索、评论流、草稿列表 |
| 单 matter 作用域 | 编辑已提交的文件 |

第一版优先跑通最小闭环。上述"不在 V1"的能力等真实使用中有明确需求再补。

---

## 十一、风险

1. **Matter API 未落地，MCP 无 upstream 可调**
   - 对策：设计并行、实现错峰。本文档就是并行设计的产物。

2. **不同 AI 客户端对 MCP 的实现成熟度不一**
   - Claude Code 最佳，Codex / ChatGPT 待验证
   - 对策：V1 联调优先在 Claude Code 上跑通；其他客户端发现不兼容再逐家适配

3. **草稿生命周期内 matter 被别人改了，AI 的草稿基础数据过期**
   - 对策：submit 时服务端会校验 matter 状态；不一致返回 409，让 AI 重新 prepare

---

## 十二、成功标准

以下流程能在**任何支持 MCP 的 AI 客户端**里跑通，不依赖该客户端的任何插件 / skill：

1. 用户在 Pivot Web 复制 URL
2. 粘到 AI 客户端，AI 自动识别出是 Pivot 上的某 matter
3. 用户和 AI 讨论后让它起草一篇 think/act/verify/result/insight
4. 用户审稿后点头
5. 该文件出现在 Pivot matter timeline 上

若以 Claude Code 为首要验证对象，上述链路跑通 = V1 达标。

---

## 十三、待评审关注点

1. **部署形态**选远程 HTTP 而不是本地命令行 MCP server，符合"一次配置、全员可用"的原则。但离线 / 企业内网场景 V1 不覆盖——是否接受？
2. **草稿 TTL 12 小时**够不够？还是应该更长（例如 24h）或更短？
3. **跨 matter 读取 V1 不做**——对当前实际使用场景影响有多大？
4. **CLI 完全延后**——是否接受，还是至少要在 V1 里放一个最简 `pivot cat matter/X` 用于调试？
