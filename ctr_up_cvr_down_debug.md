# CTR ↑ but CVR ↓：如何排查 + 归因分布怎么做（Last-Click / Last-Touch）

**目标：** 当线上出现 CTR 上升但 CVR 下降 的现象时，用一套稳定的方法判断：是 归因口径/延迟导致的假跌，还是 点击质量真的变差（目标错位）。

---

## 1. 指标拆解：先别只盯 CVR

### 1.1 基础指标定义

- **CTR** = Click / Impression
- **CVR** = Order / Click
- **CTCVR** = Order / Impression = CTR × CVR
- **GMV/Imp** = GMV / Impression

### 1.2 为什么要看 CTCVR 和 GMV/Imp？

如果出现：
- CTR ↑，CVR ↓

那你要确认：
- CTCVR 是涨还是跌？
- GMV/Imp 是涨还是跌？

因为 CVR 可能被 "归因迁移、延迟回传、口径差异" 影响而假跌。

---

## 2. 常见根因（高频）

### 2.1 点击分布变了（Click Distribution Shift）

CTR 涨可能意味着：引入了一批 "更容易点击但更不容易购买" 的流量，例如：

- 信息型点击（评测/对比/壁纸）
- 标题党/封面党
- 低价配件吸引点击但不买主品

**结果：**
- CTR ↑（更多人点）
- CVR ↓（点击质量变"水"）

### 2.2 选择偏差（Selection Bias）

CVR 只能在 "点击过的人群" 上被观察与训练：
- 你永远看不到「没点的人是否会买」
- 所以 CVR 的训练/统计天然存在选择偏差。

当 CTR 策略变化时，点击人群发生变化，会导致 CVR 不稳定。

### 2.3 目标错位（Objective Misalignment）

如果排序只优化 CTR：
- 模型会学会"最大化点击"
- 但业务最终要的是"成交 / GMV"

**典型后果：**
- 排上来高点击但低成交的 item
- CTR 漂亮但 CVR 下滑

---

## 3. 排查流程（推荐顺序）

### Step 0：先排口径/延迟问题（Sanity Check）

CVR 容易被误伤的原因：
- 转化窗口不一致（click 后 24h/7d）
- 归因规则变化（last-click / last-touch）
- 转化延迟回传（T+0 数据不成熟）

**✅ 建议做法：**
- 统一归因窗口（例如 click 后 24h）
- 看成熟期数据（T+1/T+2）

### Step 1：看 CTCVR / GMV 是否也下降

- 若 CTCVR 也跌：整体成交变差（严重）
- 若 CTCVR 持平/涨：可能只是 CVR 口径/归因迁移导致的"假跌"

### Step 2：看点击后漏斗（判断点击质量）

用更短链路的 proxy 做 sanity check：
- **ATC rate** = AddToCart / Click
- **Checkout rate** = Checkout / Click
- **Pay rate** = PaySuccess / Click (≈ CVR)

**判断逻辑：**
- CTR ↑ + ATC/Click ↓ → 点击质量变差（引入水点击）
- CTR ↑ + ATC/Click 稳 + CVR ↓ → 可能是支付/库存/归因/延迟问题

### Step 3：做分桶定位（找问题集中在哪）

**常用分桶维度：**
- Query intent（购买型 vs 浏览型）
- 新/老用户
- 新/老商品
- 价格带（低价/高价）
- 排序位置（Top1-3 vs 4-10）
- 类目（品类结构变化）

**目标：** 定位 CVR 下跌是否集中在特定桶。

---

## 4. 归因分布怎么做：Last-Click Attribution

### 4.1 Last-Click Attribution 定义

对每个订单 `order_id`：
1. 回溯订单前 `lookback_window`（例如 7 天）内用户的 click
2. 取时间最近的一次 click（last click）
3. 这条 click 的 channel = 该订单归因渠道

**常见 channel：**
- Search
- Feed/Rec
- Ads
- Cart
- Favorite
- Direct/Other

### 4.2 为什么要看"归因分布迁移"？

CTR ↑ 但 CVR ↓ 可能是因为：
- 用户在 Feed 点击种草 → 最后去 Search 下单
- 订单的 last-click 归因跑到 Search，导致 Feed CVR 看起来下滑。

