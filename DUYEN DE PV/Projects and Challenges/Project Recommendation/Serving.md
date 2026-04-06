Let me walk through both APIs request by request, very concretely.

---

## Home API

**Request comes in:**

```
{ customer_id: 12345, platform: "ios", source_screen: "home" }
```

**Step 1 — Parallel fetches**

Three things happen simultaneously:

```
Feast online store  → batch customer features (18 features, keyed by customer_id)
Redis               → session features (6 features, keyed by customer_id)
HTTP request        → platform, hour_of_day, day_of_week, is_weekend
```

No computation yet. Pure data fetching.

**Step 2 — User embedding**

Now you have all 28 features. Run them through the ONNX user tower:

```
[18 batch + 6 session + 4 request] → Dense(128) → ReLU → Dense(64) → L2 normalize
→ user_embedding [64-dim vector]
```

This is the only neural network inference in the entire serving path.

**Step 3 — Parallel candidate retrieval**

Two channels fire simultaneously:

```
Channel A: FAISS.search(user_embedding, k=1000)
           → 1000 product_ids + similarity scores
           → these are pre-computed item embeddings, FAISS just does ANN search

Channel B: Redis ZREVRANGE
           → user's top 3 batch cates from Feast + session_top_cate if different
           → ZREVRANGE trending:cate:{cate_1} 0 49
           → ZREVRANGE trending:cate:{cate_2} 0 49
           → ZREVRANGE trending:cate:{cate_3} 0 49
           → ZREVRANGE trending:global 0 49
           → ~200 trending product_ids, pipelined in single round-trip
```

**Step 4 — Merge and rank**

Union both candidate sets, dedup by product_id. Then for each candidate compute a score:

```
FAISS candidates:    similarity_score × category_match_boost × brand_match_boost
Trending candidates: trending_score × category_relevance
Items in both:       get highest combined score (strongest signal)
Bonus:               boost items similar to session_last_product_id
```

The `category_match`, `brand_match`, `price_match` cross-features are computed here — this is the moment you know both the user and the specific candidates simultaneously.

**Step 5 — Filter and diversify**

```
Remove: already purchased, not salable
Cap:    max 3 products per primary category
Return: top 10–50
```

**Total: ~50–80ms P95**

The dominant costs are ONNX inference (~5ms) and FAISS search (~10ms). Everything else is Redis round-trips.

---

## PDP API

**Request comes in:**

```
{ product_id: 98765, customer_id: 12345 (optional) }
```

Notice — no user embedding, no ONNX inference. The question being answered is about the _product_, not the user.

**Step 1 — Parallel fetches**

```
KVDal lookup(product_id)  → pre-computed similar + complementary product_ids with scores
Redis ZREVRANGE            → trending:cate:{product's own cate_id}
                           → trending:cate:{complementary_cate_1}
                           → trending:cate:{complementary_cate_2}
                           pipelined in single round-trip
```

That's it. No Feast call. No ONNX. Just two parallel lookups.

**Step 2 — Merge**

Interleave KVDal results with trending results:

```
KVDal list  → relevance-ordered (stable similarity)
Redis list  → trending-ordered (freshness)
Merged      → interleaved so neither source dominates
```

**Step 3 — Filter**

```
Remove: not salable, out of stock, dedup
Return: ranked product list per widget type
        (similar products / complementary products / infinity scroll)
```

**Total: ~5–10ms P95**

Almost entirely Redis and KVDal round-trip latency. No neural network inference at all.

---

## Side by side

```
                    HOME                        PDP
                    ────                        ───
Feast call?         Yes (batch customer feat)   No
Redis call?         Yes (session + trending)    Yes (trending only)
KVDal call?         No                          Yes (similarity lists)
ONNX inference?     Yes (user tower)            No
FAISS search?       Yes (ANN on user embedding) No
Cross-features?     Yes (at merge time)         No
Neural compute?     ~5ms                        0ms
Dominant cost?      ONNX + FAISS (~15ms)        Network round-trips (~5ms)
Total P95           ~50–80ms                    ~5–10ms
```

The PDP API being 10x faster than Home is a direct consequence of the architecture decision to pre-compute everything product-related offline. At serving time it genuinely has nothing to compute — it just fetches and merges two pre-ranked lists.

### excerpt_from_previous_claude_message.txt

1 line

txt

hm don't quite get it like user is watching product_id 123, product id have list of relevant product id which was precomputed like 435,567,... and 435 has relevant products like 111, 222. do product 111,222 get recommended because it gonna cost a lot of computation. i heard that shopee use gnn to solve this right for their recommendation problem.

10:45 AM

Great question — you're identifying a real architectural boundary.

---

## What actually happens with KVDal

Yes exactly. Product 123 has a pre-computed list:

```
product_123 → [435, 567, 891, 234, ...]   (top-N similar, stored at write time)
```

