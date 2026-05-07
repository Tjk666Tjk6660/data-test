---
type: think
author: xiongjianping
created: '2026-05-03T15:03:25+08:00'
index_state: indexed
---
# EC 底层改造方案 — 服务 SaaS-AI客服 + OPC 两类上层场景

> 日期：2026-05-03 ｜ 作者：xiongjianping ｜ 状态：v0.2，征询团队反馈

## 一、核心观点 + 边界

**一句话**：EC 是底座，CS / OPC / 其他 Agent 业务 = 上层场景；上层各自抽象出来的"通用能力"必须沉到底层共享，不允许重复造轮子；时间窗解耦——上层 S2 ≠ 底层交付窗口。

**三条原则**：
1. 同一能力同时服务 SaaS-AI客服 / OPC AI 员工 / OPC AI 客服 三场景，否则不算抽象到位
2. 上层 S2 阻塞底层时，走"接口预留 + 降级"，不停滞底层节奏
3. 单一事实源（readiness checks 等）拒绝多份定义

**明确不立项**：
- LLM Wiki 作为独立产品（按 ken 006 否定结论，等真实付费客户信号再评估）；其底层共性（知识库 CRUD / 版本历史 / 审核流程）作为 B 的 Knowledge tab 沉淀
- Notion 替代品（不在 EC 战场）

## 二、改造清单 + 抽象性自检

### 2.1 必做（A-G，已被多个上层依赖）

| 编号 | 改造项 | 用途 | 来源 |
|------|-------|------|------|
| **A** | **会话通用能力扩展（6 原语）**：多方会话 / 状态机 / 权限升级 / 消息元数据 / 全文搜索 / 消息生命周期 API | CS S2 + 工作台-会话 + OPC AI 员工共享会话底座 | `~/teamDocs/discussions/enclaws/enclaws-saas版的菜单及功能调整规划/003` Backlog #1 + `ai-customer-service-integration/020` §4 |
| **B** | **Agent 配置页架构重构**：Prompt 分类化（5 恒定 + 9 可选 + 3 业务块）+ 预设模板机制（defaults+locks）+ Knowledge tab 首次实装 | CS / OPC AI 员工 / 技能工坊共同依赖的 Agent 抽象 | `~/teamDocs/.../enclaws-saas.../003` Backlog #2 |
| **C** | **Embedding 分层配置**：Platform 标用途 + Tenant 优先顺序 + Agent override | 替代 auto-derive 临时方案；CS RAG / 未来语义搜索共享 | `~/teamDocs/.../enclaws-saas.../003` Backlog #3 |
| **D** | **LLM Provider 可用性巡检 + 模型健康快照**：调厂商账户 API（不消耗 token），快照入库 | CS gate / Agent 测试 / 套餐预检 统一 readiness 源 | `~/teamDocs/discussions/enclaws/llm-provider-可用性巡检与预警方案/001` |
| **E** | **通知/公告基础设施**：顶部横幅 + 铃铛 + 公告中心 + 通知频道一等公民化 | 公告 / 告警 / cron 失败通知 共享 UI | `~/teamDocs/.../enclaws-saas.../001` §5.7 + `llm-provider/001` §5 |
| **F** | **操作审计（双层）**：租户管理员行为 + 平台管理员行为 | SOC2 / ISO27001 / GDPR 合规底线 | `~/teamDocs/.../enclaws-saas.../001` §5.3 |
| **G** | **统一身份识别 + 跨会话权限边界 + 审计**（IM 跨频道按 user 归一） | OPC IM 集成 + SaaS 跨频道分析 共享 | `~/opc-web/docs/EC-IM与AI客服集成分析.md` §3 + §6 |

### 2.2 建议做（H-J，加分项不阻塞 S2）

| **H** | **数据管理 / 导出**（标准 markdown 一键导出） | GDPR 数据可携带权 |
| **I** | **双层 Token 上限（月 + 日）+ 日用量展示** | 防 runaway agent 烧穿月额度 |
| **J** | **标准 readiness primitive**（CS / Agent 测试 / 套餐预检统一调用） | D 的延伸，消除 ad-hoc 检查重复 |

### 2.3 抽象性自检表（A-J × 三场景）

| 改造项 | SaaS-AI客服 | OPC AI 员工 | OPC AI 客服 |
|--------|------------|-------------|------------|
| A 会话通用能力 | ✓ | ✓ | ✓ |
| B Agent 配置重构 | ✓ | ✓ | ✓ |
| C Embedding 分层 | ✓ | ✓ | ✓ |
| D 可用性巡检 | ✓ | ✓ | ✓ |
| E 通知/公告 | ✓ | ✓ | ✓ |
| F 操作审计 | ✓ | ✓ | ✓ |
| G 身份识别 | ✓ | ✓ | ✓ |
| H 数据导出 | ✓ | ✓ | ✓ |
| I 双层 Token | ✓ | ✓ | ✓ |
| J readiness primitive | ✓ | ✓ | ✓ |

