---
type: think
author: zhangbo
created: '2026-04-29T11:00:41+08:00'
index_state: indexed
---
# 新需求：Matter 详情页视觉体验与工作流改造

背景

当前 Matter 详情页主要按时间顺序展示帖子。
帖子一多，用户不容易看清当前 Matter 状态，也不容易快速找到不同类型的帖子。

目标

让用户进入 Matter 后能快速看到：

当前 Matter 处于哪个状态
各类型帖子分别有哪些
每类帖子有多少
act 和 verify 的对应关系
点击状态或帖子后能定位到正文
顶部状态机

顶部展示 Matter 状态机：

planning → executing → paused → finished → cancelled → reviewed
状态说明：

planning：规划中
executing：执行中
paused：已暂停
finished：已完成
cancelled：已取消
reviewed：已复盘
当前状态需要高亮。

paused 和 cancelled 不是常规推进阶段，而是特殊状态。
展示时需要和 planning、executing、finished、reviewed 区分出来。

帖子索引

在状态机下方展示帖子类型索引：

think
act
verify
result
insight
每类展示：

类型名称
帖子数量
帖子入口
点击帖子入口，正文自动定位到对应帖子。

执行与验收关系

act 和 verify 需要绑定展示。

支持：

一个 act 对应一个 verify
多个 act 对应一个 verify
展示方式：

act 下展示关联的 verify
verify 下展示关联的 act
点击 act 或 verify 都能定位到正文帖子

正文展示

正文按帖子类型分组展示。

每组内部按时间顺序展示
act、verify、result 默认展开
think 较多时默认收起部分内容
insight 较多时也可以收起部分内容
用户可以手动展开全部内容
如果定位目标在折叠区域内，自动展开后再定位
通知定位

从 IM、提醒、通知中心进入时，应直接定位到对应帖子。

开放问题

是否支持人工标记关键帖子。

如果支持，需要明确：

谁可以标记
是否允许多个关键帖
顶部索引是否优先展示
正文是否特殊高亮
是否影响搜索、提醒、AI 上下文
第一版不自动判断关键帖子，先作为开放问题讨论。
