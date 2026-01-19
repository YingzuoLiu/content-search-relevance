# Algorithm → Search / Recommendation Mapping

> This document maps classic algorithmic patterns to real-world Search & Recommendation system scenarios.
> The goal is to demonstrate **system-level thinking**, not LeetCode tricks.

---

## 1. BFS → Multi-Recall Expansion

### Algorithm Pattern
- Breadth-First Search (Queue + Visited)
- Layer-by-layer expansion
- Multiple starting points supported

### Typical Problems
- Shortest path
- Minimum steps
- Island / connected components

### Search / Rec Mapping
- Multi-source recall expansion
- User interest propagation
- Cold-start item discovery

### Example (Pseudo)
```python
queue = deque(seed_items)
visited = set(seed_items)

while queue:
    item = queue.popleft()
    for neighbor in related_items(item):
        if neighbor not in visited:
            visited.add(neighbor)
            queue.append(neighbor)
```

### Why BFS Fits
- Guarantees closer / more relevant candidates first
- Easy to control depth (recall radius)
- Natural fit for graph-based recall

---

## 2. Heap / TopK → Candidate Truncation

### Algorithm Pattern
- Min-heap of fixed size K
- Push → Pop when overflow

### Typical Problems
- K-th largest element
- Top-K frequent items
- Streaming top-K

### Search / Rec Mapping
- Recall → Rank candidate cutoff
- Latency control
- Multi-channel recall merge

### Example (Pseudo)
```python
heap = []
for item, score in candidates:
    heappush(heap, (score, item))
    if len(heap) > K:
        heappop(heap)
```

### Why Heap Fits
- Avoids full sorting
- Stable under streaming input
- Clear latency bound

---

## 3. Sliding Window → Session & Temporal Modeling

### Algorithm Pattern
- Two pointers (left / right)
- Incremental state update
- Shrink window when invalid

### Typical Problems
- Longest valid subarray
- At most K constraint
- Continuous segments

### Search / Rec Mapping
- Session-based behavior modeling
- Recent N actions
- Frequency / freshness constraints

### Example (Pseudo)
```python
left = 0
state = 0
for right in range(len(events)):
    state += events[right]
    while state > limit:
        state -= events[left]
        left += 1
```

### Why Sliding Window Fits
- Models temporal continuity
- O(n) instead of re-computation
- Easy to extend with decay or weights

---

## 4. Set → Deduplication & Exposure Control

### Algorithm Pattern
- Hash-based existence check
- O(1) insert / lookup

### Typical Problems
- Remove duplicates
- Visited tracking
- Intersection

### Search / Rec Mapping
- Dedup recall results
- Prevent repeated exposure
- Track visited nodes in recall graph

### Example (Pseudo)
```python
seen = set()
for item in candidates:
    if item not in seen:
        seen.add(item)
        output.append(item)
```

### Why Set Fits
- Minimal memory overhead
- Clear semantic meaning: seen vs unseen

---

## 5. Dict → Feature & State Mapping

### Algorithm Pattern
- Key → Value mapping
- Fast access to stored state

### Typical Problems
- Two Sum
- Frequency counting
- Index lookup

### Search / Rec Mapping
- User → embedding
- Item → features
- Item → score / metadata

### Example (Pseudo)
```python
item_score = {}
item_score[item_id] = score
```

### Why Dict Fits
- Central storage for system state
- Enables fast feature joins
