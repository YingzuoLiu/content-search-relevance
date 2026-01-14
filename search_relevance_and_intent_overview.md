# Search Relevance & Intent Overview

**From Matching to Understanding**

本文件系统性整理了搜索 / 推荐系统中 **Relevance（相关性）** 与 **Intent（意图）** 的核心概念、阶段差异以及工程实践中的常见问题，作为 ranking objective 之前的概念铺垫。

---

## 1️⃣ Relevance 是什么？

**Relevance（相关性）** 描述的是：

*Query 与 Item 之间"匹配得有多好"*

它通常是一个 **连续值（score）**，用于排序。

在工程实践中，相关性并非单一信号，而是由多个层面共同构成。

### 三种常见相关性来源

- **Lexical relevance**：词面是否匹配
- **Semantic relevance**：语义是否相似
- **Behavioral relevance**：历史用户行为是否验证过这种匹配

👉 相关性回答的是：

**"这个结果和 query 有多相关？"**

---

## 2️⃣ Intent 是什么？

**Intent（用户意图）** 描述的是：

*用户当前想"做什么"*

Intent 更像是一个 **离散的、结构化假设**，用于控制系统决策。

### 常见 intent 类型包括：

- Navigational（找某个确定对象）
- Transactional（想买 / 下单）
- Informational（想了解）
- Comparison（对比）
- Exploration（浏览 / 发现）

👉 Intent 回答的是：

**"用户希望看到哪一类结果？"**

---

## 3️⃣ Relevance vs Intent（核心区别）

| **维度** | **Relevance** | **Intent** |
|---------|--------------|-----------|
| 关注点 | 匹配强度 | 用户目标 |
| 形式 | 连续分数 | 离散类别 / 结构 |
| 作用 | 排序 | 路由 / 约束 |
| 问题 | 哪个更相关 | 哪类结果合理 |

**相关 ≠ 合适**

一个 item 可以在语义上很相关，但并不符合当前用户意图。

---

## 4️⃣ Relevance 在 Recall vs Ranking 中的不同含义

### Recall 阶段的 Relevance

Recall 阶段关注的是：

**"这个 item 有没有资格进入候选集？"**

**特征：**

- 潜在相关（weak relevance）
- Boolean 或粗粒度 score
- 目标是 **高召回，不漏候选**

**常用信号：**

- Lexical match
- Embedding similarity
- Intent routing / category filtering

### Ranking 阶段的 Relevance

Ranking 阶段关注的是：

**"哪个 item 更值得排在前面？"**

**特征：**

- 强相关（fine-grained relevance）
- 连续、可排序的 score
- 目标是 **高 precision**

**常用信号：**

- Semantic matching features
- Behavioral signals（CTR / CVR）
- Context & cross features

### 一句话对照

**Recall 用 relevance 做筛子，Ranking 用 relevance 做尺子**

---

## 5️⃣ Intent Taxonomy（意图分类怎么设计）

### Intent 分类的本质

Intent taxonomy 不是纯 NLP 问题，而是 **系统设计问题**。

一个 intent 类别存在的前提是：

- 它能改变系统行为（recall / ranking / UI）
- 它是可执行、可验证的

### 三条设计原则

1. **必须影响系统决策**  
   不影响系统行为的 intent 没有工程价值。

2. **可区分、可落地**  
   宁可语义粗一点，也要决策明确。

3. **少而稳**  
   工业系统中通常 3-6 类 intent 就足够。

### 常见 taxonomy（示例）

| **Intent** | **典型 Query** |
|-----------|---------------|
| Buy | "iphone 15 pro" |
| Browse | "summer dress" |
| Compare | "airpods vs buds" |
| Service | "iphone repair" |

---

## 6️⃣ Intent Errors（意图错误如何定位）

### 为什么 Intent 错误很致命？

- Intent 位于 pipeline 前端
- 会影响 recall 路由和 ranking 策略
- 下游模型很难完全纠正

### 常见 Intent 错误类型

1. **Transactional ↔ Informational 混淆**  
   → CTR 尚可，CVR 明显下降

2. **Navigational 识别失败**  
   → 用户频繁 refine query

3. **Exploration 被误判为强购买**  
   → 多样性下降，session 变短

### Intent 错误如何被发现？

- Query-level 指标异常（CTR / CVR / dwell time 不一致）
- Query reformulation 频繁
- Intent bucket 指标对比异常

### Intent 错误如何定位？

1. 固定 query，人工 review 结果
2. 强制切换 intent，对比结果差异
3. 追踪 intent → recall → ranking 的影响路径

### 工程缓解策略

- Intent 决策尽量保守
- 设置 fallback intent
- Intent 作为 soft feature，而不是 hard gate
- 给 ranking 留纠错空间

---

## 7️⃣ 总结（为 Ranking Objective 做铺垫）

- **Relevance** 决定"排得好不好"
- **Intent** 决定"排得对不对"
- Recall 和 Ranking 使用的是 **不同语义密度的 relevance**
- Intent taxonomy 要为系统服务，而不是追求语义完美

---

## 8️⃣ 明天要写的内容（承接）

**ranking_objective.md**

- relevance vs engagement

即：

- 为什么不能只优化 relevance
- engagement 信号如何进入 ranking objective
- relevance 与 engagement 的张力与取舍
