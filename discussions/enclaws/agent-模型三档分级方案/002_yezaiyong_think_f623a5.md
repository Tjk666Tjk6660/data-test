---
type: think
author: yezaiyong
created: '2026-04-21T14:57:05+08:00'
index_state: indexed
---

# EnClaws 模型 & Agent 管理 UI 改造方案

> 文档版本：v4.0（改造稿）
> 创建日期：2026-04-21
> 上游：v3 设计稿 `004_yezaiyong-liuyu_reply.md` + HTML 原型 `enclaws-model-agent-ui-prototype.html`
> 定位：v3 跟原型对齐后的最终落地方案，UI 完整 + DB 最小改动

---

## 一、为什么有这份文档

v3 方案（004）已经把"三档分级 + 档内 failover"的落地路径讲清楚了，但 v3 把档位挂在**每个 Agent 身上**（每个 Agent 独立持有 `tiers.pro/standard/lite` 数组 + `defaultTier` + `failoverEnabled`）。这在"不同 Agent 想用不同 PRO 模型"的场景下灵活，但代价是：

- Agent 管理页要塞一个完整的"三段分区 + 拖拽排序 + 单选默认 + 添加模型 Modal"配置区，UI 很重
- 数据结构要把现有 `ModelConfigEntry[]` 扁平数组改成 `TieredModelConfig` 对象（破坏性变更，需要迁移脚本 + sqlite 镜像同步）
- 新增 `sys_config` 表项 + `GET /v1/model-tier-recommendations` API + 前端 fetcher + migration，支撑"推荐档位徽章"

在实际做原型时，我们发现**多租户下几乎不会出现"同一租户里 Agent A 的 PRO 用 Opus、Agent B 的 PRO 用 GPT-5"** 这种需求——一个租户就一套账号、一套预算、一套模型偏好。**把档位归属从 Agent 上提到租户级**，UI 能简化一大截，数据结构也能不动。

HTML 原型（`enclaws-model-agent-ui-prototype.html`）走的就是这条路：

- 模型管理页独立成 tab，模型本身自带 `tier`，按档分组展示
- Agent 只需勾选"启用的档位"，档内首选/备用由"模型管理"页的 `isDefault` 和数组顺序决定
- 不同 Agent 共享同一套"tier → 模型"映射（以租户为单位）

这份 v4 方案把原型的模式正式落到现有代码结构上。

---

## 二、核心改动一览

### 2.1 从 v3 到 v4 的转变

| 维度 | v3 | v4（本方案） |
|---|---|---|
| 档位挂载位置 | 每个 Agent 独立持有 `tiers` 对象 | 模型自带 `tier` 字段（租户级共享） |
| Agent 侧存什么 | `tiers.pro/standard/lite: ModelRef[]` + `defaultTier` + `failoverEnabled` | **保持现状**：`modelConfig: ModelConfigEntry[]`；`isDefault` 语义重解释为"tier 内默认" |
| 同一租户跨 Agent 差异化 | 支持（每个 Agent 独立配） | 不支持（所有 Agent 看同一套 tier→model 映射） |
| 模型管理页作用 | Provider 容器 + 挂载模型 | Provider 容器 + 挂载模型（**每个模型带 tier**） |
| 添加模型 Modal | 双区 accordion（选已有 Provider / Quick Add） | **联动单表单**（档位 → 供应商 → 自动预填） |
| 推荐档位数据源 | `sys_config` 表 + API + 前端 fetcher | **前端常量**（`MODEL_SUGGESTIONS[provider][tier]`） |
| DB schema 改动 | `tenant_agents.model_config` JSONB shape 重构 + 迁移 + sqlite 镜像 | **零表改动**，仅 JSONB 内新增可选字段 `tier` |

### 2.2 v3 保留的东西

- "调用方传 tier 参数 / 系统不猜" 的接口契约（`agent-chat-http.ts` `MODEL_TIER_VALUES` 已占位）
- 档内 failover（技术失败切下一个）+ 跨档不降级（全挂返 `TIER_EXHAUSTED`）
- 命名 `lite / standard / pro`（跟源码已存在的 `MODEL_TIER_VALUES` 对齐）
- 失败判据清单（5xx / 429 / 网络 = 切；4xx / content policy / abort = 不切）

### 2.3 v4 砍掉的东西（相对 v3）

