# Search Error Taxonomy -- Interview Q&A

**Scope**: Query / Recall / Ranking

---

## Q1：搜索效果变差了，你会从哪一层开始排查？

**答：我会先从 Query → Recall → Ranking 这个顺序来排查。**

原因是：

- Query 错了，后面全错
- Recall 错了，Ranking 无法修复
- Ranking 错了，影响最可控

👉 **越靠前的错误，越致命；越靠后的错误，越可调。**

---

## Q2：怎么判断是 Query 问题还是 Recall 问题？

**答：看"相关 item 是否进入候选集"。**

- 如果相关 item **根本没被召回** → Recall 问题
- 如果相关 item **在候选里，但排得很低** → Ranking 问题
- 如果整个候选集方向就不对 → Query 理解问题

---

## Q3：什么是 Query Understanding Error？举个例子。

**答：Query 理解错误是指系统没理解用户"想找什么"。**

典型例子：

- 搜索 "apple"，用户想找品牌，系统理解成水果
- 搜索 "nike air force"，用户想找具体商品，系统当成类目浏览

这种错误会导致：

- 召回路径选错
- 排序目标不匹配

👉 后续模型再强也救不回来

---

## Q4：Query 有歧义时，系统应该怎么做？

**答：不应该强行选单一意图，而应该支持多意图或延迟决策。**

工程做法包括：

- 保留 intent 分布（而不是单标签）
- 多路 recall
- 在 ranking 阶段再融合判断

---

## Q5：什么是 Recall Error？为什么说它是不可逆的？

**答：Recall Error 指相关 item 没有进入候选集。**

不可逆的原因是：

- Ranking 只能在候选集内排序
- item 不在候选里，ranking 再好也没用

所以：

**Recall 是搜索系统的"生命线"。**

---

## Q6：常见的 Recall Error 有哪些？

**答：主要有三类：**

### 1. Coverage 不足

- Recall K 太小
- 召回路数太少

### 2. Recall-Intent 不匹配

- Brand query 却走 generic recall
- 强意图却走 popularity recall

### 3. 冷启动 / 长尾失败

- 新 item 无法被 embedding recall 覆盖

---

## Q7：如果 Recall 没问题，排序还是不好，问题可能在哪？

**答：那通常是 Ranking Error。**

常见原因：

- 特征和意图不匹配
- 优化目标（loss）不合理
- Popularity / position bias 过强

---

## Q8：Offline 指标涨了，Online 指标不涨，可能是哪一层的问题？

**答：通常是 Ranking Objective 或 Bias 问题，但也要回查 Query 和 Recall。**

可能原因包括：

- Offline loss 与 online 目标不一致
- Offline 数据分布与线上不一致
- Ranking 学到了 bias，而不是 relevance

---

## Q9：如何避免把 Recall 问题误判成 Ranking 问题？

**答：一定要先做 Recall Coverage Analysis。**

例如：

- Recall@K 是否足够
- 不同 recall source 的 overlap
- Relevant item 是否进入 candidate set

**如果 recall 就失败了，不应该调 ranking。**
