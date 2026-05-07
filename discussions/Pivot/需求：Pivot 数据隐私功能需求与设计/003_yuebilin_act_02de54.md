---
type: act
author: yuebilin
created: '2026-04-29T17:15:54+08:00'
index_state: indexed
---
# Pivot 角色字段与 Matter 可见范围设计

| 项 | 值 |
|---|---|
| 文件 | `AI-docs/designs/2026-04-29-role-and-matter-visibility-design.md` |
| 创建日期 | 2026-04-29 |
| 关联需求 | `需求：Pivot 数据隐私功能需求与设计` |
| 依赖 | `2026-04-28-user-management-design.md` / `2026-04-28-user-management-plan.md` |

---

## 0. 需求概览

### 0.1 这次要做什么

用户管理体系当前已经有 `pivot_user.role` 字段，但同事方案里暂时写死为 `admin/member`。本需求把 `pivot_user.role` 扩展成一个**可维护、可多选**的角色字段。

本次要补齐三件事：

- 用户管理后台允许修改用户的 `pivot_user.role`
- 一个用户可以同时拥有多个角色，比如「行政部门」「技术部门」
- Matter 创建和已发布 Matter 修改时，可以按「全部 / 角色 / 用户」选择可见范围

### 0.2 单字段、多角色口径

本设计只使用一个角色字段：

```text
pivot_user.role
```

但这个字段保存的是角色集合。建议 DB 仍用 `TEXT`，内容保存为 JSON 字符串数组：

```json
["member", "技术部门", "客户A项目组"]
```

角色示例：

```text
admin
member
行政部门
技术部门
客户A项目组
```

一个用户可以同时有多个角色。Matter 可见范围也可以选择多个角色。用户只要命中任一授权角色，或者被单独授权，就能看到该 Matter。

| 用途 | 使用方式 |
|---|---|
| 后台管理员 | 判断 `pivot_user.role` 数组是否包含 `admin` |
| 普通成员 | 可以继续包含 `member` |
| 部门/业务角色 | 直接保存到 `pivot_user.role` 数组 |
| Matter 可见范围 | 匹配用户 `role` 数组和 Matter 授权角色的交集 |

`admin` 是内置保留角色：

- 不可删除
- 不需要在发帖或修改可见范围时勾选
- 不在可见范围选择器的「角色」里显示
- 不在可见范围选择器的「用户」里显示
- 只要 active 用户的 `pivot_user.role` 包含 `admin`，就直接可读所有 Category / Matter

### 0.3 选择器形态

创建或修改可见范围时，前端使用类似飞书「选择联系人」的双栏弹窗，而不是简单平铺树。

```text
选择可见范围

┌──────────────────────────────────────┬────────────────────────────┐
│ 搜索用户、角色                       │ 已选：2 个                  │
│                                      │                            │
│ 角色 > 技术部门              下级 >  │ 技术部门                   │
│ 角色 > 行政部门              下级 >  │ 张三                       │
│ 用户   张三                          │                            │
│ 用户   李四                          │                            │
└──────────────────────────────────────┴────────────────────────────┘
                         取消          确认
```

左侧用于搜索和浏览，右侧固定展示已选对象。左侧对象分两类：

- 角色：来自 `pivot_user.role` 数组展开后的候选，例如「技术部门」「行政部门」
- 用户：来自 active 用户列表，但排除 role 数组包含 `admin` 的用户

角色可以展开查看命中的用户，类似参考图里的「下级」入口。展开只用于预览和辅助选择，不改变提交结构。

实际提交给后端：

```json
{
  "visibility": "restricted",
  "roles": ["技术部门"],
  "user_ids": ["user_zhangsan"]
}
```

后端实时按用户当前 `pivot_user.role` 数组判断。用户被移出「技术部门」后，后续列表、详情、搜索、AI、MCP、SSE 都立即失去依赖「技术部门」授权获得的访问权。

---

## 1. 关键决策一览