---

## 5. A/B 对照：归因分布对比表模板

### 5.1 Orders 口径（Last-Click）

| channel | A_orders | A_share | B_orders | B_share | Δshare (B-A) | comment |
|---------|----------|---------|----------|---------|--------------|---------|
| Search | 52,100 | 52.1% | 60,300 | 60.3% | +8.2pp | 订单更多归因到 Search |
| Feed/Rec | 38,500 | 38.5% | 29,700 | 29.7% | -8.8pp | Feed 侧 CVR 可能"假跌" |
| Ads | 5,200 | 5.2% | 5,000 | 5.0% | -0.2pp | 稳定 |
| Cart | 2,900 | 2.9% | 3,600 | 3.6% | +0.7pp | 更多从购物车完成 |
| Favorite | 1,300 | 1.3% | 1,400 | 1.4% | +0.1pp | 稳定 |

**✅ 重点看：** Δshare 是否明显迁移（Feed → Search / Cart）

### 5.2 GMV 口径（更接近业务）

| channel | A_gmv | A_gmv_share | B_gmv | B_gmv_share | Δgmv_share | comment |
|---------|-------|-------------|-------|-------------|------------|---------|
| Search | 8.2M | 55% | 9.6M | 62% | +7pp | 高客单归因上升 |
| Feed/Rec | 5.8M | 39% | 4.7M | 30% | -9pp | Feed 贡献降低/被归走 |
| Ads | 0.9M | 6% | 1.1M | 8% | +2pp | 需进一步核查 |

---

## 6. SQL 模板：Last-Click Attribution（可直接改）

### 6.1 数据表假设

**orders**
- order_id
- user_id
- order_time
- gmv
- exp_id
- variant（A/B）

**events**
- user_id
- event_time
- event_type（click / impression）
- channel
- exp_id
- variant

### 6.2 Step1：订单前7天 click join

```sql
WITH order_clicks AS (
  SELECT
    o.order_id,
    o.user_id,
    o.order_time,
    o.gmv,
    o.exp_id,
    o.variant,

    e.channel AS click_channel,
    e.event_time AS click_time
  FROM orders o
  JOIN events e
    ON o.user_id = e.user_id
   AND o.exp_id = e.exp_id
   AND o.variant = e.variant
   AND e.event_type = 'click'
   AND e.event_time <= o.order_time
   AND e.event_time >= o.order_time - INTERVAL '7' DAY
)
```

### 6.3 Step2：每个订单取 last click

```sql
, last_click_per_order AS (
  SELECT *
  FROM (
    SELECT
      order_id,
      user_id,
      order_time,
      gmv,
      exp_id,
      variant,
      click_channel,
      click_time,
      ROW_NUMBER() OVER (
        PARTITION BY order_id
        ORDER BY click_time DESC
      ) AS rn
    FROM order_clicks
  ) t
  WHERE rn = 1
)
```

### 6.4 Step3：按 channel 统计 Orders / GMV / Share

```sql
, channel_stats AS (
  SELECT
    exp_id,
    variant,
    click_channel AS channel,
    COUNT(DISTINCT order_id) AS orders,
    SUM(gmv) AS gmv
  FROM last_click_per_order
  GROUP BY exp_id, variant, click_channel
)
SELECT
  exp_id,
  variant,
  channel,
  orders,
  gmv,
  orders * 1.0 / SUM(orders) OVER (PARTITION BY exp_id, variant) AS order_share,
  gmv * 1.0 / SUM(gmv) OVER (PARTITION BY exp_id, variant) AS gmv_share
FROM channel_stats
ORDER BY exp_id, variant, orders DESC;
```

### 6.5 可选：把 A/B 拼成一张带 Δshare 的表

