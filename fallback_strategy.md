# Fallback Strategy in Search & Recommendation

当主链路结果不可靠、不稳定或不可用时，Fallback（兜底）机制用于保障系统可用性、相关性下限和用户体验。

---

## 1. 为什么必须要有 Fallback？

在真实系统中，"模型永远正确"是不存在的。常见失败场景包括：

### 1.1 输入侧问题

- Query 太短 / 太怪（`"a"`, `"??"`, `"123"`）
- 新词 / 拼写错误 / 语言混杂
- 意图无法识别或置信度很低

### 1.2 召回侧问题

- Embedding 召回 TopK 数量不足
- 向量索引异常 / 超时
- 冷启动用户 / 冷启动商品

### 1.3 排序侧问题

- Ranking model score 分布异常（塌缩、全接近）
- 特征缺失（行为特征、上下文特征为空）
- 推理超时，超过 latency budget

**👉 Fallback 的本质目标：不追求最优，只保证 "不差 + 不空 + 不慢"**

---

## 2. Fallback 在 Pipeline 中的位置

典型 Search / Rec Pipeline（简化）：

```
Query
  ↓
Query Understanding
  ↓
Multi-Recall
  ↓
Ranking
  ↓
Post-processing
  ↓
Response
```

Fallback 可以出现在多个层级：

| 层级 | Fallback 作用 |
|------|---------------|
| Query | query normalize / rewrite |
| Recall | 替补召回源 |
| Ranking | 简化排序 / 规则排序 |
| Response | 热门兜底 / 空结果兜底 |

---

## 3. 常见 Fallback 策略分类

### 3.1 Recall-Level Fallback（最关键）

当主召回失败或质量不足时：

#### （1）Popularity-based Recall

- 按 点击量 / 购买量 / 曝光量
- 全局或类目内热门

**✅ 优点：**
- 稳定、可解释、几乎不失败

**❌ 缺点：**
- 个性化弱，相关性有限

#### （2）Lexical Recall（TF-IDF / BM25）

- 当 embedding 召回失败
- 回退到 词面匹配

**✅ 优点：**
- 对拼写 / 新词友好

**❌ 缺点：**
- 语义理解弱

#### （3）Rule-based Recall

- 人工规则
- 特定 query / 类目白名单

**常见于：**
- 电商类目词
- 法规 / 风控敏感词

### 3.2 Ranking-Level Fallback

当排序模型不可信时：

#### （1）Lightweight Ranking

- 简单 MLP / LR
- 特征子集

#### （2）Score Heuristic

- `final_score = w1 * relevance + w2 * popularity`
- 不依赖复杂交叉特征

### 3.3 Response-Level Fallback

极端情况兜底：

- 空结果 → 全站热门
- 空类目 → 相邻类目
- 新用户 → 新手推荐池

---

## 4. 触发 Fallback 的条件（Trigger）

Fallback 不是随便用的，必须有明确触发条件。

### 4.1 常见 Trigger Signals

| 信号 | 示例 |
|------|------|
| Recall size | `len(candidates) < K_min` |
| Score distribution | 方差过小 / 全 0 |
| Confidence | `intent_prob < threshold` |
| Latency | 超过 SLA |
| Coverage | 新用户 / 新商品 |

### 4.2 示例逻辑（伪代码）

```python
if recall_size < MIN_RECALL:
    use_popularity_recall()

elif intent_confidence < 0.3:
    use_lexical_recall()

elif ranking_timeout:
    use_lightweight_ranking()
```

---

## 5. Fallback ≠ One-shot，而是 分层兜底

真实系统中通常是 **多级 fallback**：

```
Primary embedding recall
        ↓
Secondary lexical recall
        ↓
Popularity recall
        ↓
Global hot items
```

**关键原则：**
- 能早不晚
- 越往后越稳
- 最后一定不空

---

## 6. Fallback 的评估方式

Fallback 不追求 SOTA，但也要评估：

### 6.1 Offline

- Recall@K（是否空）
- Coverage（用户 / 商品覆盖率）
- Worst-case performance

### 6.2 Online

- Empty result rate
- Bounce rate
- CTR / Dwell time 下限