| 项 | 决策 |
|---|---|
| 角色来源 | `pivot_user.role` 字段 |
| 用户和角色关系 | 单字段、多角色；一个用户可以同时拥有多个角色 |
| 新增角色方式 | 管理员给某个用户添加新角色值，该值进入角色候选 |
| `admin` 角色 | 内置保留，不可删除；不参与选择器；active admin 直接可读全部 |
| Category 可见范围 | 公开 / 指定角色 |
| Matter 可见范围 | 公开 / 指定角色 + 指定用户 |
| Matter 越界规则 | Matter 只能在 Category 范围内收紧，不能扩大 |
| 已发布 Matter 修改权限 | owner 可改可见范围 |
| 修改历史 | Matter 可见范围每次变更写审计 |
| @ 候选 | 只返回当前 Matter 可读用户，不返回 admin |
| 通知 | 完全公开保留现有群通知；非完全公开只私信 owner 和 @ 用户 |

---

## 2. 数据模型

### 2.1 `pivot_user.role`

沿用用户管理体系里的 `pivot_user.role` 字段，但从写死枚举改成可扩展的多角色字段。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| role | TEXT | 是 | JSON 字符串数组，默认 `["member"]` |

实现要求：

- 移除 DB 层 `CHECK(role IN ('admin','member'))`
- 后端把 `role` 解析为字符串数组
- 数组不能为空
- 每个角色名非空、长度受限、不能包含控制字符
- `admin` 和 `member` 作为内置角色保留
- `admin` 不可删除，至少保留一个 active admin
- 角色候选从所有 active 用户的 `role` 数组展开后去重生成
- 可见范围选择器排除 `admin`

### 2.2 `category_visibility`

Category 的可见范围。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| category_id | 字符串 | 是 | Category 标识 |
| visibility | 枚举 | 是 | `public` / `restricted` |
| updated_at | 时间戳 | 是 | 更新时间 |
| updated_by | 字符串 | 是 | 操作人 |

### 2.3 `category_visibility_role`

Category 授权角色值。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| category_id | 字符串 | 是 | Category 标识 |
| role | 字符串 | 是 | 角色值，例如「技术部门」 |

约束：`(category_id, role)` 组合唯一。

### 2.4 `matter_visibility`

Matter 的可见范围头表。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| matter_id | 字符串 | 是 | Matter id |
| visibility | 枚举 | 是 | `public` / `restricted` |
| updated_at | 时间戳 | 是 | 更新时间 |
| updated_by | 字符串 | 是 | 操作人 |

### 2.5 `matter_visibility_role`

Matter 授权角色值。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| matter_id | 字符串 | 是 | Matter id |
| role | 字符串 | 是 | 角色值，例如「技术部门」 |

约束：`(matter_id, role)` 组合唯一。

### 2.6 `matter_visibility_user`

Matter 额外授权用户。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| matter_id | 字符串 | 是 | Matter id |
| pivot_user_id | 字符串 | 是 | 授权用户 |

约束：`(matter_id, pivot_user_id)` 组合唯一。

### 2.7 `matter_visibility_audit`

Matter 可见范围变更审计。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| id | 字符串 | 是 | 主键 |
| matter_id | 字符串 | 是 | Matter id |
| changed_by | 字符串 | 是 | 操作人 |
| changed_at | 时间戳 | 是 | 操作时间 |
| before_json | JSON 字符串 | 是 | 变更前范围 |
| after_json | JSON 字符串 | 是 | 变更后范围 |
| reason | 字符串 | 否 | 修改原因 |

---

## 3. 权限服务

新增 `server/permissions.py`，统一处理 Category / Matter 的读写判断。

核心方法：

