# Offline vs Online Gap（离线-线上指标差距）

## 1. 什么是 Offline / Online Gap？

**Offline–Online Gap** 指的是：

> 模型在离线评估指标（如 NDCG@K、Recall@K）上显著提升，  
> 但上线后在线指标（如 CTR、CVR、GMV、Dwell Time）没有提升，甚至下降。

这是搜索与推荐系统中**非常常见、也是最危险的问题之一**。

---

## 2. 常见离线指标 vs 在线指标

### 离线指标（Offline Metrics）
- Recall@K
- Precision@K
- NDCG@K
- MAP
- AUC（排序对）

特点：
- 基于 **历史日志**
- 可复现、可对比
- 低成本、迭代快
- 但 **不等于真实用户行为**

---

### 在线指标（Online Metrics）
- CTR（Click-Through Rate）
- CVR（Conversion Rate）
- Dwell Time / Watch Time
- GMV / Revenue
- Session Length

特点：
- 真实用户反馈
- 高噪声
- 成本高（A/B Test）
- 受 UI、流量、策略强烈影响

---

## 3. 为什么 Offline 指标涨了，Online 不涨？

### 3.1 数据分布不一致（Distribution Shift）

- 离线评估使用的是 **历史曝光数据**
- 线上模型会 **改变曝光分布**
- 新模型可能把 item 推给「没见过它的用户」

结果：
> 离线看起来“相关”，  
> 但线上用户并不点。

---

### 3.2 离线标签 ≠ 真实目标

常见情况：
- 离线用 click 作为正样本
- 实际业务更关心：
  - 停留时间
  - 转化
  - 用户满意度

👉 **优化目标不一致**

---

### 3.3 Position Bias / Exposure Bias

- 离线数据中：
  - 前排 item 更容易被点击
- 模型可能学到：
  > “排前的就是好”

而不是：
> “真的相关才好”

---

### 3.4 Offline Metric 本身不敏感

例如：
- Recall@50 变化不大
- 但前 3 位顺序变化很大

👉 用户只看 Top 3，但 Recall@50 看不出来

---

### 3.5 系统层因素未纳入离线评估

离线通常 **忽略**：
- latency 变慢
- fallback 触发率上升
- 多路召回去重失败
- 缓存 miss 增加

这些都会 **直接影响线上指标**

---

## 4. 如何缩小 Offline–Online Gap？

### 4.1 对齐指标（Metric Alignment）

| 离线 | 对应线上 |
|----|----|
| NDCG@K | CTR / Engagement |
| Weighted NDCG | Revenue / GMV |
| Session-based metric | Session CTR / Watch Time |

原则：
> 离线指标应尽量模拟用户真实目标

---

### 4.2 只关注 Top-K（尤其是 Top-3 / Top-5）

- 用户几乎不看后排
- 离线评估应：
  - 强调 top-heavy metric
  - 或使用 position-weighted NDCG

---

### 4.3 使用更真实的负样本

避免：
- 随机负样本

尝试：
- 曝光未点击（impression-level negatives）
- hard negatives
- 同 query / session 下的对比样本

---

### 4.4 离线 + Online Shadow / Small A/B

- Shadow test（不影响排序，仅记录）
- 小流量灰度
- 逐步放量

---

### 4.5 将系统约束纳入评估

离线实验时同步监控：
- latency
- recall size
- fallback rate
- cache hit rate

👉 **模型好 ≠ 系统可用**

---

## 5. 在个人项目中的实践方式

在本项目中，我：

- 使用 **Random / Popularity-based** 作为 sanity check
- 要求模型在多个 K 值下 **稳定优于 baseline**
- 明确指出：
  > 离线评估无法完全代表真实用户反馈
- 将 Offline 指标作为 **是否值得上线测试的筛选器**
- 强调最终结论需要 Online A/B Test 验证

---

## 6. 面试常见追问 & 回答要点

### Q1：Offline 指标涨了但 Online 不涨，你怎么办？
- 排查数据分布
- 检查 metric alignment
- 看 top-K 变化
- 小流量 A/B 验证

---

### Q2：为什么不用 Accuracy？
- 排序问题类别极度不平衡
- Accuracy 不反映排序质量
- 与用户体验弱相关

---

### Q3：Offline Evaluation 的最大局限？
- 基于历史曝光
- 无法模拟用户探索行为
- 容易过拟合日志偏差

---