At serving time you fetch that list and **stop there**. You do NOT follow the graph:

```
❌ product_123 → 435 → 111, 222   (this does NOT happen)
❌ product_123 → 567 → 333, 444   (this does NOT happen)
```

111 and 222 never get recommended via this path. The lookup is exactly one hop, pre-computed offline. That's it.

---

## Why only one hop

Because KVDal is a key-value store — it has no concept of graph traversal. It just returns whatever list was written to `key=product_123` during the offline batch job. The "graph" only exists implicitly in how you _built_ that list (co-view sessions), not in how you _serve_ it.

---

## The limitation this creates

You're right to sense a problem. One-hop pre-computation means:

- **Coverage gaps** — if product 123 has very few co-views (new product, niche item), its pre-computed list is short or empty
- **Stale relationships** — the list is rebuilt daily, so a product that suddenly becomes relevant won't appear until tomorrow
- **No transitive discovery** — product 111 might be highly relevant to users who viewed 123, but only indirectly (via 435). You miss it entirely.

---

## How Shopee's GNN approach solves this

You're right that Shopee (and most large e-commerce platforms at scale) use Graph Neural Networks for exactly this reason.

The core idea: instead of pre-computing flat lists, you build a **heterogeneous graph**:

```
Nodes: products, users, categories, brands, sellers
Edges: viewed, purchased, co-viewed, belongs_to, sold_by, ...
```

Then GNN propagates information across hops during _training_:

```
product_123 embedding gets influenced by:
  hop 1: products co-viewed with 123 (435, 567...)
  hop 2: products co-viewed with 435 (111, 222...)
  hop 3: products co-viewed with 111 (...)
```

So product 111's signal _flows into_ product 123's embedding during training — even if they were never directly co-viewed. The embedding captures multi-hop relationships.

At serving time it still just does ANN search on embeddings — no graph traversal needed. The graph complexity is baked into the embeddings during training, not at serving time.

```
Training:  GNN on full graph → rich embeddings capturing multi-hop structure
Serving:   same ANN lookup as before, just better embeddings
```

---

## Where your current architecture sits

```
Your architecture (LightFM + SBERT):
  Captures:     direct co-view pairs (1 hop)
  Misses:       transitive relationships (2+ hops)
  Serving:      flat KVDal lookup
  Complexity:   low, interpretable, fast to train

GNN approach (Shopee-style):
  Captures:     multi-hop relationships across user-item-category graph
  Handles:      cold start better (borrows signal from graph neighbors)
  Serving:      still ANN, but richer embeddings
  Complexity:   significantly higher — graph construction, neighbor sampling, training infra
```

---

## Honest tradeoff for your scale

At Tiki's current scale, moving to GNN is probably **not the first thing to do**. The wins from GNN show up most clearly when:

- Catalog is very large (100M+ items) with sparse interaction data
- Cold start is a severe problem
- You have the infra to build and maintain a heterogeneous graph pipeline

A more pragmatic next step toward capturing multi-hop signal would be **random walk based embeddings** (like Node2Vec on your co-view graph) — much simpler than full GNN, but captures 2–3 hop relationships. You'd replace or augment the LightFM interaction similarity with Node2Vec product embeddings, still write to KVDal, still serve the same way.

The GNN path is the right long-term direction but carries significant infrastructure cost — graph construction on BigQuery, neighbor sampling during training, incremental graph updates as new interactions arrive. Worth knowing about for the interview as "what we'd explore next" rather than current state.

dive into detail of training phase for both home and pdp

10:46 AM

Let me walk through both training pipelines in full detail.

---

## Home: Two-Tower Training

### What you're trying to learn

The model learns to map users and items into the same 64-dim embedding space such that **users are close to items they'll interact with**. The training signal is implicit feedback — a PDP view is a positive signal that user U is interested in item I.

---

### Step 1 — Build the training dataset from BigQuery

The core table is `pdp_view_raw`. Each row is one (user, item, timestamp) interaction.

sql

```sql
WITH raw_interactions AS (
  SELECT
    customer_id,
    product_id,
    session_id,
    event_time,
    primary_cate_id,
    event_type
  FROM `tiki-dwh.recommendation.pdp_view_raw_*`
  WHERE _TABLE_SUFFIX BETWEEN
    FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY))
    AND FORMAT_DATE('%Y%m%d', CURRENT_DATE)
    AND customer_id IS NOT NULL  -- exclude anonymous
),

-- time-based train/eval split
split AS (
  SELECT *,
    CASE
      WHEN event_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP, INTERVAL 3 DAY)
      THEN 'eval'
      ELSE 'train'
    END AS split
  FROM raw_interactions
)
```

**Why time-based split, not random:** if you split randomly, a user's future interactions leak into training. The model memorizes rather than generalizes. Time split forces the model to predict future behavior from past behavior — which is exactly the serving scenario.

