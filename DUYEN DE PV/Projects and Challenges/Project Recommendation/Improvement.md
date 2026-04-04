# Training + Inference Architecture

  

How the recommendation system trains models, retrieves candidates, and serves personalized results. There are two distinct recommendation surfaces — **Home widgets** and **PDP widgets** — with different architectures because they answer different questions.

  

---

  

## Two Surfaces, Two Architectures

  

| | Home Widgets | PDP Widgets |

|---|---|---|

| **Question** | "What should this user see next?" | "What's relevant to this product?" |

| **Key input** | User identity + browsing history + session | The product being viewed |

| **Model** | Two-Tower (user embedding × item embedding) | Product-to-product similarity (LightFM interaction + Sentence-BERT title) |

| **Real-time needed?** | Yes — session context shifts the user embedding | Partially — similarity is pre-computed, but trending products inject freshness |

| **Retrieval** | FAISS (personalized) + Redis sorted sets (trending) | KVDal (pre-computed similarity) + Redis sorted sets (trending in same/complementary cates) |

| **Latency** | ~50-80ms (model inference + ANN search) | ~5-10ms (key-value lookups) |

  

---

  

## Part 1: Home Widgets

  

### System Overview

  

```

User Request (user_id, context)

↓

┌──────────────────────────────────────────────────────┐

│ SERVING │

│ │

│ 1. Fetch features (Feast batch + Redis session) │

│ 2. Compute user embedding (ONNX user tower) │

│ 3. Retrieve candidates from TWO channels: │

│ ┌─────────────────┐ ┌────────────────────┐ │

│ │ Channel A: FAISS │ │ Channel B: Redis │ │

│ │ "what does this │ │ "what's happening │ │

│ │ user want?" │ │ right now?" │ │

│ │ (personalized) │ │ (trending) │ │

│ └────────┬─────────┘ └───────┬────────────┘ │

│ └──────┬─────────────┘ │

│ ▼ │

│ 4. Merge + rank (dedup, cross-features, scoring) │

│ 5. Filter + diversify (purchased, salable, cate cap)│

│ │

│ Latency: ~50-80ms P95 │

└──────────────────────────────────────────────────────┘

↓

Top 10-50 recommended products

```

  

FAISS and the trending channel answer **different questions**. The Two-Tower model encodes long-term preferences (30-day history) plus current session intent into a user embedding, then finds the nearest items in embedding space. The trending channel pulls from Redis sorted sets — pre-ranked lists of products that are spiking in views/ATCs/purchases right now — products the model has no way of knowing about because they weren't trending when it was trained.

  

These are two separate candidate sets that get merged. A trending laptop bag won't appear in FAISS results if the user has never browsed bags — but it will come from the trending set if it's spiking right now. Conversely, a niche product the user would love based on their history won't show up in trending — but FAISS will find it.

  

### Feature Taxonomy

  

Features live in three places, each with a different update cadence.

  

#### Batch Features (daily, dbt → BigQuery → Feast)

  

Computed daily by dbt, materialized to BigQuery, served via Feast online store. These capture stable, historical patterns over 7-90 day windows.

  

**Product features:**

  

| Feature View | Features | Source |

|---|---|---|

| `product_core` | product_name, primary_cate_id, seller_id, brand, sale_price, list_price, is_salable, tiki_hero, is_official_store, fulfillment_type, business_type | `stg_root_product` |

| `product_pdp_views_d30` | num_views_d30/d7/d1, num_distinct_customers_d30/d7, num_distinct_sessions_d30 | `int_product_pdp_views` |

| `product_impression_ctr` | num_impressions_d30/d1, num_clicks_d30/d1, ctr_d30/d1 | `int_product_impression_ctr` |

| `product_ratings` | avg_rating_all, num_ratings_all, min/max/median_rating_all, avg_rating_d30, num_ratings_d30 | `int_product_ratings` |

| `product_sales_d90` | num_distinct_buyers_d90/d30, total_quantity_d90/d30, total_amount_d90/d30, avg/min/max_price_d30 | `int_product_sales` |

| `product_price_segment` | price_segment, cate_avg_price, p25/p50/p75 | `int_product_price_segment` |

| `product_competition` | num_competitors_same_cate_same_segment, num_unique_brands, market_share | `int_product_competition` |