| 砍的项 | v3 所在章节 | 理由 |
|---|---|---|
| `TieredModelConfig` 这个类型（`tiers.pro/standard/lite`） | §4.4 Schema 改造 | `modelConfig: ModelConfigEntry[]` 扁平数组够用 |
| `defaultTier` / `failoverEnabled` 存 DB | §4.4 | `failoverEnabled` 变成全局默认 true（租户级配置可后补）；`defaultTier` 概念消失（每档各自一个 `isDefault`） |
| `sys_config` 表的 `model-tier-recommendations` 行 | §3.1 | 前端常量够用，新模型发布时改常量发版即可 |
| `GET /v1/model-tier-recommendations` API | §3.5 | 同上 |
| `seedModelTierRecommendations` migration + sqlite 对应 | §3.3 | 同上 |
| `ui/src/ui/utils/recommend-tier.ts` fetcher + 缓存 | §3.6 | 前端直接读常量，不需要异步 |
| `TenantModelDefinition.recommendedTier` 字段 | §4.4.3 | 跟本方案的 `tier` 字段合并（就是同一个东西） |
| Agent 管理页"三段分区 + 拖拽 + 添加模型 Modal" | §4.1 §4.2 | Agent 只需勾启用档位 |
| `defaultTier` 单选控件 | §4.1 | 默认档由"调用方不传 tier"走的档决定——这在 v4 里由调用方选，UI 不需要 |
| 不匹配推荐的 soft confirm 弹窗 | §4.2.1 | 模型自带 tier，"不匹配"这个概念不存在了 |

---

## 三、数据结构改动（唯一的）

### 3.1 精确位置

```
┌─ tenant_models 表（不动 SQL schema） ─────────────────────┐
│ id, tenant_id, provider_type, provider_name, base_url,... │
│ models  JSONB  DEFAULT '[]'   ← 在这个 JSONB 数组的元素内 │
└───────────────────────────────────────────────────────────┘
                   │
                   ▼  每个 TenantModelDefinition 对象
                 {
                   "id": "claude-opus-4-7",
                   "name": "Claude Opus 4.7",
                   "reasoning": true,
                   "contextWindow": 200000,
                   "tier": "pro"        ← v4 新增的这一 key
                 }
```

`tenant_agents` 表一个字段都不动。

### 3.2 TypeScript 类型改动

**文件**：`src/db/types.ts`

```typescript
// 新增一个类型别名（三个值跟 src/gateway/agent-chat-http.ts:15 MODEL_TIER_VALUES 对齐）
export type ModelTier = "lite" | "standard" | "pro";

// TenantModelDefinition 增加一个可选字段
export interface TenantModelDefinition {
  id: string;
  name: string;
  reasoning?: boolean;
  input?: Array<"text" | "image">;
  contextWindow?: number;
  maxTokens?: number;
  cost?: { input: number; output: number; cacheRead: number; cacheWrite: number };
  alias?: string;
  streaming?: boolean;
  compat?: Record<string, unknown>;
  tier?: ModelTier;   // ← v4 新增，旧数据 undefined
}
```

### 3.3 `ModelConfigEntry` 的语义变更（字段不改）

```typescript
export interface ModelConfigEntry {
  providerId: string;  // tenant_models.id
  modelId: string;     // tenant_models.models[].id
  isDefault: boolean;  // v4 前: agent 全局默认
                       // v4 后: 在其引用模型的 tier 内默认
                       //        (同一 agent 的 modelConfig 里，每个 tier 最多一个 isDefault=true)
}
```

**应用层约束**：保存 agent 时，按 tier 分组，每组确保最多一个 `isDefault=true`。

### 3.4 零迁移策略

- 旧模型条目 `tier` 就是 `undefined`
- 模型管理页渲染时，`tier=undefined` 的模型归入顶部的**"未分档"**块，旁边一个 `[分到 XX 档]` 快捷按钮
- admin 可以通过卡片上的 `[编辑]` 按钮打开完整编辑 Modal 补 tier（也可改显示名、秘钥、baseUrl 等任何字段）
- 旧 agent 的 `modelConfig` 在新语义下不需要改：原来 `isDefault=true` 的那个模型，其 tier 就是它所在档的默认；其他 `isDefault=false` 的是该档的备用
- 如果旧 agent 里有一条引用的模型 tier 未定，前端 Agent 管理页把这条单列显示为"该模型未分档，先去模型管理页配 tier"

### 3.5 为什么不把 tier 放到 `ModelConfigEntry` 上？

因为模型管理页要**独立于 Agent** 渲染档位分组。如果 tier 挂在 `ModelConfigEntry`，模型管理页就没有 tier 信息（那是 agent 侧的）。放到 `TenantModelDefinition` 上，模型管理页读 `tenant_models.models[*].tier` 就直接能分组渲染。

---

## 四、页面改造

### 4.1 模型管理页（`ui/src/ui/views/tenant/tenant-models.ts`）

#### 4.1.1 整体布局