---

### Step 2 — Reconstruct session features

At serving time, session features come from Redis. During training, you reconstruct them from the raw event log using window functions:

sql

```sql
session_features AS (
  SELECT
    customer_id,
    product_id,
    session_id,
    event_time,
    primary_cate_id,
    split,

    -- how many products viewed before this event in the same session
    COUNT(*) OVER (
      PARTITION BY customer_id, session_id
      ORDER BY event_time
      ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING
    ) AS session_view_count,

    -- ATCs before this event in the same session
    SUM(CASE WHEN event_type = 'add_to_cart' THEN 1 ELSE 0 END) OVER (
      PARTITION BY customer_id, session_id
      ORDER BY event_time
      ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING
    ) AS session_atc_count,

    -- most viewed category so far in this session
    APPROX_TOP_COUNT(primary_cate_id, 1) OVER (
      PARTITION BY customer_id, session_id
      ORDER BY event_time
      ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING
    )[OFFSET(0)].value AS session_top_cate

  FROM split
),

-- previous session features
session_summaries AS (
  SELECT
    customer_id,
    session_id,
    COUNT(*) AS total_views,
    APPROX_TOP_COUNT(primary_cate_id, 1)[OFFSET(0)].value AS top_cate,
    MIN(event_time) AS session_start
  FROM split
  GROUP BY customer_id, session_id
),

with_prev_session AS (
  SELECT
    s.*,
    LAG(total_views) OVER (
      PARTITION BY customer_id ORDER BY session_start
    ) AS last_session_view_count,
    LAG(top_cate) OVER (
      PARTITION BY customer_id ORDER BY session_start
    ) AS last_session_top_cate
  FROM session_summaries
)
```

The reconstructed values won't be byte-identical to Redis — but they're statistically consistent. Same distributions, same meaning. The model learns to use session features correctly either way.

---

### Step 3 — Join batch features

Join each (user, item) interaction with the batch features that were valid _at that point in time_. This is the point-in-time correctness problem — you don't want to use today's feature values for an interaction that happened 20 days ago.

sql

```sql
training_data AS (
  SELECT
    sf.customer_id,
    sf.product_id,
    sf.event_time,
    sf.split,

    -- session features (reconstructed)
    sf.session_view_count,
    sf.session_atc_count,
    sf.session_top_cate,
    ps.last_session_view_count,
    ps.last_session_top_cate,
    IFNULL(ps.last_session_view_count >= 5, FALSE) AS last_session_was_deep,

    -- batch user features (from Feast offline store / BQ snapshot)
    uf.age_group,
    uf.gender,
    uf.region,
    uf.num_pdp_views_d30,
    uf.num_atc_d30,
    uf.num_purchases_d365,
    uf.primary_cate_id AS top_cate,
    -- ... remaining 18 user batch features

    -- batch item features
    pf.primary_cate_id,
    pf.sale_price,
    pf.avg_rating_all,
    pf.num_views_d30,
    pf.ctr_d30,
    -- ... remaining 15 item batch features

    -- request features (derived from event_time)
    EXTRACT(HOUR FROM sf.event_time) AS hour_of_day,
    EXTRACT(DAYOFWEEK FROM sf.event_time) AS day_of_week,
    EXTRACT(DAYOFWEEK FROM sf.event_time) IN (1, 7) AS is_weekend

  FROM session_features sf
  LEFT JOIN with_prev_session ps
    ON sf.customer_id = ps.customer_id
    AND sf.session_id = ps.session_id
  LEFT JOIN customer_features_snapshot uf  -- point-in-time snapshot
    ON sf.customer_id = uf.customer_id
  LEFT JOIN product_features pf
    ON sf.product_id = pf.product_id
)
```

---

### Step 4 — Model architecture and loss

python

```python
class UserTower(nn.Module):
    def __init__(self):
        super().__init__()
        self.cate_embedding = nn.Embedding(num_cates, 8)
        self.platform_embedding = nn.Embedding(4, 4)
        self.fc1 = nn.Linear(28, 128)
        self.fc2 = nn.Linear(128, 64)

    def forward(self, batch_features, session_features, request_features):
        # embed categorical features
        cate_emb = self.cate_embedding(session_features['session_top_cate'])
        platform_emb = self.platform_embedding(request_features['platform'])

        x = torch.cat([
            batch_features,    # 18 dims
            session_features,  # 6 dims (including embedded cates)
            request_features,  # 4 dims
        ], dim=-1)             # → 28 dims total

        x = F.relu(self.fc1(x))
        x = self.fc2(x)
        return F.normalize(x, dim=-1)  # L2 normalize → unit sphere

class ItemTower(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(15, 128)
        self.fc2 = nn.Linear(128, 64)

    def forward(self, item_features):
        x = F.relu
```

  

Sonnet 4.6