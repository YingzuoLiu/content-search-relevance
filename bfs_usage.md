# BFS 场景题

## 场景 1：图召回（Item-Item Graph Recall）

### 📌 业务背景

我们有一个 item-item 共现图，从用户最近点击的 item 出发，做多跳召回，但需要控制 hop 数和候选规模，避免爆炸。

### 🧠 设计要点

- **BFS（按 hop 扩展）**
- **visited 去重**
- **max_hop 控制语义扩散**
- **max_candidates 控制 recall size**

### ✅ 工程版代码（Python）
```python
from collections import deque

def bfs_graph_recall(
    graph,
    seed_item,
    max_hop=2,
    max_candidates=200
):
    """
    graph: Dict[item_id, List[item_id]]
    seed_item: 起始 item
    """
    queue = deque([(seed_item, 0)])  # (item, hop)
    visited = set([seed_item])
    candidates = []
    
    while queue:
        cur_item, hop = queue.popleft()
        
        if hop >= max_hop:
            continue
        
        for nei in graph.get(cur_item, []):
            if nei in visited:
                continue
            
            visited.add(nei)
            candidates.append(nei)
            
            if len(candidates) >= max_candidates:
                return candidates
            
            queue.append((nei, hop + 1))
    
    return candidates
```

---

## 场景 2：Search 召回不足 → BFS 式兜底扩展

### 📌 业务背景

搜索结果太少时，我们需要逐步放宽条件进行兜底召回。

### 🧠 真实系统里的"BFS 层级"

- **Level 0**: 精准语义召回
- **Level 1**: 类目相似
- **Level 2**: 热门兜底

### ✅ 代码（策略 BFS）
```python
def bfs_fallback_recall(
    query,
    recall_funcs,
    target_k=100
):
    """
    recall_funcs: List[Callable], 每一层一个召回函数
    """
    results = []
    seen = set()
    
    for level, recall_fn in enumerate(recall_funcs):
        candidates = recall_fn(query)
        
        for item in candidates:
            if item in seen:
                continue
            
            seen.add(item)
            results.append(item)
            
            if len(results) >= target_k:
                return results
    
    return results
```

---

## 场景 3：Query 意图扩展（Search Query Understanding）

### 📌 业务背景

用户输入的 query 很短，需要做同义词 / 类目扩展，但不能无限扩散。

### ✅ 工程代码
```python
from collections import deque

def bfs_query_expand(
    query,
    synonym_graph,
    max_depth=2
):
    expanded = []
    queue = deque([(query, 0)])
    visited = set([query])
    
    while queue:
        cur, depth = queue.popleft()
        
        if depth >= max_depth:
            continue
        
        for w in synonym_graph.get(cur, []):
            if w in visited:
                continue
            
            visited.add(w)
            expanded.append(w)
            queue.append((w, depth + 1))
    
    return expanded
```

---

## ✅ 带 latency budget 的 BFS
```python
import time
from collections import deque

def bfs_with_latency_guard(
    graph,
    seed,
    max_hop,
    time_budget_ms=20
):
    start = time.time()
    queue = deque([(seed, 0)])
    visited = set([seed])
    results = []
    
    while queue:
        if (time.time() - start) * 1000 > time_budget_ms:
            break  # 超时降级
        
        cur, hop = queue.popleft()
        
        if hop >= max_hop:
            continue
        
        for nei in graph.get(cur, []):
            if nei in visited:
                continue
            
            visited.add(nei)
            results.append(nei)
            queue.append((nei, hop + 1))
    
    return results
```