| `product_quality_score` | rating_score, volume_score, conversion_score, ctr_score, composite_quality_score | `int_product_quality_score` |

  

**Customer features:**

  

| Feature View | Features | Source |

|---|---|---|

| `customer_demographics` | age_group, gender, region, city | `stg_customer_demographics` |

| `customer_behavior_d30` | num_pdp_views_d30/d7, num_atc_d30, num_purchases_d365 | `int_customer_behavior` |

| `customer_category_affinity` | primary_cate_id, num_cate_views_d90/d30, num_cate_purchases_d90, total_cate_amount_d90 | `int_customer_category_affinity` |

| `customer_brand_affinity` | primary_cate_id, top_brand_1/2/3, top_brand_1_purchases | `int_customer_brand_affinity` |

| `customer_order_quality` | num_orders_d90/d30, total_value_d90/d30, avg_order_value_d30 | `int_customer_order_quality` |

| `customer_lifetime_value` | lifetime_orders, lifetime_value, avg_order_value, days_since_first/last_purchase, customer_segment, value_segment | `int_customer_lifetime_value` |

  

**Cross-features (on_demand, computed at serving time from batch data):**

  

| Feature View | Features |

|---|---|

| `customer_product_cross_features` | has_viewed_before, has_carted_before, has_purchased_before, days_since_last_view, category_match, brand_match, price_match |

  

#### Session Features (Spark Streaming → Redis)

  

Computed by Spark Structured Streaming from Kafka events. Aggregated **per user session**, not per calendar day. A session ends after 30 minutes of inactivity (matching the existing session definition in `tiki_events_sessions_cross_devices`).

  

**User session state (key: `customer:{customer_id}:session`):**

  

| Feature | Description | Use |

|---|---|---|

| `session_id` | Current session ID | Session boundary detection |

| `session_view_count` | Products viewed in current session | **Model input** (user tower) — browsing depth signal |

| `session_atc_count` | Items added to cart in current session | **Model input** (user tower) — purchase intent signal |

| `session_cates` | Array of primary_cate_ids viewed this session | **Model input** (user tower, top-1 encoded) — category intent |

| `session_last_product_id` | Last product viewed | Merge scoring — recency |

| `last_session_cates` | Array of primary_cate_ids from previous session | **Model input** (user tower, top-1 encoded) — carries over intent across sessions |

| `last_session_view_count` | Products viewed in previous session | **Model input** (user tower) — was previous session deep or shallow |

  

When a new session starts (>30min gap), the current session state rolls into `last_session_*` fields, and current session resets to zero.

  

**Why session, not calendar day:**

- A user browsing electronics at 11pm and continuing at 1am is one continuous intent — splitting at midnight loses context

- A user who browses in the morning and comes back in the evening has two distinct intents — session boundaries capture this naturally

- Session is the standard unit in all existing event tables (`session_id` is already in `core_events`, `pdp_view_raw`)

  

#### Product Trending (Spark Streaming → Spark Streaming → Redis sorted sets)

  

| Feature | Description | Use |

|---|---|---|

| `trending_views_1h` | PDP view count in last hour | Redis trending retrieval (both Home + PDP) |

| `trending_atc_1h` | Add-to-cart count in last hour | Redis trending retrieval (both Home + PDP) |

| `trending_purchases_1h` | Purchase count in last hour | Redis trending retrieval (both Home + PDP) |

| `trending_score_1h` | Weighted: view=1, atc=3, purchase=5 | Trending retrieval (both Home + PDP) |

  

Spark Streaming writes per-product scores to Redis **and** maintains pre-sorted trending lists as Redis sorted sets:

  

```

Per-product scores:

product:{product_id}:trending_1h → {views, atc, purchases, score}

  

Pre-sorted trending lists (ZADD with trending_score):

trending:global → top products globally

trending:cate:{cate_id} → top products per primary category

```

  

Serving retrieves trending candidates via `ZREVRANGE` (~1-2ms). These sorted sets are used by **both** Home and PDP serving as a trending retrieval channel. They are **not model inputs** — the model handles personalization, trending handles recency/popularity.

  

#### Request-Time Features (from HTTP request)

  

| Feature | Description |

|---|---|

| `platform` | web, ios, android |

