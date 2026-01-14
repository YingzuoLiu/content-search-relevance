# Mapping Errors to Concrete Actions in Search & Recommendation Systems

In real-world search and recommendation systems, errors are not just evaluation results — they are signals that indicate which part of the pipeline needs to be fixed.

This document summarizes a practical error → action mapping used in my projects.

---

## 1️⃣ Recall Miss → Fix Recall, Not Ranking

### ❌ Error Signal
- Top-K results are completely irrelevant
- Recall@K is very low
- Relevant items never appear in candidates

### 🔍 Root Cause
- The correct items are not retrieved at all
- Ranking models cannot recover missing candidates

### ✅ Action
- Add multi-path recall (keyword / embedding / popularity)
- Increase recall size K
- Apply query normalization or rewrite
- Add rule-based recall for sparse or cold scenarios

**Error → Recall missing → Improve recall coverage**

---

## 2️⃣ Ranking Improvements Don't Help → Recall Bottleneck

### ❌ Error Signal
- Ranking model improves offline
- User still cannot find desired items

### 🔍 Root Cause
- Recall coverage is insufficient
- Ranking is applied on a weak candidate set

### ✅ Action
- Expand recall diversity
- Introduce long-tail or category-based recall
- Apply fallback recall strategies

**Principle: Recall sets the upper bound; ranking only reorders candidates.**

---

## 3️⃣ Unstable Ranking → Feature or Objective Misalignment

### ❌ Error Signal
- Similar items are ranked inconsistently
- NDCG fluctuates heavily between runs

### 🔍 Root Cause
- Features lack discriminative power
- Optimization target does not reflect ranking quality

### ✅ Action
- Add comparative or pairwise features
- Introduce recency / interaction signals
- Revisit ranking objective (relevance vs engagement)

---

## 4️⃣ Popularity Bias → De-bias Instead of Manual Tuning

### ❌ Error Signal
- Old or popular items dominate top positions
- New or long-tail items receive little exposure

### 🔍 Root Cause
- Training data and objectives favor frequent items
- Model overfits popularity signals

### ✅ Action
- Apply popularity de-biasing
- Introduce time decay features
- Separate ranking buckets for new vs old items

---

## 5️⃣ Cold-Start Failure → Representation Issue

### ❌ Error Signal
- New items or users receive near-zero exposure
- CTR is extremely low for cold-start cases

### 🔍 Root Cause
- Behavior-based features are unavailable
- Item representations are incomplete

### ✅ Action
- Introduce content-based features (text / image)
- Use multimodal embeddings
- Add cold-start-specific branches or gates

**Cold-start error → Representation fix**

---

## 6️⃣ Query Understanding Errors → Fix Query, Not Rank

### ❌ Error Signal
- Model misunderstands user intent
- Synonyms, typos, or colloquial queries fail

### 🔍 Root Cause
- Query and item representations are misaligned
- Input queries are noisy or ambiguous

### ✅ Action
- Apply query normalization
- Add synonym expansion or rewrite
- Use lightweight LLM-assisted rewrite (guarded)

---

## 7️⃣ Offline Improves but Online Doesn't → Metric Misalignment

### ❌ Error Signal
- Offline NDCG / Recall improves
- Online CTR or satisfaction remains flat

### 🔍 Root Cause
- Offline metrics measure the wrong objective
- Evaluation focuses on positions users rarely see

### ✅ Action (Re-align Offline Metrics)
- Replace generic metrics (AUC / Accuracy) with top-K metrics
- Set K to realistic user-visible positions
- Redefine labels to better reflect user satisfaction
- Bucket evaluation by query frequency (head vs tail)

**Offline metric ↑ ≠ Business impact ↑**

---

## 8️⃣ Head-Dominated Improvements → Bucketed Evaluation

### ❌ Error Signal
- Overall metrics improve
- Long-tail or rare queries remain poor

### 🔍 Root Cause
- Metrics are dominated by high-frequency queries
- Long-tail failures are hidden by averages

### ✅ Action
- Bucket queries by historical frequency:
  - Head queries: high-frequency
  - Tail queries: sparse / long-tail
- Evaluate ranking quality per bucket

**Bucketed evaluation prevents misleading averages.**

---

## 9️⃣ High Latency → Architecture-Level Fix

### ❌ Error Signal
- P99 latency exceeds budget
- Ranking stage slows down the entire pipeline

### 🔍 Root Cause
- Model complexity mismatches real-time constraints

### ✅ Action
- Split coarse ranking and fine ranking
- Use ANN / caching
- Apply model distillation or degradation strategies

---

## 🔟 Summary: Error → Action Mapping

| Error Type | Wrong Reaction | Correct Action |
|------------|----------------|----------------|
| **Recall miss** | Tune rank harder | Expand recall |
| **Ranking noise** | Increase model size | Improve features |
| **Cold-start failure** | Collect more logs | Fix representation |
| **Popularity bias** | Manual reweighting | De-bias design |
| **Offline ≠ Online** | Ignore metrics | Re-align metrics |
| **Latency issues** | Optimize code only | Redesign pipeline |
