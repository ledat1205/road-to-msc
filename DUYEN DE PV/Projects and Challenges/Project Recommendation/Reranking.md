## Reranking Layer — Home and PDP

---

### Why Reranking Exists at All

Retrieval and ranking solve different problems.

**Retrieval** (FAISS + Redis trending) asks: _is this item broadly relevant to this user?_ It operates over millions of items and must be fast — so it uses approximate methods (ANN search, pre-sorted lists). The scoring is coarse: cosine similarity in embedding space, trending score from a sorted set.

**Reranking** asks: _given everything I know about this specific user right now and these specific candidates simultaneously, what is the optimal ordering?_ It operates over a small candidate set so it can afford richer features and more expensive computation.

The gap between the two is where reranking lives. The retrieval stage casts a wide net. The reranker decides what actually surfaces to the user.

---

### Home Reranking

#### The candidate pool

```
FAISS(user_embedding, k=1000)     → 1000 personalized candidates
Redis ZREVRANGE (user's top cates) → ~200 trending candidates
Union + dedup                      → ~1100-1200 total
```

#### Two-stage design

You can't rerank 1200 candidates with a rich model — even a fast GBM on 1200 rows adds meaningful latency and most of those candidates are clearly irrelevant anyway. So you prune first:

```
~1200 candidates
      ↓
Stage 1 — Lightweight pruning → top 200
  Current merge scoring (arithmetic):
  FAISS items:    similarity_score × category_match_boost × brand_match_boost
  Trending items: trending_score × category_relevance
  Items in both:  combined score (strongest signal)
  Cost: ~1ms, no model call

      ↓
Stage 2 — Reranker → top 50
  LightGBM on cross-features
  Cost: ~2-3ms
```

#### What the reranker scores that merge scoring misses

Merge scoring is a hand-tuned weighted formula. It can't learn feature interactions. The reranker can:

```
Feature                         Why it matters
───────                         ──────────────
came_from_both_channels         Item in FAISS AND trending = strongest signal
                                A formula can add scores. A tree learns
                                the interaction is multiplicative not additive.

faiss_rank                      Rank 1 vs rank 800 means different things
                                depending on session depth. A user with
                                20 session views has a more reliable embedding
                                than one with 2 views. Tree learns this interaction.

price_ratio                     item_price / user_avg_order_value
                                A budget user clicking a premium item
                                is weaker signal than a premium user doing same.

days_since_last_view            "Viewed 1 hour ago" vs "viewed 3 days ago"
                                decays differently per category.
                                Electronics decay fast. Books decay slowly.

session_atc_count × cate_match  High cart intent + item matches session category
                                = very strong purchase signal. Neither feature
                                alone captures this — the interaction does.

category_saturation             How many items from this category already
                                served in this session. Penalizes redundancy
                                that diversity cap alone doesn't catch.
```

#### Training data for the Home reranker

The reranker needs **impression logs** — not just clicks, but every product shown to every user with its position and which retrieval channel it came from. Without impressions you only have positive labels, which biases toward popularity.

```sql
SELECT
  customer_id,
  product_id,
  session_id,
  impression_time,

  -- retrieval signals
  faiss_rank,
  faiss_similarity_score,
  trending_rank,
  trending_score,
  in_both_channels,

  -- cross-features at impression time
  has_viewed_before,
  has_carted_before,
  days_since_last_view,
  category_match,
  brand_match,
  price_ratio,
  session_view_count,
  session_atc_count,
  category_saturation,

  -- item quality signals
  avg_rating_all,
  ctr_d30,
  composite_quality_score,

  -- label
  CASE
    WHEN subsequent_event = 'purchase'   THEN 2
    WHEN subsequent_event = 'add_to_cart' THEN 1
    ELSE 0
  END AS label

FROM impression_log
JOIN cross_features USING (customer_id, product_id, session_id)
WHERE impression_date >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
```

**Model:**

