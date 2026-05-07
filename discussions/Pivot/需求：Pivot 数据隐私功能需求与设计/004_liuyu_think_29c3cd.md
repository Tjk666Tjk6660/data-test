---
type: think
author: liuyu
created: '2026-04-29T17:57:37+08:00'
index_state: indexed
---
整体方向我同意，特别是 §0.3 的双栏选择器（飞书联系人风格）比内嵌 chip 成熟，建议直接走这条路。下面是落地前需要补齐的契约缺口，每条直接给结论；测试用例统一放在文末。

## 1. 修改权限范围：owner ∪ 创建者

§1 决策表写「owner 可改」，与 zhangbo 002 帖追补的「**owner 和创建者**」不一致。matter 时代 `timeline[0].creator` 与 `timeline[0].owner` 可不同。

**结论**：`can_update_matter_visibility(user, matter)` 判定为 `user == owner OR user == creator`。

## 2. admin 不享有全局读权

§0.2 让 active admin 直接读所有 Category / Matter，与需求帖 §5.1 推荐的「无全局可读超管」相反，且未承诺 break-glass 审计。

**结论**：admin 仅是系统角色（管 admin 后台），**不**参与业务可见性。admin 对未授权他的 restricted matter 与普通用户行为一致。

## 3. 新建 Matter 同步建 Category 必须设 visibility

需求帖 §3.1 后端要点："新建 Matter 时如果同时创建 Category，必须同步设置 Category 为公开或指定角色可见"。§4.3 没体现这条强制。

**结论**：POST /api/matters 当 `category` 不存在时，body 必须携带 `new_category_visibility: {mode, authorized_roles}`。Category + Matter 单事务建立，校验 Matter 范围 ⊆ Category 范围。

## 4. 不可见 matter 统一返回 404

§4.3 / §7 写 403。403 = "存在但你无权"，攻击者可枚举 matter_id 探测存在性。

**结论**：所有不可见路径统一返回 `404 {"code":"matter_not_found"}`，与 matter 真不存在时同形。详情、PATCH visibility、POST files / comments / result、收藏、read、file_read 全部如此；SSE 不推送任何相关事件。

## 5. AI tools 直读 fs 必须逐 handler 加权限

§6 表格只写"Web AI 读 index / Markdown 前校验"。但 `server/ai/tools.py` 直读 `index_dir` 和 `discussions_dir`，**绕过 REST**——MCP 走 PAT 调 REST 自动继承，Web AI 不行。

**结论**：

- MCP tools：不动，REST 加权限自然继承
- Web AI tools：`AITools` 注入 `permission_service` + `current_user_id_provider`，user_id 经 ai chat handler 的 contextvar 传入；以下 5 个 handler 全部加 `can_read_matter` 拦截：`list_matters` / `search_indexes` / `read_matter_index` / `read_post` / `read_posts`
- `read_post` / `read_posts` 由 path 反查 matter_id：`discussions/<cat>/<matter_id>/<file>.md`

## 6. SSE 过滤位置 + 新增 visibility_changed 事件

§6 没说 per-user 过滤的位置；ring buffer 是全局共享的。

**结论**：

- emit 不过滤，事件全量进 ring
- live 路径：每条事件在 put 队列前 `can_read_matter(user, event.matter_id)`，不可见即丢
- replay 路径：从 ring 拉出的切片走 `filter_visible_matters`，过滤后逐条 yield
- 新增 `matter.visibility_changed` 事件：接收对象 = 改前可见用户 ∪ 改后可见用户（并集）。前端收到后无条件 invalidate 该 matter 的列表项与详情缓存

## 7. 迁移默认全公开 + dry-run + 回滚

§7 完全没提既有数据怎么处理。

**结论**：迁移脚本（`scripts/migrate_visibility.py`）按用户管理 spec §9 格式办：单事务、`--dry-run` 支持、既有 matter 与 category 全部默认 `public`，行为与今天零变化；runbook：停服 → 备份 → dry-run → 正式跑 → 启动 → 出错恢复备份回滚代码。

---

## 附两个小调整

**A. 角色 typo 防御**：003 把角色当字符串，管理员手输"技术部"/"技术部门"会产生两个独立角色无察觉。

**结论**：POST /api/admin/users/{id}/role 收到不在已有候选集合中的角色名时，必须在 body 中显式 `confirm_create_role: true` 才允许通过；否则返回 422 + code=`unknown_role` + suggestions[]。前端用户编辑页用「选已有」+「新建角色」两个分立动作落实。