| `hour_of_day` | 0-23 |

| `day_of_week` | 0-6 (Monday=0) |

| `is_weekend` | Saturday/Sunday |

  

### Two-Tower Model (Home)

  

#### Architecture

  

```

USER TOWER ITEM TOWER

══════════ ══════════

  

Batch features (18 dims): Batch features (15 dims):

behavior_d30 (4) core attrs (6)

cate_affinity (10: top 5 cates × 2) pdp_views (3)

order_quality (2) ctr (1)

lifetime (2) ratings (2)

demographics (3: encoded) sales (2)

price_segment (1)

Session features (6 dims):

session_view_count (1) ─────────────────────

session_atc_count (1) Embedding layer

session_top_cate (1, embedded) ↓

last_session_top_cate (1, embedded) Dense(128) → ReLU

last_session_view_count (1) ↓

last_session_was_deep (1, bool) Dense(64) → L2 normalize

↓

Request features (4 dims): item_embedding [64]

hour_of_day (1)

day_of_week (1)

is_weekend (1)

platform (1, embedded)

  

─────────────────────

Embedding layer

↓

Dense(128) → ReLU

↓

Dense(64) → L2 normalize

↓

user_embedding [64]

  

LOSS: cosine similarity + in-batch negatives

```

  

- User tower: 18 batch + 6 session + 4 request = **28 features** → 64-dim embedding

- Item tower: **15 features** → 64-dim embedding (pre-computed, stored in FAISS)

  

#### Which features go where

  

| Feature | Model input? | Merge/rank signal? | Why |

|---|---|---|---|

| Batch customer features (18) | User tower | No | Encodes 30-day preferences into embedding |

| Batch product features (15) | Item tower | No | Pre-computed into item embeddings in FAISS |

| Session features (6) | User tower | No | Shifts embedding to reflect current + last session intent |

| Request features (4) | User tower | No | Time/platform context |

| Product trending (4) | No | Yes (Redis trending channel) | Model can't know what's trending now; separate retrieval |

| Cross-features (7) | No | Yes (merge scoring) | Need both user and item at serve time |

| `session_last_product_id` | No | Yes (boost similar) | Boost items similar to most recently viewed |

  

### Training Pipeline (Home)

  

Training runs on BigQuery only. No dependency on Redis or Spark Streaming.

  

#### Session feature reconstruction

  

At serving time, session features come from Redis. During training, the raw event logs in BigQuery contain the same information. We reconstruct what the session features would have been at the moment of each training interaction:

  

```sql

WITH session_features AS (

SELECT

customer_id,

product_id,

event_time,

session_id,

primary_cate_id,

event_type,

  

-- session_view_count: products viewed earlier in this session

COUNT(*) OVER (

PARTITION BY customer_id, session_id

ORDER BY event_time

ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING

) AS session_view_count,

  

-- session_atc_count: ATCs earlier in this session

SUM(CASE WHEN event_type = 'add_to_cart' THEN 1 ELSE 0 END) OVER (

PARTITION BY customer_id, session_id

ORDER BY event_time

ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING

) AS session_atc_count,

  

-- session_top_cate: most viewed cate so far in this session

-- (simplified — in practice use MODE aggregation)

FIRST_VALUE(primary_cate_id) OVER (

PARTITION BY customer_id, session_id

ORDER BY event_time DESC

ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING

) AS session_top_cate

  

FROM `tiki-dwh.recommendation.pdp_view_raw_{yyyymmdd}`

),

  

-- Previous session features: join each session to its predecessor

session_summaries AS (

SELECT

customer_id,

session_id,

COUNT(*) AS total_views,

APPROX_TOP_COUNT(primary_cate_id, 1)[OFFSET(0)].value AS top_cate,

MIN(event_time) AS session_start

FROM `tiki-dwh.recommendation.pdp_view_raw_{yyyymmdd}`

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

FROM session_summaries s

)

  

SELECT

sf.*,

ps.last_session_view_count,

ps.last_session_top_cate,

IFNULL(ps.last_session_view_count >= 5, FALSE) AS last_session_was_deep

FROM session_features sf

LEFT JOIN with_prev_session ps

ON sf.customer_id = ps.customer_id

AND sf.session_id = ps.session_id

```

  