```
┌─ 模型管理 ───────────────────────────────────────────────┐
│                                                           │
│ 已配置 4 个模型                          [+ 添加模型]    │
│ ──────────────────────────────────────────────────────── │
│                                                           │
│ [PRO 专家] 1 个                                          │
│ ┌──────────────────────────────────────────────────────┐│
│ │ claude-opus-4-7                         [⭐ 默认]     ││
│ │ Anthropic · 我的 Anthropic 账号                       ││
│ │ https://api.anthropic.com/v1 · anthropic-messages   ││
│ │                             [编辑] [设为默认] [删除]  ││
│ └──────────────────────────────────────────────────────┘│
│                                                           │
│ [STANDARD 标准] 2 个                                     │
│ ┌──────────────────────────────────────────────────────┐│
│ │ claude-sonnet-4-6                       [⭐ 默认]     ││
│ │ Anthropic · 我的 Anthropic 账号                       ││
│ │                             [编辑] [设为默认] [删除]  ││
│ └──────────────────────────────────────────────────────┘│
│ ┌──────────────────────────────────────────────────────┐│
│ │ qwen-plus                                             ││
│ │ 阿里 Qwen · Qwen 备用                                 ││
│ │                             [编辑] [设为默认] [删除]  ││
│ └──────────────────────────────────────────────────────┘│
│                                                           │
│ [LITE 快速] 1 个                                         │
│ ┌──────────────────────────────────────────────────────┐│
│ │ claude-haiku-4-5                        [⭐ 默认]     ││
│ └──────────────────────────────────────────────────────┘│
│                                                           │
│ [未分档] 0 个（或迁移期有旧数据时显示）                  │
└───────────────────────────────────────────────────────────┘
```

#### 4.1.2 交互规则

- 三档固定显示顺序 PRO → STANDARD → LITE
- 空档不显示分组标题（原型行为：`.filter(t => groups[t].length > 0)`）
- 每个模型卡显示 `[⭐ 默认]` 徽章时，表示它是该档内第一个被所有 Agent 默认使用的模型
- `[设为默认]` 按钮点击后，同档其他模型的 `isDefault` 全部置 false，当前这个置 true
- `[编辑]` 打开跟"添加模型"**同一个 Modal**（模式切换为 edit），把现值预填进去，可改：档位、显示名、baseUrl、协议、认证方式、秘钥、modelId
    - API 秘钥字段在编辑模式显示掩码 `•••• abc1` + placeholder"留空=保持不变"——支持秘钥轮换不打扰其他字段
    - 档位改动会影响所有引用该模型的 Agent（弹 soft confirm："该模型被 X 个 Agent 使用，改档位会影响这些 Agent 的启用档位列表，继续？"）
- 删除模型时：
    - 警告"该模型正在被 X 个 Agent 使用"
    - 二次确认后删除，同时级联从各 Agent 的 `modelConfig` 里清掉对应条目

#### 4.1.3 添加 / 编辑模型 Modal（联动单表单，双模式共用）

Modal 同时承载"添加"和"编辑"两种模式，走同一个组件，差异只在初始值和提交路径。

**添加模式**（点页面右上角 `[+ 添加模型]` 打开）：所有字段空；联动规则生效（选档位 → 刷供应商 → 预填 baseUrl / 协议 / modelId）；保存走 create。

**编辑模式**（点卡片 `[编辑]` 按钮打开）：所有字段预填当前值；联动规则仍生效（用户改档位/供应商时仍自动预填新值）；API 秘钥字段特殊——显示掩码 `•••• abc1` + placeholder"留空=保持不变"，提交时秘钥为空字符串表示不更新；保存走 update。

```
┌─ 添加模型 / 编辑模型 ──────────────────────────────┐
│ 选档位和供应商后，下方字段会自动预填               │
│                                                    │
│ ① 模型档位                                         │
│ ┌─────────┬──────────┬─────────┐                  │
│ │ PRO     │ STANDARD │ LITE    │ (单选 pill)       │
│ │ 专家    │ 标准     │ 快速    │                   │
│ └─────────┴──────────┴─────────┘                  │
│                                                    │
│ ② 供应商                                           │
│ [ 请选择供应商 ▾ ]                                 │
│ 提示：STANDARD 档可选 6 家供应商                   │
│                                                    │
│ ③ Provider 显示名                                  │
│ [ 我的 Anthropic 账号_____________ ]               │
│                                                    │
│ API 地址                                           │
│ [ https://api.anthropic.com/v1____ ] (联动预填)    │
│                                                    │
│ API 协议          认证方式                         │
│ [anthropic- ▾]    [api-key ▾]                      │
│                                                    │
│ API 秘钥                                           │
│ 添加模式: [ sk-ant-__________________ ]            │
│ 编辑模式: [ •••• abc1                    ]         │
│           (留空=保持不变；有值=更新)               │
│                                                    │
│ 模型 ID                                            │
│ [ claude-sonnet-4-6__________ ] (联动预填推荐值)   │
│                                                    │
│ ☑ ⭐ 设为此档位的默认模型                          │
│                                                    │
│                             [取消] [保存]          │
└────────────────────────────────────────────────────┘
```

**联动逻辑**（跟原型 HTML 的 `onTierChange` / `onProviderChange` 一致）：

