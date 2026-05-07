---
type: think
author: liuyu
created: '2026-04-30T17:10:52+08:00'
index_state: indexed
---
整体方向我同意 006 的 §3、§4，§1 admin 那块也大致赞成把 admin 拉进治理路径——不过 §1 和 §2 放在一起看，有两个产品口径需要再讨论一下。

## 一、admin 拿「治理权」，不拿「业务读权」

§1 的意图我理解是让 admin 作为治理兜底，可以维护 Category 这种基础设施性质的可见性边界，特别是 creator 离场后没人能改的场景。这个我同意。但要避免 admin 顺势拿到「业务读权」，那样就和 005 §2.2 已经定的口径冲突了。建议把规则拆成两条：

- **业务读权**：admin **不享**特权。读非公开 Matter 必须通过普通业务角色或 authorized_users 获得授权（保持 005 §2.2 + 004 §2 不变）。
- **治理权**：admin 可改 Category visibility（接受 §1）。但本期不扩到 Matter visibility——Matter 仍然是 `creator ∪ owner` 改，admin 不介入；理由是 Matter 数量大，admin 改的频率 / 影响面比 Category 大一档，让创建者自己负责更稳。

具体落地我倾向这样处理：

- Category visibility 修改：`creator ∪ admin`
- Matter visibility 修改：`creator ∪ owner`
- admin 后台 Category 管理页只展示 Category 元数据 + 该 Category 下 matter 数量，**不展示 matter 标题、不进 matter 详情**——避免 admin 通过列表反推业务信息
- admin 改 Category visibility 时如果有冲突 matter，沿用 §4 的拒绝保存方案，配合 dry-run 预览（影响面只显示 matter 数量，不显示具体标题）

唯一遗留场景是 Matter creator/owner 都离场时 visibility 没人改——本期建议先登记到「本期不做」，等真出现再决定走 admin 兜底还是 owner 转交。Category 由 admin 兜底已经覆盖了组织级问题。

## 二、authorized_users 是否能 escape Category 角色边界

§2 说「Matter.user_ids 对应用户必须至少拥有 Category 内某个授权角色」——这条我重新想了一下，和需求帖 §2 横向规则其实有点张力：

- 「Matter 不越界」：Matter 不能超过 Category 范围
- 「角色 + 用户」：Matter 可见范围 = 授权角色 ∪ 授权用户

如果严格执行第一条（即 §2 现在的写法），第二条里的"+ 授权用户"就基本没用——能进 Matter 的用户必然先在 Category 范围内（要么持 Category 授权角色，要么 Category=public），既然他已经能读 Category 下任何东西，再单独加进 Matter authorized_users 是冗余的。

实际场景里，"+ 指定用户"的本意是给 Category 角色覆盖不到的人单独开口子，比如：

- Category=「技术部门」，临时拉财务部小王进某条 matter review
- Category=「客户A」，想给客户的外部联系人 review 某条 matter
- Category=「财务部」，财务建的 matter 想拉法务做合规 review

按 §2 现在的写法，这些场景要么"先把人加到 Category 角色"——但这样他能看到该 Category 下所有他被授权的 matter，权限放大了；要么"建新 Category"，运维成本高。

我的倾向是：**让 authorized_users 真正成为 Category 角色边界的 escape hatch**，单人授权先于 Category 角色判断。Category 仍然是上限，但上限的语义是"角色集合"层面的——单人破例由 matter creator 显式做，留迹清楚，不会变成默认通道。

`can_read_matter` 的判断顺序：

```
1. 命中 matter.authorized_users → 可读（escape Category）
2. matter.mode=public：
     - category.mode=public → 可读
     - category.mode=restricted → 命中 category.authorized_roles 才可读
3. matter.mode=restricted：
     - 命中 matter.authorized_roles 且（category 为 restricted 时）
       命中 category.authorized_roles → 可读
     - 否则不可读
```

如果走这条，§2 的校验调整为：只校验 `matter.authorized_roles ⊆ category.authorized_roles`，不校验 `matter.user_ids 必须在 category 范围内`。

如果你坚持严格路径，也可行，但前端 visibility picker 在 Category=restricted 时就需要把"添加用户"入口也按 Category 角色范围裁剪——否则 creator 选完发现 422 体验很差。

---

§3（new_category_visibility 不论 mode 都强制）和 §4（收窄 Category 拒绝保存+列冲突）保留，无意见。

不知道你怎么看？