```python
model = lgb.LGBMRanker(
    objective='lambdarank',  # optimizes NDCG directly
    metric='ndcg',
    n_estimators=300,
    learning_rate=0.05,
    num_leaves=63,
)

model.fit(
    X_train, y_train,
    group=group_sizes,   # number of candidates per user session
)
```

LambdaRank is the right objective here — it directly optimizes the ranking metric (NDCG) rather than treating each item independently. It knows that moving a relevant item from rank 10 to rank 1 matters more than moving it from rank 40 to rank 30.

#### Infinite scroll case

For infinite scroll, the reranker runs on every batch request as the user scrolls deeper. The architecture changes:

```
First request:
  FAISS(k=3000) + Redis → ~3200 candidate pool
  Store pool in Redis: session:{session_id}:candidate_pool  TTL=30min
  Rerank top 200 → serve batch 1 (items 1-50)

Each scroll request:
  Fetch remaining pool from Redis (~2ms, no FAISS call)
  Update session context from scroll behavior since last batch
  Rebuild cross-features for remaining pool with updated context
  LightGBM rerank remaining pool (~3-5ms, pool shrinks each batch)
  Apply decaying diversity penalty
  Serve next batch (items 51-100, 101-150, ...)

Pool refresh (async, triggered when pool_size < 500):
  Recompute user_embedding with latest session features
  FAISS(updated_embedding, k=2000) → fresh candidates
  Dedup against already-served items
  Append net new to existing pool
```

The diversity penalty decays over scroll depth — categories get progressively penalized as the user has already seen many items from them, naturally introducing variety without abandoning the user's demonstrated intent:

```python
def diversity_penalty(candidate_cate, served_category_counts, scroll_depth):
    times_shown = served_category_counts.get(candidate_cate, 0)
    penalty = times_shown * 0.1 * (1 + scroll_depth / 100)
    return penalty

final_score = reranker_score - diversity_penalty(...)
```

---

### PDP Reranking

#### Why PDP reranking is a different problem

Home reranking is about personalization — the same candidate set gets reordered differently for different users. PDP reranking has no user identity at its core (or a weak one) — it's about product-to-product relevance quality and freshness balance.

The candidate pool is also much smaller:

```
KVDal lookup(product_id)    → top-200 pre-computed similar products
                              + top-200 complementary products
Redis ZREVRANGE             → ~100-150 trending in same + complementary cates
Union + dedup               → ~300-400 total candidates
```

At this scale a full two-stage design is overkill. One reranking pass over 300-400 candidates is fast enough.

#### What PDP reranking optimizes

The core tension in PDP is **relevance vs freshness**:

- KVDal gives you relevance — pre-computed similarity scores that are stable and accurate but computed yesterday
- Redis gives you freshness — trending products right now but with no direct relevance signal to the anchor product

The reranker learns to balance these two signals given the specific context:

```
Features for PDP reranker:

Relevance signals:
  lightfm_similarity_score      (behavioral co-occurrence strength)
  sbert_similarity_score        (semantic title similarity)
  combined_similarity_score     (weighted combination)
  cate_match                    (same vs complementary category)
  rank_in_kvdal                 (position in pre-computed list)

Freshness signals:
  trending_score_1h             (raw trending score from Redis)
  trending_rank_in_cate         (position in category trending list)
  is_from_trending_only         (not in KVDal, pure trending candidate)
  in_both_sources               (in KVDal AND trending — strongest signal)

Item quality signals:
  avg_rating_all
  ctr_d30
  composite_quality_score
  is_official_store
  fulfillment_type

Context signals:
  anchor_cate_id                (what category is the anchor product)
  anchor_price_segment          (price tier of the anchor)
  price_ratio                   (candidate_price / anchor_price)
                                (too expensive or cheap = poor complement)
  same_brand                    (brand match with anchor)
  same_seller                   (same seller — can be good or bad signal)
```