| 触发 | 动作 |
|---|---|
| 选档位 | 刷新供应商下拉（`PROVIDERS_BY_TIER[tier]`）；清空 modelId / baseUrl |
| 选供应商 | 预填 baseUrl + 协议（`PROVIDER_DEFAULTS[provider]`）；预填推荐 modelId（`MODEL_SUGGESTIONS[provider][tier]`）；Provider 显示名默认值"我的 XX 账号"（用户可改） |
| 勾设为默认 | 保存时清掉同档其他模型的 `isDefault` |

**推荐 modelId 常量表**（前端 hardcode，发版可改）：

```typescript
// ui/src/ui/utils/model-suggestions.ts（v4 新建）
export const PROVIDER_DEFAULTS: Record<string, { baseUrl: string; protocol: string }> = {
  anthropic: { baseUrl: "https://api.anthropic.com/v1", protocol: "anthropic-messages" },
  openai:    { baseUrl: "https://api.openai.com/v1",    protocol: "openai-completions" },
  qwen:      { baseUrl: "https://dashscope.aliyuncs.com/compatible-mode/v1", protocol: "openai-completions" },
  deepseek:  { baseUrl: "https://api.deepseek.com/v1",  protocol: "openai-completions" },
  glm:       { baseUrl: "https://open.bigmodel.cn/api/paas/v4", protocol: "openai-completions" },
  custom:    { baseUrl: "",                             protocol: "openai-completions" },
};

export const MODEL_SUGGESTIONS: Record<string, Record<ModelTier, string>> = {
  anthropic: { lite: "claude-haiku-4-5", standard: "claude-sonnet-4-6", pro: "claude-opus-4-7" },
  openai:    { lite: "gpt-4o-mini",      standard: "gpt-5",             pro: "gpt-5-high" },
  qwen:      { lite: "qwen-turbo",       standard: "qwen-plus",         pro: "qwen-max" },
  deepseek:  { lite: "",                 standard: "deepseek-v3",       pro: "deepseek-r1" },
  glm:       { lite: "glm-4-flash",      standard: "glm-4",             pro: "glm-5" },
  custom:    { lite: "",                 standard: "",                  pro: "" },
};

export const PROVIDERS_BY_TIER: Record<ModelTier, string[]> = {
  pro:      ["anthropic", "openai", "qwen", "deepseek", "glm", "custom"],
  standard: ["anthropic", "openai", "qwen", "deepseek", "glm", "custom"],
  lite:     ["anthropic", "openai", "qwen", "glm", "custom"],
};
```

#### 4.1.4 保存路径

**添加模式**点"保存"时：

1. 如果 Provider 显示名 + baseUrl 的组合命中已有 `TenantModel`（同租户内按"显示名+baseUrl"查重），往那个记录的 `models` 数组追加一条 `TenantModelDefinition{ id, name, tier, ... }`
2. 否则调 `tenant.models.create` 新建一个 `TenantModel`，把该条模型作为它的唯一挂载

这个路径等于把原型 HTML 里的扁平 `models` 数组自动收敛回 EnClaws 的两层结构（Provider 容器 + 挂载模型）。用户不感知。

**编辑模式**点"保存"时：

1. Modal 打开时记录原始 `{providerId, modelDefinitionId}` 作为 handle
2. 若**仅改模型层字段**（tier / 显示名 / modelId）：调 `tenant.models.update`（只改该 `TenantModel.models[*]` 里对应那条）
3. 若**改了 Provider 层字段**（显示名 / baseUrl / 协议 / 认证方式 / 秘钥）：更新 `TenantModel` 本体；秘钥字段为空则保留原值，非空则走加密存储
4. 若 admin 同时改了档位：保存前弹 soft confirm"该模型被 X 个 Agent 使用，改档位会影响这些 Agent"，确认后再写
5. 任何字段改动不级联修改 `TenantAgent.modelConfig`（entry 存的只是 `providerId + modelId` 引用，模型本体字段变了引用不失效）

### 4.2 Agent 管理页（`ui/src/ui/views/tenant/tenant-agents.ts`）

#### 4.2.1 整体布局

