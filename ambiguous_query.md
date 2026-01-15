# Ambiguous Query 处理指南

## 1. Ambiguous Query 定义

用户输入太短/太泛/有多种可能意图，系统在"到底要搜什么/要哪个类目/要哪个实体"上不确定。

**典型特征：**
- 多义词
- 缺上下文
- 缺约束
- 口语缩写
- 拼写变体

---

## 2. 在搜索/推荐中造成的问题（按 Pipeline）

### 2.1 Query Understanding（最先爆）

- **意图分类不确定：** 同一个词既可能是商品、品牌、问题、地点
- **实体识别/消歧失败：** 同名/同缩写（Apple：水果 or 公司）
- **Query rewrite 误改：** 把用户真正想要的词替换错了（slang/拼写纠正过头）

### 2.2 Recall（召回阶段）

- **召回太发散：** 为了不漏，拉一堆不相关候选 → 噪声大
- **召回太保守：** 只召回一个方向 → 漏掉用户真实意图（Recall drop）

### 2.3 Ranking（排序阶段）

- **特征冲突：** 文本相关性 vs 热度/CTR 特征把结果"带歪"
- **短 query 下行为特征更强：** 容易被 popularity bias 主导，相关性反而差

---

## 3. 常见类型（Taxonomy）

1. **Lexical ambiguity（词面多义）**
   - 例：apple, jaguar

2. **Entity ambiguity（实体同名/缩写）**
   - 例：PS5（主机/游戏/维修/二手）

3. **Intent ambiguity（意图不明）**
   - 例："iPhone 15" → 买？比价？评测？维修？

4. **Constraint missing（缺约束）**
   - 例："laptop" 没预算/尺寸/用途

5. **Session-dependent（依赖上下文）**
   - 例：上一个 query 是 "running shoes"，下一个只打 "nike"

---

## 4. 工业界处理方案（从轻到重）

### 4.1 低成本稳健方案（优先推荐）

#### (A) Query Normalization（归一化）
- lower/trim/拼写纠正/同义词映射
- **保护措施：** 别改实体名、别改品牌关键字

#### (B) Multi-recall + Dedup（多路召回兜底）
- **Lexical**（BM25/倒排） + **Semantic**（embedding） + **Popularity**（热度兜底）
- merge + 去重 + 限制每路配额（避免全被热门吞了）

#### (C) Intent-aware Ranking（用"意图分布"当条件特征）
模型不需要100%知道意图，只要输出一个 `intent_probs`，在 rank 里当 feature：
- `P(navigate)` 高 → 更偏实体直达
- `P(transaction)` 高 → 更偏可买、价格、库存、配送

#### (D) Guardrail / Fallback（降级策略）
当"模型不确定"时：
- 上调 lexical 权重（避免语义乱飞）
- 或切到 popularity / 类目榜单（至少不空、不离谱）

### 4.2 高收益但成本更高的方案

#### (E) Ask-to-clarify（澄清）
**不是所有场景都问一句"你想要哪种？"**

只有在以下情况才问：
- 不确定性高
- 错的代价大（比如医疗、金融、投诉）
- 用户愿意交互

#### (F) LLM Query Understanding（LLM 做理解，不直接做排序）
用 LLM 产出结构化结果：
- intent
- 实体
- 属性（预算/尺寸/品牌）
- rewrite candidates

**重点：** 成本控制（只对 top ambiguous 的 query 或低置信触发）

---

## 5. 判断 Query 是否 Ambiguous（可量化的触发信号）

- **Query 很短：** 1-2 tokens，且不是强实体（如"facebook login"反而不模糊）
- **Intent classifier 低置信：** max(intent_probs) 很低
- **Recall 分布太分散：** 候选来自很多无关类目、相似度分布很平
- **用户行为信号：**
  - 高曝光低点击、快速返回（pogosticking）
  - reformulation（1分钟内连续改 query）

---

## 6. 错误分析方法

### 6.1 从日志里捞出"模糊 query 集合"

- 短 query + reformulation 多
- CTR 低 + dwell time 低
- top results 类目分散

### 6.2 给每条打标签

| 维度 | 内容 |
|------|------|
| Ambiguity 类型 | lexical/entity/intent/缺约束/依赖上下文 |
| 失败阶段 | understanding/recall/rank |
| 最小改法 | 归一化？加一条召回路？加 guardrail？ |

---

## 7. 面试常见问题 & 回答要点

### Q1：为什么 ambiguous query 不能只靠 embedding？

**A：** 短 query 语义信息不足，embedding 容易"随便贴近"，会把热门/相似词带歪；需要 lexical + 行为 + 多路召回兜底，且用 guardrail 控制不确定性。

### Q2：你怎么做 fallback？

**A：** 用置信度触发：intent 低置信或 recall 分布异常时，上调 lexical、限制语义召回配额、必要时回退到 popularity/类目榜单，保证结果稳定且可解释。

### Q3：离线指标涨了线上不涨，和 ambiguous query 有关吗？

**A：** 可能。离线数据常被 head query 主导，ambiguous/head 上的 popularity bias 会"虚假提升"；线上长尾/模糊 query 多，错误会集中暴露。要分桶评估（short queries / high reformulation bucket）看改善是否真实。