The values from reconstruction won't be byte-identical to Redis at serving time, but they are **statistically consistent** — same distributions, same meaning. The model learns the same signal either way.

  

There is **no need to sync streaming features back to BigQuery for training**. The streaming pipeline writes to Redis for serving only. Raw Kafka events already land in BigQuery via existing  ingestion.

  

```

Kafka → Spark Streaming → Redis (serving only)

  

BigQuery raw events → Window functions → Training features (reconstruction)

```

  

### Serving Pipeline (Home)

  

```

Request: { customer_id: 12345, platform: "ios", source_screen: "home" }

  

Step 1 — Fetch features (parallel)

├─ Feast online store → batch customer features (~10ms, cached)

├─ Redis → session features: session_view_count, session_atc_count,

│ session_top_cate, last_session_top_cate,

│ last_session_view_count, last_session_was_deep (~3ms)

└─ Extract request features from HTTP context (0ms)

  

Step 2 — Compute user embedding

└─ ONNX user tower(28 features) → 64-dim embedding (~5ms)

  

Step 3 — Retrieve candidates (parallel)

├─ Channel A: FAISS ANN(user_embedding, k=1000) → personalized products (~10ms)

└─ Channel B: Redis ZREVRANGE(trending:global + trending:cate:{top_cates}, limit=200)

→ trending products (~2ms)

  

Step 4 — Merge + rank

├─ Union both sets, dedup by product_id

├─ Fetch cross-features from Feast on_demand (~5ms)

├─ Score each candidate:

│ FAISS items: similarity × category_match_boost × brand_match_boost

│ Trending items: trending_score × category_relevance

│ Both sources: items in both get highest combined score

│ Bonus: boost items similar to session_last_product_id

└─ Sort by final score

  

Step 5 — Filter + diversify

├─ Remove: purchased before, not salable

├─ Diversify: max 3 per primary category

└─ Top K (10-50)

  

Total: ~50-80ms P95 (FAISS and Redis trending run in parallel)

```

  

### How Home recommendations change as a user browses

  

**First visit (new session):**

  

1. Session features = zeros, last_session features = from previous visit (if any)

2. User embedding reflects batch history (30-day) + last session context

3. FAISS returns candidates based on long-term preferences + last session carryover

4. Redis trending returns globally trending + trending in user's historical top categories

5. Result: historical preferences + last session continuity + what's hot now

  

**5 products into the session (all electronics):**

  

1. Redis session: `session_view_count=5, session_atc_count=0, session_top_cate=1846 (electronics)`

2. User embedding shifts toward electronics in embedding space

3. FAISS returns a **different 1000 candidates** — more electronics, peripherals, related items

4. Redis trending queries in electronics (cate 1846) + global

5. Result: personalized electronics from FAISS + trending electronics from Redis trending that FAISS wouldn't know about

  

**After adding a laptop to cart:**

  

1. Redis: `session_atc_count=1` — high intent

2. Embedding shifts further toward purchase-intent electronics

3. FAISS: laptop accessories, monitors, peripherals

4. Redis trending: trending laptop accessories (e.g., a flash-sale laptop bag going viral — invisible to FAISS)

5. Filter removes the carted laptop

6. Result: complementary products from both channels

  

**Next day, new session:**

  

1. Last session rolls over: `last_session_top_cate=1846, last_session_view_count=6, last_session_was_deep=true`

2. Current session starts at zero

3. Embedding still carries electronics signal from `last_session_*` even though current session is empty

4. As user starts browsing something else (e.g., books), current session features gradually override

  

---

  

## Part 2: PDP Widgets

  

PDP widgets show products related to the **product being displayed**. The core similarity is pre-computed daily (product-to-product relationships are stable), but trending products in the same or complementary categories are injected from Redis to keep results fresh.

  

### Architecture

  