```sql
WITH base AS (
  SELECT
    exp_id,
    variant,
    channel,
    orders,
    order_share
  FROM (...)  -- 替换成你上一步的结果
),
pivot AS (
  SELECT
    channel,
    MAX(CASE WHEN variant = 'A' THEN orders END) AS A_orders,
    MAX(CASE WHEN variant = 'A' THEN order_share END) AS A_share,
    MAX(CASE WHEN variant = 'B' THEN orders END) AS B_orders,
    MAX(CASE WHEN variant = 'B' THEN order_share END) AS B_share
  FROM base
  GROUP BY channel
)
SELECT
  channel,
  A_orders,
  A_share,
  B_orders,
  B_share,
  (B_share - A_share) AS delta_share
FROM pivot
ORDER BY ABS(B_share - A_share) DESC;
```

---

## 7. Last-Touch Attribution（最后触达）怎么做？

只要把 click 改为 click + impression：

```sql
AND e.event_type IN ('impression', 'click')
```

其余逻辑保持一致（仍取最后一次触达）。

---

## 8. 结论到动作（Error → Action）

### Case A：发现订单 share 从 Feed → Search / Cart 迁移明显

**✅ 可能是"归因导致 CVR 假跌"**

**下一步：**
- 用 last-touch / assisted conversion 验证真实贡献
- 用 CTCVR、GMV/Imp 判断真实收益

### Case B：ATC/Click 也下降（点击质量下降）

**✅ 真实点击变"水"**

**下一步：**
- 检查目标错位（只优化 CTR）
- 改目标：CTCVR / GMV 或多目标（CTR + ATC）

### Case C：订单总量（CTCVR）也下降

**✅ 整体转化真的被伤到**

**下一步：**
- 排查 recall 引入低质量候选
- 或 rank 过度奖励 clickbait
- 增加 guardrail / 调整目标 / 分桶修复

---

## 10. ESMM：为什么能缓解 CVR 的选择偏差？

### 10.1 CVR 的核心问题：只能在点击样本上训练

CVR 的定义是：

```
CVR = P(order=1 | click=1, impression=1)
```

但现实中你只能观察到：
- `click=1` 的样本是否下单
- `click=0` 的样本没有后续行为（无法知道是否会下单）

因此 CVR 的训练数据天然是被 CTR "筛选"过的点击人群，存在 **Selection Bias（选择偏差）**：

```
CVR 学到的分布 = "点了的人"
但线上要面对的变化 = "点的人群会随 CTR 策略变化而漂移"
```

### 10.2 ESMM 的核心思想：用曝光级别学习"点击+成交"

ESMM 不直接学 CVR，而是学习两个曝光级别（impression-level）的概率：

**任务1：CTR（点击率）**
```
CTR = P(click=1 | impression=1)
```

**任务2：CTCVR（点击且转化）**
```
CTCVR = P(click=1, order=1 | impression=1)
```

因为 CTCVR 是在曝光级别定义的，所以它可以利用全量曝光数据（样本更多、更稳定）。

### 10.3 ESMM 如何得到 CVR？

由于：
```
P(click, order | impression) = P(click | impression) · P(order | click, impression)
```

所以：
```
CTCVR = CTR · CVR
```

推得：
```
CVR = CTCVR / CTR
```

这就是 ESMM 的关键：
- ✅ 用全量曝光去学 CTR 和 CTCVR
- ✅ 再通过比例推回 CVR

### 10.4 为什么 ESMM 能让 CVR 更稳？

当线上 CTR 策略变化时，点击样本分布会变化（click distribution shift）：
- 只训练 CVR（只看 click 人群） → 很容易漂
- ESMM 用 impression-level 数据去约束 CTCVR → 更稳

**一句话总结：**

ESMM 用"曝光级别的监督信号"对抗 CVR 的稀疏与选择偏差。

### 10.5 ESMM vs 直接训练 CVR：对比一句话总结

- **直接 CVR 模型：** 学习 `P(order | click)`，数据少且偏
- **ESMM：** 学习 `P(click)` 和 `P(click&order)`，数据更大更稳，CVR 由比例推得

### 10.6 什么时候 ESMM 特别有用？

ESMM 常在以下情况更有效：
- 转化非常稀疏（购买/付费极少）
- CTR 排序策略变化频繁（点击人群漂移大）
- 需要同时稳定 CTR 和 CVR（不希望 CVR 随点击波动）

---

## 11. PLE：为什么适合多场景/多意图 CVR（解决负迁移）

### 11.1 为什么 CVR 更容易出现"负迁移"？