```
┌─ Agent 管理 ─────────────────────────────────────────┐
│                                                        │
│ 已配置 2 个 Agent                    [+ 新建 Agent]   │
│ ────────────────────────────────────────────────── │
│                                                        │
│ ┌─────────────────────────────────────────────────┐  │
│ │ 客服机器人                                       │  │
│ │ 启用档位:                                        │  │
│ │  [PRO] → ⭐ claude-opus-4-7                      │  │
│ │  [STANDARD] → ⭐ claude-sonnet-4-6               │  │
│ │               · qwen-plus（备用）                │  │
│ │  [LITE] → ⭐ claude-haiku-4-5                    │  │
│ │                          [编辑]  [删除]          │  │
│ └─────────────────────────────────────────────────┘  │
│                                                        │
│ ┌─────────────────────────────────────────────────┐  │
│ │ 内部助手                                         │  │
│ │ 启用档位:                                        │  │
│ │  [STANDARD] → ⭐ claude-sonnet-4-6               │  │
│ │                          [编辑]  [删除]          │  │
│ └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

#### 4.2.2 编辑 / 新建 Agent Modal

```
┌─ 新建 Agent ──────────────────────────────────────────┐
│                                                        │
│ Agent 名称                                             │
│ [ 客服机器人_________________________ ]                │
│                                                        │
│ 启用的档位（可多选）                                   │
│ ┌───────────────────────────────────────────────────┐│
│ │ ☑ [PRO 专家]  1 个可用                            ││
│ │   ⭐ 默认: claude-opus-4-7                        ││
│ │   备用: (无)                                       ││
│ │   ⚠️ 该档无备用模型，默认挂了就会返回错误           ││
│ └───────────────────────────────────────────────────┘│
│ ┌───────────────────────────────────────────────────┐│
│ │ ☑ [STANDARD 标准]  2 个可用                       ││
│ │   ⭐ 默认: claude-sonnet-4-6                      ││
│ │   备用: qwen-plus                                  ││
│ └───────────────────────────────────────────────────┘│
│ ┌───────────────────────────────────────────────────┐│
│ │ ☐ [LITE 快速]  1 个可用                           ││
│ │   ⭐ 默认: claude-haiku-4-5                       ││
│ └───────────────────────────────────────────────────┘│
│                                                        │
│ 💡 档位选完后，系统怎么选模型：                        │
│ · 默认走该档 ⭐ 默认模型                               │
│ · 默认模型故障时自动切到同档其他模型（档内 failover） │
│ · 档内全挂时返回错误给调用方（不跨档降级）            │
│                                                        │
│                        [取消]  [保存 Agent]           │
└────────────────────────────────────────────────────────┘
```

#### 4.2.3 保存路径（从"勾选档位"到 `ModelConfigEntry[]`）

用户勾了 `enabledTiers = ["pro", "standard"]`，保存时：

```typescript
// 读取当前租户所有模型，按 tier 分组
const modelsByTier = groupModelsByTier(tenantModels);
// { pro: [{providerId, modelId, isDefault}, ...],
//   standard: [...], lite: [...] }

// 投影出该 agent 的 modelConfig
const modelConfig: ModelConfigEntry[] = [];
for (const tier of enabledTiers) {
  // 按"原租户模型定义里 isDefault=true 的排第一，其余按原顺序"
  modelConfig.push(...modelsByTier[tier]);
}

await updateAgent({ ..., modelConfig });
```

**注意**：`TenantModelDefinition.tier` 是模型侧的 tier，`ModelConfigEntry` 只是对这组"已选"模型的引用。`isDefault` 在 entry 层继承自模型层当前的默认状态。

#### 4.2.4 空态与禁用态

- 该租户**一个模型都没配**时：整个档位勾选区替换成"⚠️ 还没有配置任何模型。请先去「模型管理」页面添加模型后再来。" + `[跳转到模型管理]` 按钮
- 某档**没有任何模型**时：该档 checkbox 置灰禁用，文案"该档无可用模型，先去模型管理页添加"
- 所有档 checkbox 全空时：`[保存 Agent]` 置灰，提示"至少启用一档"

### 4.3 引导页（`ui/src/ui/views/onboarding-wizard.ts`）

#### 4.3.1 Step 2 配置模型 —— 双模式

跟原型一致，保留 `shared` / `custom` 两种入口：

```
◉ 使用共享模型（推荐新手）
  平台管理员预配好的，无需自己填 API key

○ 自己配 API Key
  你有 Anthropic / OpenAI / Qwen / DeepSeek 等账号
```

#### 4.3.2 shared 模式

```
┌─ 选择一个共享模型 ─────────────────────────────┐
│ [ Claude Sonnet 4.6 (shared) ▾ ]              │
│                                                │
│ 推荐档位: [STANDARD]                           │
└────────────────────────────────────────────────┘
```

- 下拉列表从共享模型池读（租户级可见的 shared `TenantModel`）
- 展示档位徽章：直接读模型自带的 `tier`（共享模型必须预置 tier）
- 一键完成：选中后直接把该模型塞进**新租户自动创建的第一个 agent** 的 `modelConfig`

#### 4.3.3 custom 模式 —— 按档逐一添加

跟原型 HTML 设计一致，核心是**进度徽章 + 联动单表单 + 加完一档继续加**：

```
┌─ 已配置进度 ────────────────────────────────┐
│ [PRO 未配置] [STANDARD ✓ qwen-plus] [LITE 未配置] │
└──────────────────────────────────────────────┘

┌─ ① 模型档位 ──────────────────────────────┐
│ [PRO] [STANDARD ✓] [LITE]  (单选 pill)    │
│ 新租户第一个模型推荐选 STANDARD 档         │
└────────────────────────────────────────────┘

┌─ ② 供应商 ────────────────────────────────┐
│ [ 请选择供应商 ▾ ]                         │
└────────────────────────────────────────────┘

