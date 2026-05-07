---
type: think
author: yuebilin
created: '2026-04-30T16:03:33+08:00'
body_source: manual
index_state: indexed
---
# Category 与讨论可见范围补充说明

我补充一下目前实现还缺的几个闭环，尤其是 Category 可见范围和 Matter/讨论可见范围之间的边界。

当前规则已经明确：Category 是权限上限，Matter/讨论只能在 Category 范围内继续收窄，不能突破 Category。

## 1. 已存在 Category 还缺少管理入口，admin 可以新增和修改 Category

现在新建 Matter 时，如果同时创建新的 Category，可以设置这个 Category 的可见范围。

但 Category 不应该只能依赖“发帖时顺手创建”。后续需要有一个 admin 管理入口，允许 admin 主动维护 Category。

建议本期补一个 admin 的 Category 管理页面：

- admin 可以新增 Category。
- admin 新增 Category 时必须设置可见范围：公开 / 指定角色可见。
- admin 可以修改已存在 Category 的可见范围。
- admin 可以查看每个 Category 当前是公开还是指定角色可见。
- 普通用户、Matter creator、Matter owner 都不能修改 Category 可见范围。

这样 Category 有两个创建入口：

1. 新建 Matter 时顺带创建 Category。
   创建者可以设置这个新 Category 的初始可见范围。
2. admin 后台主动创建 Category。
   admin 可以提前建好 Category，并设置它的可见范围。

无论从哪个入口创建 Category，都必须同步设置 Category 可见范围，不能偷偷默认创建。

## 2. 后端需要校验 user_ids 不能绕过 Category

现在规则已经明确：Matter 可见范围不能大于 Category。

这个限制不只针对角色，也要针对单独用户。

比如：

```text
Category1 只允许「技术部」可见。

小王没有「技术部」角色。
```

这时即使小李是讨论1的 owner，也不能把小王单独加入讨论1的可见范围。

所以后端保存 Matter 可见范围时，需要校验：

- Matter.roles 必须是 Category.authorized_roles 的子集。
- Matter.user_ids 对应的用户，必须至少拥有 Category 范围内的某个角色。
- 如果用户不在 Category 范围内，返回 `visibility_scope_exceeds_category`。

前端选择器可以提前过滤，但不能只靠前端，后端必须兜底。

## 3. 新建 Category 必须始终带 new_category_visibility

需求里已经写了：新建 Matter 时如果同时创建 Category，必须同步设置 Category 是公开还是指定角色可见。

所以这个规则不应该只在 Matter 是指定范围时才生效。

只要 Category 不存在，POST /api/matters 就必须带：

```json
{
  "new_category_visibility": {
    "mode": "public",
    "authorized_roles": []
  }
}
```

或：

```json
{
  "new_category_visibility": {
    "mode": "restricted",
    "authorized_roles": ["技术部"]
  }
}
```

这样可以避免新 Category 被偷偷按默认值创建，导致权限边界不清楚。

## 4. 修改 Category 可见范围时，要处理已有讨论越界问题

如果 admin 后续可以修改已存在 Category 的可见范围，就会出现一个需要明确处理的情况：Category 范围被收窄后，下面已有讨论可能会超过新的 Category 上限。

举例：

```text
Category1 可见范围：技术部、行政部

讨论1 可见范围：技术部
讨论2 可见范围：行政部
讨论3 可见范围：技术部、行政部
```

这时三个讨论都是合法的，因为它们都没有超过 Category1 的范围。

现在 admin 想把 Category1 改成：

```text
Category1 可见范围：技术部
```

那么修改后会变成：

```text
讨论1：仍然合法，因为技术部还在 Category1 里。
讨论2：不合法，因为行政部已经不在 Category1 里。
讨论3：不合法，因为它包含了行政部。
```

也就是说，Category 是上限。上限变小以后，下面已有 Matter 的可见范围也必须仍然小于等于这个上限。

这里有两种处理方式。

方案 A：拒绝保存，并提示冲突讨论。

后端检查该 Category 下所有 Matter。如果发现有讨论会超过新的 Category 范围，就拒绝这次修改，并返回冲突列表。

示例：

```json
{
  "code": "category_visibility_conflict",
  "conflicts": [
    {
      "matter_id": "讨论2",
      "title": "讨论2",
      "roles": ["行政部"],
      "user_ids": []
    },
    {
      "matter_id": "讨论3",
      "title": "讨论3",
      "roles": ["行政部"],
      "user_ids": []
    }
  ]
}
```

前端提示 admin：需要先调整这些讨论的可见范围，再回来修改 Category。

方案 B：自动收窄下面已有讨论。

系统自动把讨论2、讨论3里超出的角色或用户删掉。

我建议本期采用方案 A，不自动修改已有讨论。

原因是自动收窄会悄悄改变已有讨论的访问权限，可能导致 owner、creator 或其他协作者突然看不到原来的讨论。权限变化应该由人明确操作，而不是系统自动替用户做决定。

所以本期建议规则：

- admin 修改 Category 可见范围前，后端先做冲突检查。
- 如果没有冲突，保存新的 Category 可见范围。
- 如果有冲突，拒绝保存，返回冲突讨论列表。
- 前端展示冲突原因和讨论列表。
- admin 先去调整这些讨论的可见范围，再重新修改 Category。
- 本期不做自动收窄。

这样可以保证：Category 始终是上限，讨论始终不能超过 Category，同时不会静默改变已有讨论权限。