**任一格 ✗ → 该项为单一场景定制，必须重新抽象。当前 Jason 自检全部 ✓，请 liuyu / 团队回填验证。**

## 三、上层 S2 依赖映射 + 启动条件

### 3.1 AI 客服 S2 依赖
- **阻塞型**：A 的"消息元数据 + 全文搜索 + 消息生命周期"三原语 + B + C
- **建议同步**：D + E + F + G

### 3.2 OPC S2 依赖
- **阻塞型**：G（IM 集成）+ A（接口预留即可，实现可滞后）+ D（OPC 独立部署强依赖 readiness 降级）
- **建议同步**：B + C + E + F
- **OPC 独有约束**：所有 EC 能力必须 API 化 + 显式降级语义（独立部署 + 跨网络调用；EC 不可用时 OPC 业务必须可手工推进）

### 3.3 启动条件矩阵

| 类型 | 项 | 触发 |
|------|-----|------|
| S2 内必交 | A 三原语 / B / C | CS S2 启动 |
| 接口预留 | A 其余三原语 | OPC AI 员工接入 |
| 按需启动 | E / F / G / H | 客户合规需求或 OPC 试点 |
| 事件触发 | I | runaway agent 出现 |
| 长期立项 | A 多方会话 / 状态机 / 权限升级 | S3 Operator 启动 |

## 四、给 liuyu 的需求表（核心交付物）

每条改造请 liuyu 填后两列：

| 编号 | 需求 | 上层用途 | 期望交付时间窗 | 若延期上层降级方案 |
|------|------|---------|--------------|------------------|
| A.1 | 消息元数据接口 | CS 置信度/反馈/标签 | _____ | CS 退回固定模板回复 |
| A.2 | 全文搜索 | CS 历史会话 + 工作台-会话 | _____ | 模糊匹配兜底 |
| A.3 | 消息生命周期 API | CS 撤回/删除 | _____ | 仅标"已删除"不真删 |
| A.4 | 多方会话 | S3 Operator | _____ | 接口预留 |
| A.5 | 会话状态机 | S3 Operator | _____ | 接口预留 |
| A.6 | 权限升级 | S3 Operator | _____ | 接口预留 |
| B | Agent 配置页重构 | 全场景 | _____ | 当前架构继续打补丁 |
| C | Embedding 分层配置 | CS RAG | _____ | 沿用 auto-derive 临时方案 |
| D | 可用性巡检 + 健康快照 | CS gate / OPC readiness | _____ | inline 配置存在性检查 |
| E | 通知/公告 + 频道一等公民 | 全场景 | _____ | 不做 UI 通知，仅日志 |
| F | 操作审计（双层） | 合规 | _____ | 后端有数据无 UI |
| G | 身份识别 + 跨会话权限 + 审计 | OPC IM 集成 | _____ | OPC 暂不接 IM |
| H | 数据管理 / 导出 | GDPR | _____ | 手工导出 SQL |
| I | 双层 Token 上限 | 防失控 | _____ | 沿用月上限 |
| J | readiness primitive | 全场景 | _____ | ad-hoc 检查重复 |

**这一章就是 liuyu 的回复界面**——读完只需在最后两列填字。其他成员也欢迎对每条项补充判断。

## 五、高价值内容迁移说明

新 matter 同时承担多个 teamDocs 话题的统一推进入口：

| 来源 | 迁移内容 | 在本方案中的落位 |
|------|---------|----------------|
| `~/teamDocs/discussions/enclaws/enclaws-saas版的菜单及功能调整规划/`（001 + 003） | 菜单设计三原则 + Phase 0/+1 行动方案 + Backlog 启动条件 | §1 三原则 + §2 改造清单 A-F + §3 启动条件 |
| `~/teamDocs/discussions/enclaws/llm-provider-可用性巡检与预警方案/`（001） | 健康巡检设计 + 公告告警基础设施 | §2 D + E |
| `~/teamDocs/discussions/enclaws/llm-wiki/`（001+003+005 + ken 006 否定） | Ken 否定结论 + 知识库 CRUD/版本/审核 共性 | §1 边界（不立项）+ §2 B Knowledge tab |
| `~/opc-web/docs/EC-IM与AI客服集成分析.md` | OPC IM 集成 + 独立部署 + 跨会话权限 | §2 G + §3.2 OPC 独有约束 |
| `~/teamDocs/discussions/enclaws/ai-customer-service-integration/`（016-020 共识） | S2 6 原语 / Agent 重构 / Embedding 分层 | §2 A + B + C 直接来源 |

**演进约定**：以上话题的后续讨论收敛到本 matter；teamDocs 原帖保留作为历史。

---

**征询反馈**：
1. **必做 A-G 是否完整？** 抛砖引玉，期待团队补充未识别的依赖项
2. **liuyu** 请优先回复 §四 需求表（最后两列）——交付时间窗 + 延期降级方案
3. **dengke / daisy / 其他** 欢迎对 §1 边界、§2 不立项判断、§5 迁移范围 提意见