┌─ ③ 配置 ──────────────────────────────────┐
│ (Provider 显示名 / API 地址 / 协议 /       │
│  认证方式 / 秘钥 / 模型 ID 字段)           │
│ ☑ ⭐ 设为此档位的默认模型（首个默认勾上）  │
│                                            │
│                    [✓ 添加并继续 →]        │
└────────────────────────────────────────────┘

(添加成功后)
✅ 模型配置成功！claude-sonnet-4-6 已添加到 STANDARD 档
你已配置 1 档，Agent 启动时会使用所有已配档位。
[+ 继续添加另一档]    [下一步 (频道) →]
```

**关键交互**：
- 已配档位的 pill 会灰禁 + 打勾（`.disabled::after content: "✓"`），避免重复添加
- 三档全配完后"继续添加"按钮自动隐藏，只剩"下一步"
- 引导页结束时把 `obConfiguredTiers` 直接投影成新 agent 的 `modelConfig`（同 §4.2.3 的逻辑）

#### 4.3.4 引导页最后产生的数据

走完"配置模型"这一步时，后端实际发生的事：

1. 对 custom 模式的每一档：
    - 查重：按"Provider 显示名 + baseUrl"命中已有 `TenantModel` 就追加 `models[]` 一条
    - 没命中就新建 `TenantModel`，挂一条 `models[0]`
    - 新 `TenantModelDefinition` 填 `tier: <用户当前选的档>`
2. 新建 `TenantAgent`（引导页产物），`modelConfig` 投影自所有已配档位

### 4.4 三页共用：i18n key

新增到 `ui/src/i18n/locales/zh-CN.ts`（en / de 可后续补）：

```typescript
tenantModels: {
  "tier.pro": "PRO 专家",
  "tier.standard": "STANDARD 标准",
  "tier.lite": "LITE 快速",
  "tier.unassigned": "未分档",
  "addModal.title": "添加模型",
  "addModal.subtitle": "选档位和供应商后，下方字段会自动预填",
  "addModal.fieldTier": "① 模型档位",
  "addModal.fieldProvider": "② 供应商",
  "addModal.fieldProviderHint": "{tier} 档可选 {count} 家供应商",
  "addModal.fieldProviderPlaceholder": "请选择供应商",
  "addModal.fieldName": "③ Provider 显示名",
  "addModal.fieldBaseUrl": "API 地址",
  "addModal.fieldProtocol": "API 协议",
  "addModal.fieldAuth": "认证方式",
  "addModal.fieldApiKey": "API 秘钥",
  "addModal.fieldModelId": "模型 ID",
  "addModal.fieldModelIdPlaceholder": "选完档位和供应商会自动填推荐值",
  "addModal.setDefault": "⭐ 设为此档位的默认模型",
  "editModal.title": "编辑模型",
  "editModal.subtitle": "改完点保存；API 秘钥留空=保持不变",
  "editModal.apiKeyPlaceholder": "留空=保持不变",
  "editModal.apiKeyMaskedHint": "当前秘钥：{masked}",
  "editModal.tierChangeConfirm":
    "该模型被 {agentCount} 个 Agent 使用，改档位会影响这些 Agent 的启用档位列表。继续？",
  "list.edit": "编辑",
  "list.setDefault": "设为默认",
  "list.countHeader": "已配置 {count} 个模型",
  "list.emptyTitle": "还没添加任何模型",
  "list.emptyHint": "点击右上角 + 添加模型 开始",
  "deleteConfirm.inUse": "该模型正在被 {agentCount} 个 Agent 使用，确认删除？",
  "validate.requiredFields": "请填完必填字段（显示名 / API 地址 / 模型 ID）",
  "validate.authNeedsKey": "该认证方式需要 API 秘钥",
},
tenantAgents: {
  "tier.enabledLabel": "启用的档位（可多选）",
  "tier.availableCount": "{count} 个可用",
  "tier.defaultModel": "⭐ 默认",
  "tier.backupModels": "备用",
  "tier.noBackup": "⚠️ 该档无备用模型，默认挂了就会返回错误",
  "tier.empty": "还没有配置任何模型。请先去「模型管理」页面添加模型后再来。",
  "tier.gotoModels": "跳转到模型管理",
  "tier.helpBox":
    "档位选完后，系统怎么选模型：\n· 默认走该档 ⭐ 默认模型\n· 默认模型故障时自动切到同档其他模型（档内 failover）\n· 档内全挂时返回错误给调用方（不跨档降级）",
  "validate.atLeastOneTier": "至少启用一档",
},
onboarding: {
  "model.stepTitle": "配置你的第一个 AI 模型",
  "model.stepSubtitle": "AI 助手需要至少一个大模型才能工作",
  "model.modeShared": "✨ 使用共享模型（推荐新手）",
  "model.modeCustom": "🔑 自己配 API Key",
  "model.progressTitle": "已配置进度",
  "model.tierHintFirst": "新租户第一个模型推荐选 STANDARD 档",
  "model.addAndContinue": "✓ 添加并继续",
  "model.continueAnother": "+ 继续添加另一档",
  "model.nextStep": "下一步 (频道) →",
},
```

---

## 五、运行时（接口层）

跟 v3 §五一致，v4 不改动：

1. `POST /v1/agent/chat/completions` 带 `model_tier?: "lite"|"standard"|"pro"`
2. 没传 tier → 走 agent 的"默认 tier"（这里的"默认 tier"需要一个语义映射：因为 v4 取消了 `defaultTier` 字段，改为**用 `modelConfig` 里第一条 `isDefault=true` 的模型所在 tier**作为默认）
3. 传了 tier → 过滤 `modelConfig` 里引用模型 tier == 请求 tier 的那些条目，组成 failover 链
4. 档内全挂 → `TIER_EXHAUSTED`；传了 tier 但该档空 → `TIER_NOT_CONFIGURED`

**关键实现点**：

`src/gateway/agent-chat-http.ts` 要做的改动（当前 `MODEL_TIER_VALUES` 已占位）：

```typescript
// --- 5. Validate model_tier (unchanged) ---
const modelTier: ModelTier | undefined = validatedModelTier;