CVR 受很多因素影响，且不同人群/场景的转化逻辑差异很大，例如：

**不同用户类型：**
- 价格敏感型用户：折扣/券决定是否买
- 品牌偏好型用户：品牌决定是否买
- 履约敏感型用户：配送/库存决定是否买
- 新用户：信任成本高，CVR 天生低
- 老用户：复购更稳定

如果你用一个共享模型去学所有情况，很容易出现：

```
某类场景的梯度更新，会伤害另一类场景（互相干扰）
这就叫 负迁移（Negative Transfer）
```

**典型现象：**
- 总体 CVR 有提升，但某些桶（新用户/高价带/特定品类）明显下降
- 多任务训练时一个任务涨、另一个任务掉

### 11.2 PLE 的核心：专家拆分 + 样本级门控

**PLE 全称：** Progressive Layered Extraction

它主要解决的问题是：
```
"共享信息要共享,但特有模式也要能独立学习"
```

PLE 的结构通常包含三类模块：

**1）Shared Experts（共享专家）**

学习多个任务之间的共性模式

例如：用户购买力、商品质量、基础匹配度等。

**2）Task-specific Experts（任务专属专家）**

学习每个任务/场景独有的规律

例如：CVR 更关心价格/库存/履约，CTR 更关心兴趣/吸引力。

**3）Gating Network（门控网络）**

对每个样本，动态决定：
- 该样本应该更依赖哪个专家
- 不同任务的 gating 不同

**一句话理解：**

PLE 让不同样本走不同"专家组合路径"，减少互相干扰。

### 11.3 PLE 为什么叫 Progressive（逐层提取）？

它不是只在一层做专家拆分，而是 **多层逐步提取**：
- 浅层：共享信息多（通用 embedding/基础交互）
- 深层：任务差异变大（CVR 更深的购买逻辑）

因此它比简单的 MMoE 更容易做到：
- 共享与私有更清晰
- 更不容易互相污染

### 11.4 PLE 更适合解决什么类型的"CTR↑ CVR↓"？

如果你的现象是：
- 整体 CTR ↑
- 整体 CVR ↓

并且你通过分桶发现：

**✅ CVR 下跌集中在某些桶，比如：**
- 新用户桶明显跌
- 高价带明显跌
- 购买意图 query 跌，但浏览意图 query 涨
- 某些类目跌得特别多

这通常说明：
```
不同场景的转化模式冲突很大，单一共享网络会互相拉扯
```

这种情况下，用 PLE 去分离不同模式更合理。

### 11.5 PLE 通常怎么用（常见配置）

在电商/推荐里，PLE 经常用于这些组合：

**多任务（Multi-task）**
- CTR + CVR + CTCVR
- CTR + ATC + CVR
- CTR + GMV proxy + CVR

**多场景（Multi-domain / Multi-intent）**
- Search vs Feed
- 新用户 vs 老用户
- 不同国家/语言站点
- 不同入口（首页、活动页、详情页推荐）

### 11.6 ESMM vs PLE：怎么选？

这两个不是同一类东西，解决的问题不同：

**✅ ESMM 更像 "解决 CVR 的数据偏差与稀疏"**
- 核心关键词：曝光级别监督、选择偏差、稳定性

**✅ PLE 更像 "解决多任务/多场景的训练冲突"**
- 核心关键词：多专家、门控、减少负迁移

**实际工程里也常见组合：**
- 用 ESMM 的思想定义目标（CTR + CTCVR → CVR）
- 再用 PLE 的结构增强多任务学习能力

---

## 12. 实战：CTR↑ CVR↓ Debug Checklist（1页速查版）

**场景：** 线上观察到 CTR 上升，但 CVR 下降

**目标：** 快速判断是 假跌（口径/归因/延迟） 还是 真实质量问题（目标错位/流量变差），并落到修复动作。

### 12.1 第 0 步：确认现象是否成立（Sanity Check）

**✅ 必查：**
- CVR 的转化窗口是否一致？（click 后 24h / 7d）
- 数据是否成熟？（看 T+1/T+2，避免 T+0 假跌）
- 实验组/对照组流量是否均衡？（sample ratio mismatch）
- 是否有外部问题：库存/支付/履约/券（影响 CVR 但和排序无关）