**The price_ratio feature deserves emphasis.** If a user is viewing a budget laptop (5 million VND), showing a 50 million VND monitor as "complementary" is poor UX even if they co-occur in sessions occasionally. The reranker learns that price_ratio extremes should be penalized regardless of co-view signal.

#### Training data for the PDP reranker

```sql
SELECT
  anchor_product_id,
  candidate_product_id,
  impression_time,

  -- relevance signals
  lightfm_similarity_score,
  sbert_similarity_score,
  combined_similarity_score,
  rank_in_kvdal,
  cate_relationship,           -- 'same' or 'complementary'

  -- freshness signals
  trending_score_1h,
  trending_rank_in_cate,
  in_both_sources,

  -- item quality
  avg_rating_all,
  ctr_d30,
  composite_quality_score,

  -- context
  price_ratio,
  same_brand,
  same_seller,

  -- label
  CASE
    WHEN subsequent_event = 'purchase'    THEN 2
    WHEN subsequent_event = 'add_to_cart' THEN 1
    WHEN subsequent_event = 'pdp_view'    THEN 0.5
    ELSE 0
  END AS label

FROM pdp_impression_log
JOIN pdp_cross_features USING (anchor_product_id, candidate_product_id)
WHERE impression_date >= DATE_SUB(CURRENT_DATE, INTERVAL 30 DAY)
```

Same LightGBM LambdaRank model as Home — different features, same objective. Group is now `anchor_product_id` instead of user session — you're ranking candidates relative to an anchor product, not relative to a user.

#### Per-widget reranking

PDP has three distinct widgets, each with a different optimization target. You either train one model with a `widget_type` feature, or three separate models:

```
Widget               Optimize for        Key features to weight
──────               ────────────        ──────────────────────
Similar products     Relevance           lightfm_score, sbert_score,
                                         same cate, same brand

Complementary        Purchase together   complementary cate pairs,
products             intent              price_ratio within reasonable range,
                                         in_both_sources

Infinity scroll      Diversity +         Blend of all signals with
                     sustained           decaying similarity threshold
                     engagement          as scroll deepens
```

For the infinity scroll widget on PDP the same pool + progressive rerank pattern from Home applies — store a larger candidate pool, rerank each batch, apply decaying diversity penalty as the user scrolls.

---

### Side by Side

```
                    Home reranking          PDP reranking
                    ──────────────          ─────────────
Candidate pool      ~1200                   ~300-400
Stages              2 (prune to 200,        1 (rerank full pool)
                    then rerank)
Primary signal      User × item             Anchor product × candidate
                    cross-features          relevance + freshness balance
Key features        session context,        price_ratio, similarity scores,
                    channel membership,     trending signals, cate relationship
                    price sensitivity
Group for           user session_id         anchor_product_id
LambdaRank
Training label      click / ATC /           pdp_view / ATC / purchase
                    purchase                from anchor PDP
Infinite scroll?    Yes — pool stored       Yes — for infinity scroll
                    in Redis per session    widget only
Latency added       ~2-3ms                  ~1-2ms
Model               LightGBM LambdaRank     LightGBM LambdaRank
Retrain cadence     Weekly                  Weekly
```

---

### The Prerequisite Neither Model Can Skip

Both rerankers need **impression logs** — a record of every candidate shown to every user (or shown on every PDP), not just the ones that got clicked.

Without impression logs:

```
You only know: user saw product_A and clicked it
You don't know: user also saw product_B, C, D and ignored them

Training signal: only positives
Model learns:    popular products score high
                 (they appear in positives most often)
Result:          reranker degrades to a popularity model
```

With impression logs:

```
You know: user saw A, B, C, D — clicked A, ignored B, C, D
Training signal: A=positive, B/C/D=negative
Model learns:    what distinguishes A from B, C, D
                 for this specific user in this specific context
Result:          genuine reranking signal
```

Impression logging is the infrastructure prerequisite to build before either reranker. Without it both models are training on biased data and the reranking layer adds noise rather than signal.