// --- 7. Resolve tier chain (新逻辑) ---
const entriesWithModel = await Promise.all(
  agent.modelConfig.map(async (e) => ({
    entry: e,
    model: await lookupTenantModel(tenantId, e.providerId, e.modelId),
  })),
);

function resolveTierChain(tier: ModelTier | undefined): ModelConfigEntry[] {
  if (tier === undefined) {
    // 走默认：第一条 isDefault=true 的 entry 所在 tier
    const def = entriesWithModel.find(x => x.entry.isDefault);
    if (!def?.model?.tier) throw new Error("NO_DEFAULT_TIER");
    tier = def.model.tier;
  }
  const chain = entriesWithModel
    .filter(x => x.model?.tier === tier)
    .map(x => x.entry);
  // 该 tier 内 isDefault=true 的排首位，其他按原顺序
  chain.sort((a, b) => Number(b.isDefault) - Number(a.isDefault));
  return chain;
}

const chain = resolveTierChain(modelTier);
if (chain.length === 0) {
  return sendError(res, 400, `TIER_NOT_CONFIGURED: tier=${modelTier}`);
}

// --- 8. 档内 failover 循环（v3 §4.3 逻辑）---
// ... 按 §五失败判据执行
```

---

## 六、代码改动点 + 工时

| 改动点 | 文件 | 工时 | 说明 |
|---|---|---|---|
| 类型定义 | `src/db/types.ts` | ~15 分钟 | 加 `ModelTier` 别名 + `TenantModelDefinition.tier?` |
| PG 模型读写透传 | `src/db/models/tenant-model.ts` | ~30 分钟 | `rowToTenantModel` / create / update 里把 `tier` 透传（JSONB 本来就是原样进出）；update API 接收 tier 字段 |
| SQLite 镜像 | `src/db/sqlite/models/tenant-model.ts` | ~15 分钟 | 同上 |
| 租户模型 API | `src/gateway/server-methods/tenant-models-api.ts` | ~45 分钟 | create/update 全字段接受 tier；update 对秘钥字段做"空值=保留原值"处理；不需要额外 PATCH 小接口（编辑 Modal 复用 update） |
| Agent API | `src/gateway/server-methods/tenant-agents-api.ts` | ~30 分钟 | 接收前端的 `enabledTiers: string[]`，投影成 `ModelConfigEntry[]` 存；反向读出来时按 tier 分组给前端（可选——也可以前端自己按 modelConfig 投影） |
| 接口层 tier 路由 | `src/gateway/agent-chat-http.ts` | ~1-2 小时 | `resolveTierChain` + 档内 failover 循环 + `TIER_NOT_CONFIGURED` / `TIER_EXHAUSTED` 错误；§五失败判据 |
| 失败判据 | `src/agents/models/failure-classifier.ts` (新建) | ~1 小时 | v3 §五清单 |
| 模型管理页 UI | `ui/src/ui/views/tenant/tenant-models.ts` | ~4-5 小时 | 按档分组渲染 + 联动单表单 Modal（添加/编辑双模式共用组件）+ 秘钥掩码回显 + 档位改动 soft confirm + 未分档块 |
| 模型建议常量 | `ui/src/ui/utils/model-suggestions.ts` (新建) | ~30 分钟 | `PROVIDERS_BY_TIER` / `MODEL_SUGGESTIONS` / `PROVIDER_DEFAULTS` |
| Agent 管理页 UI | `ui/src/ui/views/tenant/tenant-agents.ts` | ~2-3 小时 | 卡片展示启用档位 + 编辑 Modal（勾选档位 + 预览该档默认/备用）+ 空态提示 |
| 引导页 UI | `ui/src/ui/views/onboarding-wizard.ts` | ~2-3 小时 | shared 模式显示档位徽章 + custom 模式按档逐一添加 + 进度徽章 + 禁用已配 pill |
| i18n | `ui/src/i18n/locales/zh-CN.ts` (+ en / de 补齐) | ~1 小时 | §4.4 的 key |

**合计：约 2-3 天（AI 辅助），零迁移零新表**。

---

## 七、对比 v3 的"砍掉" vs "保留"总览

| 项 | v3 | v4 | 说明 |
|---|---|---|---|
| `TieredModelConfig` 类型 | ✅ | ❌ | 砍掉，复用 `ModelConfigEntry[]` |
| `sys_config` 推荐档位存储 | ✅ | ❌ | 前端常量代替 |
| `GET /v1/model-tier-recommendations` API | ✅ | ❌ | 同上 |
| `recommendTier()` fetcher 工具 | ✅ | ❌ | 没数据要 fetch |
| `defaultTier` 字段 | ✅ | ❌ | 用"isDefault=true 条目的 tier"代替 |
| `failoverEnabled` 字段 | ✅ | ❌ | 默认全开，后续真需要关再加租户级配置 |
| `TenantModelDefinition.tier` | - | ✅ | v4 新增（替代 v3 的 `recommendedTier`） |
| Agent 侧"三段分区 UI" | ✅ | ❌ | 只勾启用档位 |
| 添加模型 Modal 双区 accordion | ✅ | ❌ | 联动单表单（更好用） |
| 接口层 tier 路由 + failover | ✅ | ✅ | v4 保留完整 |
| `lite/standard/pro` 命名 | ✅ | ✅ | 保留 |
| `MODEL_TIER_VALUES` 接口占位 | ✅ | ✅ | 保留 |
| 失败判据（5xx/429 切，4xx/content 不切） | ✅ | ✅ | 保留 |
| `TIER_EXHAUSTED` / `TIER_NOT_CONFIGURED` 错误 | ✅ | ✅ | 保留 |
| 数据迁移 migration | ✅ | ❌ | 不需要（JSONB 内字段兼容） |

---

## 八、风险与应对

| 风险 | 可能性 | 影响 | 应对 |
|---|---|---|---|
| 旧模型数据 `tier` 全是 undefined | 高 | UI 显示"未分档"块 | 模型管理页卡片上的 `[编辑]` 按钮可补 tier；引导页新租户不受影响（全是新数据） |
| 编辑 Modal 秘钥字段回传空串误清原值 | 中 | 秘钥被清空，调用失败 | 前后端约定"空字符串 = 保留原值"；非空才 encrypt + 覆盖；前端 placeholder 文案显著提示"留空=保持不变" |
| 编辑 Modal 里秘钥掩码被 bruteforce 反查 | 低 | 泄漏尾部 4 位不构成实质风险 | `•••• abc1` 只回传显示用途，真实密文不进前端；审计日志记录编辑事件 |
| admin 不同 agent 想用不同 PRO 模型 | 低 | 做不到（v4 退化） | 如果确实出现，可在 Agent 编辑页加一个"高级模式"开关，暴露 v3 原有的 ModelRef[] 配置（后续扩展） |
| admin 改了某模型的 tier，影响所有 Agent | 中 | 所有引用该模型的 Agent 行为变化 | UI 在修改时提示"该模型被 X 个 Agent 使用，改档位会影响这些 Agent" |
| 前端 `MODEL_SUGGESTIONS` 常量跟不上新模型发布 | 中 | 新模型没预填 | 常量维护和发版节奏绑定；admin 可手填 modelId 不受影响 |
| 调用方传了 tier 但该 tier 在 agent 的 modelConfig 里没对应模型 | 中 | `TIER_NOT_CONFIGURED` 报错 | 错误信息清晰提示"该 agent 未启用 XX 档"；admin 按提示去 agent 编辑页勾上该档 |
| 共享模型（visibility=shared）必须预置 tier | 中 | 不预置时引导页没法展示推荐档位 | 平台管理员后台建 shared 模型时强制填 tier；前端校验 |

---

## 九、落地顺序

建议分 3 批小步释放风险：

1. **P0 —— 类型 + 后端透传**（0.5 天）
    - `TenantModelDefinition.tier?` + 所有读写路径透传
    - `tenant-models-api` 接收 tier 字段
    - 单测 + 手动验证一条 INSERT/UPDATE
2. **P0 —— 前端三页改造**（2-3 天）
    - 模型管理页 + Agent 管理页 + 引导页
    - i18n
    - 手动回归：新建租户走引导页、老租户跳过引导页在模型管理页补 tier、Agent 管理页勾选档位
3. **P0 —— 接口层 tier 路由**（1 天）
    - `resolveTierChain` + 档内 failover 循环
    - 失败判据 + 两类错误
    - 跟调用方（APP pipeline）联调

合计约 4 天（含缓冲），**零表结构改动，零迁移脚本**。

---

（方案稿结束）
