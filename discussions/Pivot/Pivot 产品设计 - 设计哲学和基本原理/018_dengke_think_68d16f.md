---
type: think
author: dengke
created: '2026-04-25T00:20:29+08:00'
index_state: indexed
---
针对你提出的迁移方案，整体框架清晰且边界意识很强。结合 Pivot 产品文档的设计哲学与老数据实际特征，有四项核心规则需要调整，请评审确认：

## 1. `comment` 不作为独立 timeline 事件
老版 `comment` 本质是 `proposal` / `reply` 的附属增量更新，不具备独立推进语义。
**修订规则**：迁移脚本识别到 `type: comment` 时，**不生成独立的 `timeline[i]`**。将其 `refs` 合并至父级帖子的 `refer[]`；若包含 `@mention` 或关键补充，转为父级 timeline item 的 `comments[]`。原 MD 文件保留在磁盘，仅从 index 时间线中剔除，避免污染事项主线。

## 2. `concluded` / `produced` → `act` + 触发 `status_change`
老语境下这两类状态均表示讨论已收口（得出结论或明确下一步），直接映射为 `finished` 会切断后续执行链路，且与 §八原则中“plan 阶段完成即转入 execution”的设计不符。
**修订规则**：
- 状态为 `concluded` / `produced` 的 thread，取其最后一条有效文件作为该事项 `planning` 阶段的终点。
- 该条目 `type` 标记为 `act`。
- 自动追加 `status_change` 事件：`{ from: "planning", to: "executing", triggered_by: "<文件path>" }`
- `matter.current_status` 终态统一写为 `executing`，保留后续推进能力。

## 3. 帖子 Frontmatter 需按新版格式更新
“原始 Markdown 是第一手事实”指**正文内容（body）不可篡改**，但元数据属于结构化机器可读层，应当与新版 schema 对齐，否则会导致 index 与文件头语义割裂。
**修订规则**：
- **正文一字不改**，严格保留原始渲染效果。
- 重写 frontmatter 字段，映射关系如下：
  | 老字段 | 新字段 | 说明 |
  |---|---|---|
  | `type: proposal` | `type: think` | 规划期讨论 |
  | `type: reply` | `type: think` / `act` | 若该 reply 触发 `concluded/produced` 则标记 `act` |
  | `type: comment` | 保留原值 | 文件本身不动，仅从 index 剥离 |
  | `status` | `status` | 同步新 matter 状态 |
  | `created` | `created_at` | ISO 格式保留 |
  | `author` | `creator` / `owner` | 默认双填同值（后续可人工按责调整） |
- 迁移脚本使用 YAML parser 安全覆盖头部，不碰 body 区域。

## 4. 上线前验证流程调整
原方案缺少实战验证环节。按规范，需先在生产副本上跑通，确认无误后再切生产。
**修订流程**：
1. **Fork 生产库**：将当前生产仓库（含完整 git history 与 `var/index`）完整克隆至测试环境。
2. **脚本改造**：在 fork 副本上实施上述三项规则变更。
3. **连接测试版 Pivot**：执行 dry-run → 确认 `report` 无 error → `--apply` 落盘。
4. **人工验证**：启动测试版 Pivot 指向该副本，重点核对：
   - matter 状态机跳转是否正确（`planning → executing`）
   - timeline 是否已过滤冗余 `comment` 事件
   - MD frontmatter 是否与 index 一致且 body 未受损
   - `mention → comments` 转换是否完整
5. **生产上线**：验证通过后，按原停服→备份→拉取→apply→启服流程执行。

## 下一步
我将按上述规则重构 `scripts/migrate_index_schema.py` 的核心映射逻辑与 frontmatter 重写模块，并补充对应单测。脚本就绪后，先在 fork 出的测试副本跑全量 dry-run，输出 `migration-report.json` 供二次复核。确认无误后再安排测试版联调与生产发布。

如无其他异议，即按此方向推进。
