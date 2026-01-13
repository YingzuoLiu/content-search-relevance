# Boyer-Moore（多数投票）算法总结

## 1️⃣ 问题本质

Boyer-Moore 解决的不是"统计最多"，而是这一类问题：

**当一个主导信号一定存在时，能否在只扫一遍、几乎不存数据的情况下把它找出来？**

关键词只有三个：
- **主导信号**
- **对抗 / 抵消**
- **streaming**

---

## 2️⃣ 核心直觉

用一个计数器，让相同的信号互相累加，冲突的信号互相抵消，最后留下的就是主导信号。

如果某个元素出现次数 > 其他所有元素加起来，那么不管怎么两两抵消，它都不可能被完全抵消掉。

**这不是技巧，而是数量上的必然性。**

- **每一次抵消：**
  - 主导元素 −1
  - 非主导元素 −1

- **但主导元素一开始就更多 👉 所以最后一定还能剩下**

---

## 4️⃣ 最小可记忆代码结构

- `count == 0` → 换候选
- 相同 → +1
- 不同 → -1

**对应代码：**
```python
candidate = None
count = 0

for x in nums:
    if count == 0:
        candidate = x
    count += 1 if x == candidate else -1
```

---

## 5️⃣ 隐含前提

Boyer-Moore 成立的前提只有一个：

**主导信号一定存在（出现次数 > n/2）**

如果这个前提不成立：
- 算法仍会返回一个 candidate
- 但这个 candidate 可能是错的

👉 所以它不是"万能算法"，而是有使用边界的工具

---

## 6️⃣ 工程中的真实意义

在真实系统中，它通常不会以"多数元素"的形式出现，而是：

**常见工程语义**
- session 主导意图
- 用户行为的正 / 负倾向
- 实时流中的主流信号
- 噪声中的 dominant pattern

**它的价值在于：**
- O(1) 空间
- 单次扫描
- 自动消噪

---

## 7️⃣ 主导信号不存在时的工程兜底（含代码）

我们假设输入是一个**行为 / 信号序列**：
```python
actions = ["click", "skip", "click", "skip", "click"]
```

### 7.1 方案一：Boyer-Moore + 二次验证

**代码**
```python
def decide(actions):
    # 1) Boyer-Moore 找候选
    cand = None
    count = 0
    
    for a in actions:
        if count == 0:
            cand = a
        count += 1 if a == cand else -1
    
    # 2) 二次验证：是否真的"足够强"
    freq = sum(1 for a in actions if a == cand)
    
    if freq > len(actions) / 2:
        return ("use_dominant", cand)  # 走主路径
    else:
        return ("fallback", None)  # 走兜底
```

---

### 7.2 方案二：置信度阈值（线上）

不一定非要 > 50%，但**太弱的信号不能信**

**代码**
```python
def decide(actions, threshold=0.2):
    cand = None
    count = 0
    
    for a in actions:
        if count == 0:
            cand = a
        count += 1 if a == cand else -1
    
    confidence = abs(count) / len(actions)
    
    if confidence >= threshold:
        return ("use_dominant", cand, confidence)
    else:
        return ("fallback", None, confidence)
```

---

### 7.3 保留多候选

不强行压成一个信号，**交给 ranking 再融合**

**代码示例（简单版）**
```python
from collections import Counter

def multi_signal(actions, min_ratio=0.3):
    freq = Counter(actions)
    total = len(actions)
    
    return {
        k: v / total
        for k, v in freq.items()
        if v / total >= min_ratio
    }
```

**返回示例**
```python
{"click": 0.5, "skip": 0.4}
```

👉 常见于：
- multi-head ranking
- intent-aware models

---