```

┌─ OFFLINE (daily batch) ─────────────────────────────┐

│ │

│ 1. Product-product similarity │

│ - Interaction similarity (LightFM on session │

│ co-views → product pairs with scores) │

│ - Title similarity (Sentence-BERT on product │

│ names → cosine similarity) │

│ - Combined score with category bonus │

│ │

│ 2. Complementary products │

│ - Cross-category pairs from session co-views │

│ - Filtered by complementary_cate_pairs table │

│ │

│ 3. Write to BigQuery → KVDal │

│ Key: product_id │

│ Value: ranked list of similar/complementary │

│ product_ids with scores │

└──────────────────────────────────────────────────────┘

  

┌─ SERVING (per PDP request) ─────────────────────────┐

│ │

│ Channel A: KVDal lookup by product_id (~3ms) │

│ → pre-computed similar + complementary products │

│ │

│ Channel B: Redis ZREVRANGE (~1-2ms) │

│ → trending:cate:{product's cate_id} │

│ → trending:cate:{complementary cate_ids} │

│ → trending products in relevant categories │

│ │

│ Merge: interleave KVDal results with trending │

│ Filter: is_salable, in_stock, dedup │

│ Return ranked product list │

│ │

│ ~5-10ms P95 │

└──────────────────────────────────────────────────────┘

```

  

### PDP Widget Types

  

| Widget | What it shows | Similarity source | Trending source |

|---|---|---|---|

| **Similar products** | Products like the one being viewed | KVDal: interaction + title similarity (same cate) | Redis: `trending:cate:{same_cate_id}` |

| **Complementary products** | Products that go with the one being viewed | KVDal: cross-category co-view pairs | Redis: `trending:cate:{complementary_cate_ids}` |

| **Infinity scroll** | Extended feed of related products | KVDal: combined similarity | Redis: `trending:cate:{same + complementary cates}` |

  

### How trending helps PDP

  

Product-to-product similarity is **stable** — a laptop's related products don't change hour to hour. But within that stable set of related categories, which specific products are hot changes constantly:

  

- The KVDal pre-computed list says "laptop bags are complementary to laptops" — but it can't tell you which laptop bag is flying off the shelves right now

- Redis `trending:cate:{laptop_bags_cate}` fills that gap — it knows a specific bag has 500 views in the last hour because of a flash sale

- The merge interleaves: pre-computed similar products (relevance) with trending products in relevant categories (freshness)

  

### Training Pipeline (PDP)

  

This uses the existing pipeline pattern (LightFM for interactions, Sentence-BERT replacing TF-IDF/pysparnn for titles):

  

```

BigQuery:

pdp_view_raw (90 days of session co-views)

product_features_v2 (product attributes)

↓

1. Build interaction matrix:

session × product (sessions with 2-15 products, 1-6 categories)

↓

2. Train LightFM → product-product interaction similarity

↓

3. Compute title embeddings (Sentence-BERT) → product-product title similarity

↓

4. Combine:

score = w_interaction × score_interaction

+ w_title × score_title

+ w_cate × score_cate_cooccurrence

↓

5. For each product: top-N similar, top-N complementary

↓

6. Write to BigQuery → KVDal

Key: product_id → Array<{similar_product_id, rank}>

```

  

---

  

## Spark Streaming Scope

  

The streaming job computes **10 features total**: 6 user session features + 4 product trending features. Everything else is batch.

  

### User session state

  

```

Key: customer:{customer_id}:session

TTL: 24h (but logically reset on new session — 30min inactivity gap)

  

Current session:

session_id — current session identifier

session_view_count — products viewed so far

session_atc_count — items added to cart so far

session_cates — array of cate_ids viewed (derive top-1 from this)

session_last_product_id — most recently viewed product

  

Previous session (rolled on session boundary):

last_session_top_cate — top category from previous session

last_session_view_count — view count from previous session

```

  

Session boundary: when a new event arrives and `event_time - last_event_time > 30 minutes`, roll current session into `last_session_*` fields and reset current session.

  

### Product trending

  

```

Key: product:{product_id}:trending_1h

TTL: 2h

Window: 1h tumbling

  

trending_views_1h

trending_atc_1h

trending_purchases_1h

trending_score_1h — weighted: view=1, atc=3, purchase=5

```

  

These feed the Redis sorted sets used by both Home and PDP trending retrieval.

  

### What is NOT computed in streaming

  

| Feature | Why batch is sufficient |

|---|---|

| Sales velocity (24h) | Too sparse for most products. Batch `product_sales_d90` is more stable. |

| Rating momentum | <1 review/day for most products. Batch `product_ratings` sufficient. |

| Brand affinity | Too sparse per session. Batch `customer_brand_affinity` (30d) is reliable. |

