---
type: think
author: dengke
created: '2026-04-30T16:04:36+08:00'
index_state: indexed
---
# 关于员工评价体系中评分对象、mention 与 annotation 的建议

我认同这个评分体系的大方向，但建议在两个地方做调整：第一，不要把评分对象收敛到 matter.owner；第二，重新区分 mention 和 annotation，避免继续滥用 comments 这个概念。

## 1. 不要限制评分对象只能是 owner

我不建议在产品层面限制评分对象。

一个 matter 的推进过程中，团队成员会创建 think、act、verify、result、insight 等各种文件。任何人都可以对其中任何一篇文件做 annotation 评价；同一篇文件也可以被多人评价。

评价本身应当像 360 环评一样自然发生：我看到某篇文件代表的思考、行动、验收或结果做得好或不好，就可以直接在这篇文件上留下评价。系统不需要预先判断谁应该被评价，也不需要限制只能评价 matter.owner。

至于评价会不会影响其他同事参与讨论和评论的积极性，这更像是团队管理和组织文化需要自己把握的问题，不应该在产品模型里硬约束。实际上，产品层面也很难真正约束这件事。

后续 AI 做员工评分或组织复盘时，再基于这些 annotation 进行聚合分析。它需要考虑：

- 文件 creator
- 文件 owner
- 评价者身份权重
- 评价内容强弱
- 评价维度
- 该文件在 matter 时间线中的作用
- 是否有 verify / result 等硬事实支撑
- 同一事实是否被多人重复评价

这样既保留了团队评价的自由度，也避免把评分系统设计成一套复杂的权限和对象识别机制。

## 2. comments 这个概念应该拆掉

现在系统里有一点滥用 comments 这个名词。

当前所谓 comments，本质上更像是 mention 的内容。它表达的是：我在某个文件下面留了一段话，并希望某些人看到。这个能力应该叫 mention，而不是 comment。

建议把现有结构从：

```yaml
comments:
  - body: 你看看这个方案
    mentions:
      - liuyu
```

调整为：

```yaml
mentions:
  - body: 你看看这个方案
    targets:
      - liuyu
```

也就是说：

- mention 是这个功能本身。
- body 是 mention 的正文。
- targets 是被提醒的人。

mention 的主要作用是提醒、拉人参与、要求对方看一下或回应一下。它可以被 AI 作为弱信号参考，但它不应该和正式评价混在一起。

## 3. 新增 annotation，专门承载评价

annotation 是另一类能力，不应该和 mention 混在一起。

mention 是为了让别人看到、回应或参与。

annotation 是为了对某个文件代表的工作事实做评价、标注或补充判断。

例如：

```yaml
annotations:
  - type: evaluation
    target: discussions/Pivot/xxx/002_liuyu_act_2780d3.md
    creator: dengke
    body: 这个设计收敛得很好，事实层和语义层的边界清楚，适合作为后续评分系统的基础。
```

这个 annotation 的评价对象是某一篇文件。至于这个评价后续应该归因给谁，是 AI 在评分统计阶段根据文件 creator、owner、上下文和评价内容去分析的事情，不需要在 annotation 创建时强行复杂化。

## 4. annotation 不要让用户手填 weight 和 rating

我不建议让用户在 annotation 里手工填写 `weight` 或 `rating`。

`weight` 不应该由评价者自己填，而应该来自组织身份和系统配置，比如 CEO、CTO、负责人、普通成员。否则用户自己写 `weight: high` 没有治理意义，也容易被滥用。

`rating` 也不建议由人直接打分。不同人的打分尺度差异很大，上级也可能不好意思给低分，最后会变成人情分或形式分。更好的方式是：人只负责写清楚自己的真实评价，AI 再根据评价内容、评价者身份、上下文事实，推断这个评价对应的情绪、强度、维度和评分影响。

也就是说，原始 annotation 只保留人的自然语言评价：

```yaml
type: evaluation
creator: dengke
target: discussions/Pivot/xxx/002_liuyu_act_2780d3.md
body: 这个方案判断很准，提前识别了权限边界问题，后面实施也基本按这个思路走通了。
```

AI 或评分系统再生成派生解释：

```yaml
author_weight: 2.0
sentiment: positive
strength: strong
dimension: judgment
inferred_score_delta: +0.4
confidence: high
reason: 评价中明确提到“判断很准”和“后面实施走通”，属于判断质量的强正向证据。
```

这里的关键边界是：AI 推断出的 rating 或 score delta 不能冒充用户原始输入。用户原始输入永远只是他说过的话；评分、维度、强度、置信度都是系统的派生解释。

## 5. 建议形成四层结构

这样调整后，评分系统可以同时读取四层信息：

1. timeline 事实层：think、act、verify、result、insight。
2. mention 提醒层：谁在某个文件下提醒了谁、说了什么。
3. annotation 评价层：人对某个文件或某个工作事实的明确评价。
4. AI 派生评分层：系统基于事实、mention 和 annotation 生成的结构化评分结果。

这四层语义应该分开。

comments 这个名字建议逐步废弃。普通提醒和协作入口应叫 mention；正式评价和结构化标注应叫 annotation。这样语义会更清楚，也能避免把轻量提醒、普通讨论和正式评价全部混在一个字段里。

## 6. 总结建议

我建议评分系统不要在产品层面规定“只评价 owner”，而是允许团队成员对任意文件自由留下 annotation 评价。

评分系统真正要做的，是在 matter 结束后，对这些分散在文件上的评价进行聚合、归因和加权：

- 谁创建了被评价的文件
- 谁是文件 owner
- 谁做出了评价
- 评价者在组织中的权重是多少
- 评价内容指向什么维度
- 评价强度如何
- 是否有硬事实支撑
- 是否存在多人重复评价同一事实

这样既保留了 Pivot 文件流的简洁性，也让员工评价系统有足够丰富、真实、可追溯的证据来源。
