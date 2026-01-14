# Relevance vs Engagement

在搜索与推荐系统中，**排序（Ranking）的目标函数并不唯一**。

一个核心设计问题是：**我们到底在优化什么？**

常见的两类目标可以概括为：

- **Relevance（相关性）**
- **Engagement（互动/行为）**

它们看起来相似，但在建模目标、数据来源、系统风险上有本质差异。

---

## 1. 什么是 Relevance（相关性）

**Relevance** 关注的是：

**给定一个 query / 用户意图，这个 item 是否"应该被看到"？**

它强调的是 **语义和匹配正确性**。

### 常见特征

- Query-Item 的语义相似度
- 关键词匹配（BM25 / TF-IDF）
- Embedding 相似度
- 类目、属性、约束条件是否满足

### 常见训练信号

- 人工标注相关性（relevance label）
- 点击作为弱相关信号（clicked vs not clicked）
- Query-Item 正负样本对

### 常见目标

- Cross Entropy（pointwise）
- Pairwise loss（相关 > 不相关）
- NDCG@K（离线）

📌 **Relevance 的本质**：

排序是否"合理、正确、可解释"。

---

## 2. 什么是 Engagement（互动）

**Engagement** 关注的是：

**用户会不会点？会不会停留？会不会转化？**

它强调的是 **行为概率最大化**。

### 常见指标

- CTR（点击率）
- Dwell Time（停留时长）
- CVR（转化率）
- Like / Share / Purchase

### 常见特征

- 用户历史行为
- 时序特征（recency）
- Item 热度、位置偏差
- 用户画像 + 上下文

### 常见目标

- 点击概率预测（P(click | u, i, ctx)）
- Expected Engagement 最大化
- 多目标加权（CTR + CVR）

📌 **Engagement 的本质**：

排序是否"能让用户产生行为"。

---

## 3. Relevance vs Engagement 的核心差异

| **维度** | **Relevance** | **Engagement** |
|---------|--------------|---------------|
| 关注点 | 匹配是否正确 | 行为是否发生 |
| 主要风险 | 排序不准 | Clickbait / 偏置放大 |
| 数据来源 | 标注 / 规则 / 弱监督 | 用户日志 |
| 可解释性 | 强 | 相对弱 |
| 泛化能力 | 更稳定 | 易受分布影响 |

一个典型冲突是：

**"高度相关，但不一定好点"**  
**"很好点，但不一定相关"**

---

## 4. 为什么不能只优化 Engagement？

如果排序目标**只剩 CTR / Engagement**，常见问题包括：

- Clickbait 内容被过度放大
- 热门 item 持续占据前排（popularity bias）
- 新 item / 长尾 item 被压制
- 搜索场景下出现"答非所问但很吸引人"的结果

📌 在 **Search 场景** 中尤其危险，因为：

- 用户带着明确 intent
- 错配的高 CTR 结果会破坏信任

---

## 5. 工业界的常见做法：分层目标

### 5.1 Recall / 粗排：Relevance 优先

- 强约束意图匹配
- 保证结果"都说得过去"
- 宁可多一点，也不能错太多

### 5.2 精排 / 重排：Engagement 调优

- 在"相关集合"内优化点击/转化
- 使用行为特征
- 控制 bias 和 overfitting

📌 一句话总结：

**先保证"对不对"，再优化"好不好点"。**

---

## 6. 常见设计方式

### 方式一：Two-stage Objective

- Stage 1：Relevance-based filtering
- Stage 2：Engagement-based ranking

### 方式二：Multi-objective Ranking
```
score = α · relevance_score + β · engagement_score
```

权重 α / β 可根据：

- 场景（Search vs Feed）
- 流量类型
- 用户阶段动态调整

---

## 7. 本项目中的取舍

在本项目中，我：

- 使用 relevance-aware 特征确保排序基础正确性
- 在精排阶段引入行为特征进行 engagement 调优
- 通过离线指标（NDCG@K）验证相关性不退化
- 明确指出纯 engagement 优化的风险与限制