**B. SQLite 重建路径**：003 把 visibility 全放 SQLite，DB 损坏 / 迁服务器后无法从 git 工作区还原，违反「Git 是内容权威」。

**结论**：matter visibility 反向落在 `index/<id>.index.yaml` 的 `matter.visibility` 字段（与 `current_status` 同层），category visibility 落在 `categories/<name>.yaml`。SQLite 仅作 list / filter cache，启动 `recover()` 之后 `rebuild_acl_cache()` 全量重灌。审计走 git log，不另建审计表。

---

## 测试用例补全要求

请按上述结论补全单测与集成测，至少覆盖以下场景（编号对应上文）：

**1. 修改权限**
- `test_visibility_creator_can_update`：A 创建后把 owner 转给 B，A 仍可改 visibility
- `test_visibility_owner_can_update`：B（owner）可改 visibility
- `test_visibility_neither_owner_nor_creator_forbidden`：C 调 PUT 返回 403

**2. admin 无全局读权**
- `test_admin_no_global_read_on_restricted_matter`：admin 对未授权他的 restricted matter，列表 / 详情 / SSE / AI 全部不可见
- `test_admin_authorized_path_works_normally`：admin 被显式加入 authorized_users 后，所有读路径正常

**3. 同步建 Category**
- `test_new_category_requires_visibility`：category 不存在且缺 new_category_visibility → 422 + code=`new_category_visibility_required`
- `test_matter_scope_must_be_within_new_category`：Matter authorized_roles 越过 new_category authorized_roles → 422 + code=`matter_visibility_exceeds_category`
- `test_atomic_create_rolls_back_on_matter_failure`：matter 写失败时 Category 在 git 工作区 + SQLite 都不留痕

**4. 统一 404**
- `test_restricted_matter_returns_404_for_unauthorized`：C 调 GET / PATCH / POST files / comments / result / favorite / read 全部 404
- `test_existence_is_not_leaked`：matter 真不存在 vs 存在但无权，两种响应字节级一致

**5. AI tools 权限**
- `test_ai_list_matters_filters_by_visibility`
- `test_ai_search_indexes_skips_restricted_hits`：关键词命中 restricted matter index，C 调 0 hit
- `test_ai_read_matter_index_blocks_unauthorized`：ToolError 文案与"matter 不存在"一致
- `test_ai_read_post_path_traversal_to_acl`：C 调 `read_post(discussions/cat/restricted_matter/...md)` 返回 ToolError
- `test_ai_path_resolution_uses_current_user_contextvar`：同一 AITools 实例并发不同 chat session 下 user_id 各自独立

**6. SSE**
- `test_sse_live_drops_unauthorized_event`：C 不应收到 restricted matter 的 .updated 事件，B 应收到
- `test_sse_replay_filters_by_current_user`：C 用 last-event-id 重连 replay 时过滤掉 restricted 事件
- `test_visibility_changed_emits_to_union`：matter public → restricted(only B)，B 与 C 都收到 `matter.visibility_changed`
- `test_visibility_changed_post_change_filtering`：该事件后续的 .updated，C 不再收到，B 继续收到

**7. 迁移**
- `test_migration_dry_run_no_writes`：`--dry-run` 跑完 SQLite + git 工作区零变化
- `test_migration_default_public_for_existing`：迁移后所有 matter_acl 行 visibility=public、authorized_roles=[]、authorized_users=[]
- `test_migration_baseline_unchanged`：迁移前后 GET /api/matters 对同一用户返回的 matter 列表行级一致
- `test_migration_idempotent`：连跑两次第二次为 no-op

**A. 角色 typo 防御**
- `test_unknown_role_requires_confirm`：传新角色名不带 confirm → 422 + suggestions
- `test_unknown_role_with_confirm_creates`：带 `confirm_create_role: true` → 200，新角色出现在 GET /api/admin/roles

**B. visibility 上 YAML**
- `test_acl_cache_rebuilt_from_yaml_on_boot`：删 SQLite 重启，所有 visibility 从 YAML / categories 自动恢复
- `test_visibility_change_writes_yaml_and_cache`：PUT visibility 同时写 YAML、刷 cache、产生 git commit
- `test_audit_via_git_log`：`git log -p index/<id>.index.yaml` 能列出全部历史 visibility 变更