### 7.4 多意图概率提取
```python
from collections import Counter

def extract_intent_probs(actions, intents, min_ratio=0.2):
    """
    actions: 例如 ["brand", "category", "brand", "brand", "other"]
    intents: 例如 ["brand", "category", "deal", "other"]
    返回: dict {intent: prob}，只保留占比>=min_ratio 的意图
    """
    cnt = Counter(actions)
    total = len(actions)
    
    probs = {k: cnt.get(k, 0) / total for k in intents}
    probs = {k: v for k, v in probs.items() if v >= min_ratio}
    
    # 可选：归一化（只在你过滤后想让权重和=1时需要）
    s = sum(probs.values())
    if s > 0:
        probs = {k: v / s for k, v in probs.items()}
    
    return probs
```

---

### 7.5 Multi-Head Ranker 实现
```python
import torch
import torch.nn as nn

class MultiHeadRanker(nn.Module):
    def __init__(self, in_dim: int, hidden_dim: int, intents):
        super().__init__()
        self.intents = list(intents)
        
        # shared trunk（共享表征）
        self.trunk = nn.Sequential(
            nn.Linear(in_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
        )
        
        # 每个 intent 一个 head（最简单：小 MLP / 线性层）
        self.heads = nn.ModuleDict({
            intent: nn.Linear(hidden_dim, 1) for intent in self.intents
        })
    
    def forward(self, x, intent_probs=None, hard_intent=None):
        """
        x: [B, in_dim]
        intent_probs: dict {intent: weight}，来自方案四
        hard_intent: str 或 None（当意图非常确定时直接走对应 head）
        返回: scores [B]
        """
        h = self.trunk(x)  # [B, hidden_dim]
        
        # 1) hard route：意图很确定，直接走一个 head
        if hard_intent is not None:
            return self.heads[hard_intent](h).squeeze(-1)
        
        # 2) soft route：意图不确定，用 intent_probs 做加权融合
        # score = Σ w_i * head_i(h)
        if intent_probs is None or len(intent_probs) == 0:
            # 工程兜底：没有意图就走 "other" 或均匀平均
            # 这里示例：均匀平均
            scores = [self.heads[i](h) for i in self.intents]  # list of [B,1]
            return torch.mean(torch.cat(scores, dim=1), dim=1)
        
        total_w = 0.0
        fused = 0.0
        
        for intent, w in intent_probs.items():
            if intent in self.heads:
                fused = fused + w * self.heads[intent](h)  # [B,1]
                total_w += w
        
        # 如果 intent_probs 里都是未知 key，仍要兜底
        if total_w == 0:
            return self.heads["other"](h).squeeze(-1)
        
        return (fused / total_w).squeeze(-1)
```

---

### 7.6 Intent Routing 策略
```python
def route_intent(intent_probs, hard_threshold=0.7):
    """
    如果某个意图权重大到足够确定 → hard route
    否则 → soft route（加权融合）
    """
    if not intent_probs:
        return None  # 走 soft route 的兜底逻辑
    
    best_intent, best_w = max(intent_probs.items(), key=lambda kv: kv[1])
    
    if best_w >= hard_threshold:
        return best_intent  # hard route
    
    return None
```

---

### 7.7 完整使用示例
```python
if __name__ == "__main__":
    intents = ["brand", "category", "deal", "other"]
    
    # 方案四：多信号输入（真实系统里来自 session 行为、query intent 识别等）
    actions = ["brand", "category", "brand", "brand", "other"]
    intent_probs = extract_intent_probs(actions, intents=intents, min_ratio=0.2)
    print("intent_probs:", intent_probs)
    
    # 模型输入特征（这里随便造一个 batch）
    B, in_dim = 4, 16
    x = torch.randn(B, in_dim)
    
    model = MultiHeadRanker(in_dim=in_dim, hidden_dim=32, intents=intents)
    hard_intent = route_intent(intent_probs, hard_threshold=0.7)
    
    # 前向：要么 hard route，要么 soft route（用 intent_probs 加权）
    scores = model(x, intent_probs=intent_probs, hard_intent=hard_intent)
    
    print("hard_intent:", hard_intent)
    print("scores:", scores)
```