```python
class PermissionService:
    def can_read_category(self, user_id: str, category_id: str) -> bool: ...
    def can_read_matter(self, user_id: str, matter_id: str) -> bool: ...
    def can_write_matter(self, user_id: str, matter_id: str) -> bool: ...
    def can_update_matter_visibility(self, user_id: str, matter_id: str) -> bool: ...
    def filter_visible_matters(self, user_id: str, matters: list[MatterSummary]) -> list[MatterSummary]: ...
    def list_mentionable_users(self, user_id: str, matter_id: str) -> list[PivotUser]: ...
    def validate_matter_visibility_scope(self, matter_id: str, scope: VisibilityScope) -> None: ...
```

读权限计算：

```text
active 用户 roles 包含 admin：
  直接可读所有 Category / Matter

Category 公开：
  Matter 公开 -> 全部 active 用户可读
  Matter restricted -> 用户任一角色命中 Matter.roles，或命中 Matter.user_ids，可读

Category restricted：
  Matter 公开 -> 用户任一角色命中 Category.roles，可读
  Matter restricted -> 先命中 Category.roles，再命中 Matter.roles 或 Matter.user_ids
```

`can_update_matter_visibility` 初版规则：

- Matter owner 可以修改
- 修改者必须已经能读该 Matter
- 新范围不能越过所属 Category 范围
- 创建者和 owner 必须仍在可读范围内，否则拒绝

---

## 4. 接口设计

### 4.1 用户管理角色接口

用户管理体系已有 `changeUserRole` / `/api/admin/users/{id}/role` 思路，本需求把它从二选一改为多选角色。

| 方法 | 路径 | 说明 |
|---|---|---|
| `GET` | `/api/admin/roles` | 返回当前可选角色，来自所有 active 用户 `role` 数组展开去重 |
| `POST` | `/api/admin/users/{user_id}/role` | 修改用户 `pivot_user.role` 角色集合，允许新角色名 |

`POST /api/admin/users/{user_id}/role` 请求示例：

```json
{
  "roles": ["member", "技术部门", "客户A项目组"]
}
```

`GET /api/admin/roles` 返回示例：

```json
{
  "items": [
    { "role": "admin", "user_count": 1 },
    { "role": "member", "user_count": 8 },
    { "role": "行政部门", "user_count": 2 },
    { "role": "技术部门", "user_count": 3 }
  ]
}
```

### 4.2 可见范围选择器数据

| 方法 | 路径 | 说明 |
|---|---|---|
| `GET` | `/api/visibility-options` | 返回树状选择器需要的全部、角色、用户 |

返回示例：

```json
{
  "all": { "label": "全部用户", "value": "public" },
  "roles": [
    {
      "role": "技术部门",
      "users": [{ "id": "u1", "display_name": "张三" }]
    }
  ],
  "users": [{ "id": "u1", "display_name": "张三" }]
}
```

返回规则：

- `roles` 不返回 `admin`
- `users` 不返回 role 数组包含 `admin` 的用户
- admin 不需要被勾选也能访问所有 Matter
- 左侧列表按「角色在前、用户在后」展示；搜索同时命中角色名和用户名
- 右侧返回当前已选角色和用户，便于编辑已发布 Matter 时回显

### 4.3 Matter 创建与修改可见范围

`POST /api/matters` 增加：

```json
{
  "visibility": {
    "mode": "restricted",
    "roles": ["技术部门"],
    "user_ids": []
  }
}
```

新增：

| 方法 | 路径 | 说明 |
|---|---|---|
| `GET` | `/api/matters/{matter_id}/visibility` | 读取当前范围 |
| `PUT` | `/api/matters/{matter_id}/visibility` | owner 修改已发布 Matter 的可见范围 |

`PUT` 成功后：

- 写 `matter_visibility*` 表
- 写 `matter_visibility_audit`
- 刷新 Matter 详情返回里的 `visibility`
- 对非完全公开 Matter 不触发群通知

---

## 5. 前端设计

### 5.1 Admin 用户角色修改

在用户管理页里扩展角色编辑：

- 角色下拉展示当前已有角色
- 支持输入新角色名
- 支持多选，一个用户可以同时选中多个角色
- 保存后直接更新该用户的 `pivot_user.role` 角色集合
- 新角色名会在后续选择器里出现