### 12.2 第 1 步：别只看 CVR，看全链路指标

**✅ 同时拉这四个核心指标：**
- CTR = Click / Impression
- CVR = Order / Click
- CTCVR = Order / Impression
- GMV/Imp = GMV / Impression

**快速结论：**
- CTR↑、CVR↓，但 CTCVR↑或持平 → 可能不是坏事（或归因迁移）
- CTCVR↓ → 真实伤到成交（高优先级处理）

### 12.3 第 2 步：看点击后漏斗（判断点击质量）

**✅ 最强诊断：Click → ATC → Checkout → Pay**
- ATC/Click（加购率）
- Checkout/Click（发起支付率）
- Pay/Click（支付成功率 ≈ CVR）
- Refund/Order（退货率：成交质量）

**结论判断：**
- CTR↑ 且 ATC/Click↓ → 点击质量变"水"（多半目标错位）
- CTR↑ 且 ATC/Click≈稳，但 Pay/Click↓ → 可能支付/库存/归因/延迟
- CTR↑ 且 ATC/Click↑ 但 CVR↓ → 检查归因规则（订单算走了）

### 12.4 第 3 步：归因分布对比（判断订单是否"被算走"）

**✅ 做 last-click 分布表（A/B 对照）：**
- Orders share by channel（Search / Feed / Cart / Ads / …）
- GMV share by channel

**重点看：**
- 是否发生 Feed → Search 的归因迁移？
- 是否发生 Feed → Cart 的归因迁移？

**解释模板：**

"实验组订单更多归因到 Search/Cart，说明用户可能先在 Feed 被种草，后在 Search 或 Cart 完成购买，Feed CVR 下降可能是归因迁移导致的假象。"

**✅ 补充验证：**
- 用 Last-touch / assisted conversion 再算一遍（更宽松）

### 12.5 第 4 步：做分桶定位（找到 CVR 下跌集中在哪）

**常用分桶维度（每桶看 CTR/CVR/CTCVR）：**
- Query intent：购买型 vs 浏览型
- New user vs returning user
- New item vs old item
- Price bucket：低价/中价/高价
- Position bucket：Top1-3 vs 4-10
- Category bucket（品类结构）
- Device / region

**结论：**
- CVR 下跌集中在特定桶 → 优先做桶内修复（更快见效）

### 12.6 常见根因 → 对应修复动作（Error → Action）

#### Case A：归因迁移导致的 CVR 假跌

**特征：**
- CTCVR 或 GMV/Imp 没跌
- last-click 分布发生迁移（Feed → Search/Cart）

**行动：**
- ✅ 用 last-touch / assisted 口径补充评估
- ✅ 以 GMV/Imp、CTCVR 作为主要决策指标
- ✅ 不急着改模型，先确认业务最终收益

#### Case B：点击质量变差（目标错位：只追 CTR）

**特征：**
- CTR↑ 且 ATC/Click↓、Checkout/Click↓
- TopK 更多 clickbait item（高点低买）

**行动（优先级从高到低）：**
- ✅ 改排序目标：CTCVR / GMV/Imp / 多目标（CTR+ATC+CVR）
- ✅ 加 guardrail：对高点击低成交 item 限权/降权
- ✅ 分 intent 做策略：购买型 query 更偏交易结果

#### Case C：CVR 训练被点击分布漂移影响（选择偏差放大）

**特征：**
- CTR 策略变化频繁
- 点击人群特征分布变化大（PSI/KS 显著）
- CVR 模型线上波动大

**行动：**
- ✅ 用 ESMM / CTCVR 建模增强稳定性（曝光级别监督）
- ✅ 多任务学习（CTR + CTCVR + CVR）
- ✅ 训练/特征分布监控（drift detection）

#### Case D：多场景/多意图冲突（负迁移）

**特征：**
- 总体可能还行，但某些桶 CVR 崩
- 不同人群/场景的规律明显不同

**行动：**
- ✅ 上 PLE / MMoE（多专家 + gating）
- ✅ 分场景建模或分桶再融合
- ✅ 增加 intent-aware 的建模与路由