| Price sensitivity | Needs multi-session data. Batch `avg_order_value_d30` is a better proxy. |

| User×product interactions | Batch `customer_pdp_view` / `customer_purchased` already covers this. |

| Product-product similarity | Stable relationships, daily recomputation is enough (PDP pipeline). |

  

### Infrastructure

  

```

Kafka (tiki_tracking_v3)

↓

Spark Structured Streaming (GKE, 8 CPU / 16GB)

↓

Redis (session state + trending)

```

  

No ScyllaDB, no BigQuery append from streaming. Training reconstructs session features from BQ raw event logs. Estimated state: ~4GB (2MB product trending + ~4GB active user sessions).

  

---

  

## Complete Data Flow

  

```

┌──────────────────────────────────────────────────────────────┐

│ DATA SOURCES │

│ core_events, nmv, dim_product_full, reviews, │

│ product_impressions, cdp.datasources │

└───────────────────────┬──────────────────────────────────────┘

│

┌─────────────┼──────────────┐

▼ ▼ ▼

┌────────────┐ ┌──────────┐ ┌────────────┐

│ dbt (daily)│ │ Kafka │ │ Airflow │

│ → BQ tables│ │ (live) │ │ (training) │

└─────┬──────┘ └────┬─────┘ └──┬─────┬───┘

│ │ │ │

▼ ▼ │ │

┌────────────┐ ┌──────────┐ │ │

│Feast online│ │ Spark │ │ │

│+ offline │ │ Streaming│ │ │

│(batch feat)│ │ → Redis │ │ │

└─────┬──────┘ └────┬─────┘ │ │

│ │ │ │

▼ ▼ ▼ ▼

┌──────────────────────┐ ┌──────────────────────────────┐

│ HOME SERVING │ │ TRAINING │

│ │ │ │

│ Feast + Redis │ │ Home: Two-Tower model │

│ → ONNX user tower │ │ BQ batch + reconstructed │

│ → FAISS + Redis trending │ │ session features │

│ → merge + filter │ │ │

│ │ │ PDP: LightFM + Sentence-BERT │

│ ~50-80ms P95 │ │ BQ session co-views + │

│ │ │ product features │

└──────────────────────┘ │ │

│ → ONNX + FAISS (Home) │

┌──────────────────────┐ │ → KVDal lists (PDP) │

│ PDP SERVING │ └──────────────────────────────┘

│ │

│ KVDal lookup │

│ + Redis trending │

│ → merge + filter │

│ │

│ ~5-10ms P95 │

└──────────────────────┘

```

  

---

  

## Implementation Phases

  

### Phase 1: Batch-only Two-Tower for Home + improved PDP

  

No streaming dependency. Already better than the current LightFM-for-everything pipeline.

  

1. Build dbt models for all batch feature views

2. **Home**: Train Two-Tower model (28 user features + 15 item features). Session features are reconstructed from BQ for training, default to zero at serving time.

3. **PDP**: Replace TF-IDF/pysparnn with Sentence-BERT for title similarity. Keep LightFM for interaction similarity.

4. Export: ONNX user tower + FAISS index (Home), KVDal similarity lists (PDP)

5. Serve Home with Feast batch features only; session = zeros

6. Serve PDP via KVDal lookup (same pattern as today, better similarity)

7. Use Redis trending sorted sets as second retrieval channel (both Home and PDP)

  

The model is trained with reconstructed session features, so it learns to use them. At serving time they're zero, which the model interprets as "no session context yet" — it falls back to batch-only behavior gracefully.

  

### Phase 2: Add Spark Streaming for Home

  

Adds session-aware personalization to Home widgets. PDP unchanged.

  

1. Deploy Spark streaming: Kafka → session features + trending → Redis

2. Home serving reads session features from Redis instead of defaulting to zero

3. User embedding now shifts as user browses → different FAISS candidates per session

4. Last session carryover provides continuity across visits

  

### Phase 3: Tuning and iteration

  

1. A/B test: FAISS-only vs FAISS+trending vs current LightFM pipeline (Home)

2. A/B test: Sentence-BERT vs pysparnn title similarity (PDP)

3. Tune merge scoring weights (personalized vs trending)

4. Tune category diversification

5. Monitor train/serve skew on session features