不单独做「角色成员管理页」。用户有哪些角色，直接在用户管理页里编辑。

### 5.2 VisibilityScopePicker

新增通用组件 `VisibilityScopePicker`，用于新建 Matter 和已发布 Matter 修改范围。

交互：

- 顶部提供 `全部用户` / `指定范围` 两种模式
- 选择 `指定范围` 后打开双栏弹窗
- 左侧顶部是搜索框，占位文案：`搜索用户、角色`
- 左侧列表先显示角色，再显示用户
- 角色行显示角色图标、角色名、`下级 >` 入口
- 点击角色行勾选该角色，表示授权当前所有 `pivot_user.role` 包含该值的用户
- 点击 `下级 >` 进入该角色下用户预览，可返回上一级
- 用户行显示头像、姓名，勾选后表示额外授权用户
- 右侧固定展示已选角色和用户，顶部显示 `已选：N 个`
- 右侧每项可移除
- `admin` 角色和 admin 用户不展示，不需要选择
- 如果 Category 是 restricted，则左侧只展示 Category 范围内可选的角色和用户

### 5.3 Matter 详情修改入口

Matter 详情页在 owner 可写时显示一个「可见范围」入口：

- 当前范围摘要：公开 / 指定范围
- 点击打开 dialog
- 保存前显示影响提示：被移出范围的人将无法再访问该 Matter
- 保存后刷新详情与 timeline

---

## 6. 通知、搜索、AI、MCP

| 出口 | 规则 |
|---|---|
| Matter 列表 | `filter_visible_matters` 过滤 |
| Matter 详情 | `can_read_matter` 失败返回 403 |
| 追加文件 / 评论 / result | `can_write_matter` |
| @ 选择器 | `list_mentionable_users` |
| 搜索 | 只搜索可读 Matter |
| Web AI | 读 index / Markdown 前校验 Matter 权限 |
| MCP | 走 REST 后自然继承；本地工具直接读文件时必须补权限 |
| SSE | live 和 replay 都按 `matter.read` 过滤 |
| IM 通知 | 非完全公开 Matter 不发群，只私信 owner 和 @ 用户 |

---

## 7. 验收标准

主线验收：

> 管理员把 B 的 `pivot_user.role` 改为 `["member", "技术部门"]`；A 创建只给「技术部门」可见的 Matter；B 能看到，C 在列表、搜索、SSE、IM、AI、MCP 中都看不到；A 后续把 C 作为单独用户加入范围后，C 立即可见。

具体验收：

| 场景 | 预期 |
|---|---|
| 管理员给用户添加新角色 | 新角色进入角色候选 |
| active admin 查看任意 Matter | 不受可见范围限制，直接可见 |
| 发帖或修改可见范围 | 不展示 `admin` 角色和 admin 用户 |
| 用户任一角色命中 Matter 授权 | 用户获得 Matter 可读权限 |
| 用户被移除某个角色 | 后续不能再读只依赖该角色授权的 Matter |
| 新建公开 Matter | 所有 active 用户可见；群通知保持现有逻辑 |
| 新建指定角色 Matter | 只有命中角色或单独授权用户可见；不发群 |
| owner 修改已发布 Matter 范围 | 新范围实时生效，并写审计 |
| 非 owner 修改范围 | 返回 403 |
| Matter 范围越过 Category | 返回 400 |
| @ 不可见用户 | 前端不出现；伪造请求后端阻断 |
| C 直接访问 URL | 返回 403 |
| C 用 AI/MCP 读取 | 返回无权限 |

---

## 8. 本期不做

| 项 | 原因 |
|---|---|
| 组织架构同步 | 先手动维护 `pivot_user.role`，后续再接企业通讯录部门 |
| Category 专属群通知 | 群成员和 Category 权限一致性未设计，本期非公开一律不发群 |
