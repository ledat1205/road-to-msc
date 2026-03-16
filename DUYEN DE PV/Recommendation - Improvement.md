## Unified Training + Serving Enhancement Roadmap

  

This document covers end-to-end improvements across both the **SmartFlow training pipeline** (Airflow + BigQuery + K8s) and the **smarter-model-serving** inference service (Go + Cassandra/Scylla + Redis).

  

---

  

## 1. Current Architecture

  

### 1.1 Training Pipeline (SmartFlow)

  

| Model | Algorithm | Config | Output |

|-------|-----------|--------|--------|

| Interaction similarity | LightFM (WARP loss, 50d, 50 epochs) → Annoy (100 trees) | No tuning, no temporal decay | `prediction_interaction_similar_YYYYMMDD` (root_product_id level) |

| Title similarity | TF-IDF + pyvi tokenizer → pysparnn KNN (k=100, per-category) | Bag-of-words, no semantic | `prediction_title_similar_YYYYMMDD` (product_id level) |

| User propensity | XGBoost (100 trees, lr=0.1, 4 features) | Binary per-cate | `propensity_cate_YYYYMMDD`, `combine_propensity` |

| Score combining | SQL: `max(score_interaction, score_title)` + `score_cate` (6-level hierarchy match) | Hardcoded weights | `product_similar_prediction_v2_YYYYMMDD` |

| Home ranking | SQL: 4 streams (propensity, recent views, similar-to-viewed, top popular) + NTILE batching `500 - 2*batch + score` | Hardcoded | Pre-computed per-user lists → Cassandra |

| PDP ranking | SQL: root→product mapping, filter rating>3, cheapest SKU per root, `score_root = max_score_model + total_cate_score` | Hardcoded | Pre-computed per-product lists → Cassandra |

  

### 1.2 Serving Layer (smarter-model-serving)

  

**Architecture**: Go gRPC service → Cassandra (Scylla) for pre-computed recommendations + Redis for user behavior metrics

  

**Home widget flow** (query by `user_id`):

```

Request(customer_id, trackity_id)

→ KeysBuilder: [customer_id, trackity_id, "_" (popular)]

→ Cassandra Multi-Get (parallel, table: p_reco_<version>)

→ Decode (FlatBuffer v3 or Protobuf v2)

→ BuildFinalSuggestList (dedup by product_id, priority: customer > trackity > popular)

→ SimpleRank reranking (if BlockFlagRotate enabled):

- Fetch UserBrowsing from Redis (impressions, clicks, CTR per product)

- Split into good (has clicks or < 2 impressions) vs bad (≥2 impressions, 0 clicks)

- Sort good by CTR desc, append bad sorted by impression asc

→ Return top-N ItemRank list

```

  

**PDP widget flow** (query by `product_id`):

```

Request(customer_id, mpid)

→ Keys: ["{customer_id}_{mpid}", "{trackity_id}_{mpid}", "_{mpid}"]

→ Cassandra Multi-Get (same pattern)

→ Decode + Dedup

→ SimpleRank reranking (same CTR-based logic)

→ Return top-N similar products

```

  

**Key limitations**:

- **Pure lookup service**: No computation at serving time, just Cassandra reads + light reranking

- **Stale recommendations**: All personalization is pre-computed daily; a user's actions during the day only affect reranking (demote seen items), not the candidate set

- **Cold start**: Falls back from customer_id → trackity_id → popular ("_"), but popular is the same for everyone

- **No feature awareness**: Serving layer knows nothing about product attributes, only pre-computed rank lists

- **Reranking is primitive**: Binary good/bad split by impression count, CTR sorting. No learned model.

  

---

  

## 2. Improvement Phases

  

### Phase 1: Quick Wins — Training Only (Weeks 1-3)

  

No serving changes needed. Improves quality of pre-computed lists in Cassandra.

  

#### 1A. Hyperparameter Tuning for LightFM

  

**Current**: Hardcoded `no_components=50, epochs=50, loss='warp'` in `train/script/view_together_mf.py`

  

**Change**:

- Add Optuna sweep in training K8s pod

- Search space: `no_components=[32,64,128]`, `learning_rate=[0.01,0.05,0.1]`, `loss=['warp','bpr']`, `epochs=[30,50,80]`

- Validation: Hold out last 7 days of sessions, measure Recall@50

- Store best params in GCS, load on next run

  

**Files to modify**:

- `Olympus/recommendation/train/script/view_together_mf.py` — Add Optuna optimization loop

- `Olympus/recommendation/train/params/train.yml` — Add search space config

  

**Serving impact**: None. Same output format, better quality.

  

#### 1B. Temporal Decay on Session Co-views

  

**Current**: `sale_orders` treats all 90-day sessions equally

  

**Change**: Weight recent sessions exponentially higher

```sql

-- In input_data_train_interaction.sql

EXP(-0.03 * DATE_DIFF(CURRENT_DATE(), session_date, DAY)) AS session_weight

```

Pass weights to LightFM via `sample_weight` parameter.

  

**Files to modify**:

- `train/sql/train_similar_interaction_v2/input_data_train_interaction.sql`

- `train/script/view_together_mf.py`

  

**Serving impact**: None.

  

#### 1C. Enrich User Propensity Features (4 → 15+ features)

  

**Current**: Only `eventView3M/1M`, `eventPurchase3M/1M`

  

**Add** (already available in aggregation tables):

- `days_since_last_purchase_in_cate` (from nmv)

- `avg_order_value_in_cate` (from nmv)

- `num_distinct_products_viewed_in_cate` (from event_staging)

- `impression_ctr_in_cate` (from product_impressions)

- `num_add_to_cart_in_cate` (from event_staging)

- `total_spend_all_cates`, `num_active_cates` (cross-cate signals)

- `avg_rating_of_viewed_products` (from review)

- `avg_price_of_viewed_products` (from product_features_v2)

- `days_since_first_activity` (user tenure)

  

**Files to modify**:

- `train/sql/train_user_favorite_cate/purchase_feature.sql`

- `train/sql/train_user_favorite_cate/view_feature.sql`

- `train/sql/train_user_favorite_cate/data_train.sql`

  

**Serving impact**: None. Same `combine_propensity` output, better propensity scores.

  

---

  

### Phase 2: Semantic Title + Learned Ranker (Weeks 4-7)

  

#### 2A. Sentence-BERT for Title Similarity

  

**Current**: TF-IDF + pyvi tokenizer → pysparnn (bag-of-words, per-category isolation)

- Misses semantic similarity: "áo khoác gió" vs "jacket chắn gió"

- Per-category isolation prevents cross-category discovery

  

**Replace with**: Multilingual Sentence-BERT (`paraphrase-multilingual-MiniLM-L12-v2`, 384d)

```

Product titles → Sentence-BERT batch encode → 384d embeddings

→ FAISS index (IVF4096,PQ48) → top-100 similar per product

```

  

**Hybrid scoring**: Keep TF-IDF for exact-match signal

```python

score_title = 0.4 * tfidf_score + 0.6 * semantic_score

```

  

**New files**:

- `Olympus/recommendation/train/script/title_similarity_sbert.py`

- Add `sentence-transformers`, `faiss-cpu` to `requirements.txt`

  

**Serving impact**: None. Same `prediction_title_similar_YYYYMMDD` output format.

  

#### 2B. Learned Ranker (Replace Rule-Based Reranking)

  

This is the first change that touches the **serving layer**.

  

**Current serving reranking** (`metrics_ranking.go`):

```go

// Binary split: good (has clicks or <2 impressions) vs bad (≥2 impressions, 0 clicks)

// Sort good by CTR desc, append bad

```

  

**Replace with**: LightGBM LambdaRank model, served via ONNX in Go

  

**Training side** (new SmartFlow DAG tasks):

```

impression/click logs → training pairs → LightGBM lambdarank

Features per (user, item) pair:

- score_interaction, score_title, propensity_score (from existing models)

- product features: price, rating, CTR, days_since_launch, num_sold_d30

- user×product features: user_cate_propensity for this product's cate

- position bias: log(position + 1) as bias feature

Output: ONNX model artifact → GCS

```

  

**New SmartFlow files**:

- `Olympus/recommendation/train/script/learn_to_rank.py` — Train LambdaRank

- `Olympus/recommendation/train/sql/ranking_training_data.sql` — Prepare impression/click pairs

- Pipeline task to export ONNX model to GCS

  

**Serving side changes** (`smarter-model-serving`):

  

```

New component: OnnxRanker

- Loads LambdaRank ONNX model from GCS at startup (reload daily)

- On each request: score candidate items using model features

- Replace SimpleRank in GeneralSuggestHandler.Get()

```

  

**Files to modify in smarter**:

- `pkg/coremetrics/metrics_ranking.go` — Add `LearnedRank()` method alongside existing `SimpleRank()`

- `handlers/v1/grpc/general_suggest_handler.go` — Call `LearnedRank()` when block flag enabled

- New: `pkg/ranking/onnx_ranker.go` — ONNX Runtime wrapper in Go

- New: `pkg/ranking/feature_builder.go` — Build feature vector from item metadata + user metrics

  

**Feature serving for the ranker** requires enriching what's stored:

- **Cassandra**: Store item features alongside rank (extend FlatBuffer schema to include price, rating, CTR, cate_id, propensity for that cate)

- **Redis**: Already has user browsing metrics; add user propensity scores per-cate

  

**Rollout**: Use existing `BlockFlag` mechanism to A/B test:

```go

if blockFlag.Bool(common.BlockFlagLearnedRank) {

items = u.LearnedRank(ctx, items, userMetrics)

} else {

items = rerankProduct(userMetrics, items, blockFlag)

}

```

  

---

  

### Phase 3: Two-Tower Model + Real-Time User Embeddings (Weeks 8-15)

  

This is the **biggest architectural change** — transforms the serving layer from a pure lookup service to a real-time inference service.

  

#### 3A. Two-Tower Model (Training Side)

  

**Replace LightFM** (which only uses session×product matrix) with a neural two-tower that incorporates all available features.

  

```

User Tower: Item Tower:

├─ viewed_product_embeddings (avg) ├─ product_id embedding (64d)

├─ viewed_cate_embeddings (avg) ├─ cate_id embedding (32d)

├─ propensity_scores (per-cate) ├─ seller_id embedding (32d)

├─ purchase_history_embedding ├─ price (log-normalized)

├─ days_since_last_view ├─ rating, num_reviews

└─ user_segment features ├─ CTR, view_count, sold_count

├─ product_tier

↓ └─ business_type, brand flags

Dense(256) → Dense(128) ↓

↓ Dense(256) → Dense(128)

user_embedding (64d) ↓

↓ item_embedding (64d)

└────── dot product ─────────────┘

```

  

**Training**: In-batch negatives on session co-views. Framework: PyTorch.

  

**Daily Airflow output**:

1. All item embeddings (64d per product) → exported to **Cassandra** (new table: `item_embeddings_<version>`)

2. FAISS index over item embeddings → exported to **GCS**

3. User tower model weights → exported as **ONNX** to GCS

4. Pre-computed user embeddings for active users → **Cassandra** (new table: `user_embeddings_<version>`) as warm cache

  

**New SmartFlow files**:

- `Olympus/recommendation/train/script/two_tower_model.py` — Model definition + training

- `Olympus/recommendation/train/script/export_embeddings.py` — Export to Cassandra + FAISS + ONNX

- New DAG or extend `recommendation_train_interaction_v2`

  

#### 3B. Real-Time User Embeddings (Serving Side)

  

**The key insight**: Item embeddings are precomputed daily (expensive, millions of products). User embeddings are computed **on-the-fly** at request time (cheap, one user).

  

##### Sub-phase 3B-i: Weighted Average (No Model Serving)

  

Simplest approach, no ONNX needed. Approximate user embedding as weighted average of recently viewed item embeddings.

  

**New serving flow for Home widget**:

```

Request(customer_id)

→ Fetch user's recent actions from Redis (last 20 product_ids + timestamps + action types)

→ Batch-fetch item embeddings from Cassandra (new table: item_embeddings)

→ Compute: user_emb = weighted_avg(item_embs)

weights: exp(-0.1 * hours_ago) * action_weight (purchase=3x, ATC=2x, view=1x)

→ L2 normalize user_emb

→ Query FAISS index for top-200 nearest items

→ Apply learned ranker (Phase 2B) for final ordering

→ Return top-N

```

  

**Files to modify in smarter**:

- New: `pkg/embedding/embedding_store.go` — Cassandra reader for item embeddings

- New: `pkg/embedding/user_embedding.go` — Weighted-average user embedding computation

- New: `pkg/embedding/faiss_index.go` — FAISS index loaded from GCS, queried in-process

- `service/data_service.go` — New `GetWithEmbeddings()` method

- `handlers/v1/grpc/general_suggest_handler.go` — Route to embedding-based flow when model version indicates two-tower

  

**PDP widget** doesn't change much — still lookup by product_id in Cassandra. But now the similar products come from FAISS nearest-neighbor search on item embeddings instead of pre-computed lists.

  

**New infrastructure needed**:

- **Redis**: Store user action history (last 50 actions per user). Consume from existing Kafka topics (`trackity.tiki.v2.core_events`, `trackity_tiki.product_impressions`)

- **Cassandra**: New table `item_embeddings_<version>` (product_id → 64d float vector, ~2.5M rows × 256 bytes)

- **In-process FAISS**: Load FAISS index into serving pod memory (~500MB for 2.5M × 64d with IVF4096,PQ48). Reload daily from GCS.

  

**User action ingestion** (new microservice or sidecar):

```

Kafka (trackity.tiki.v2.core_events)

→ Simple Go consumer

→ Parse: user_id, product_id, action_type, timestamp

→ Redis LPUSH: user:{user_id}:actions → {product_id, action_type, timestamp}

→ Redis LTRIM: Keep last 50 actions

```

  

##### Sub-phase 3B-ii: ONNX User Tower (Better Accuracy)

  

Once weighted-average is validated, upgrade to running the actual user tower network.

  

**Change**: Replace weighted-average computation with ONNX Runtime inference

```go

// In user_embedding.go

func (s *UserEmbeddingService) ComputeUserEmbedding(actions []UserAction) []float32 {

// Build input tensor from recent actions + item embeddings

input := s.buildInputTensor(actions)

// Run user tower ONNX model (~1-2ms)

output, _ := s.onnxSession.Run(input)

return output[0].([]float32)

}

```

  

**New dependency**: `github.com/yalue/onnxruntime_go` (Go ONNX Runtime bindings)

  

**Serving pod resources**: Increase memory request from current defaults to accommodate FAISS index + ONNX model in memory.

  

##### Sub-phase 3B-iii: Streaming Embedding Updates (Optional, If Latency Matters)

  

Pre-compute user embeddings on every action via Kafka consumer + Flink/Beam job, cache in Redis. Serving just reads the cached embedding — zero compute at request time.

  

```

Kafka (core_events) → Flink job → Redis (user:{id}:embedding → 64d vector)

```

  

Only pursue this if 3B-ii latency (FAISS query + ONNX inference ~5-10ms) is too high for SLA.

  

#### 3C. Item Tower Freshness

  

Item embeddings don't need true real-time, but should handle new/changed products:

  

| Scenario | Solution |

|----------|----------|

| New product (no embedding) | Compute item embedding on-the-fly via item tower ONNX + product features from cache |

| Price/stock/rating change | Hourly micro-batch: re-run item tower on changed products, update Cassandra + FAISS |

| Product deactivated | TTL on Cassandra entries + hourly FAISS rebuild |

  

**New SmartFlow DAG**: `recommendation_embedding_refresh` (hourly, lightweight)

- Query products with `updated_at > 1 hour ago`

- Run item tower ONNX on changed products

- Update Cassandra `item_embeddings` table

- Rebuild FAISS index and upload to GCS

  

---

  

### Phase 4: Sequential Recommendation (Weeks 16+)

  

#### 4A. SASRec Model

  

**Current**: Home ranking treats user history as a bag (unordered set of viewed products)

  

**Replace with**: Self-Attentive Sequential Recommendation (SASRec)

- Input: Last 50 user actions in timestamp order from `pdp_view_raw`

- Architecture: Causal self-attention (2 layers, 2 heads, 64d) → next-item prediction

- Captures: "user viewed phone → phone case → screen protector" sequential patterns

  

**Training**: Daily in SmartFlow, same K8s pod pattern. Export as ONNX.

  

**Serving**: Runs at request time in smarter-model-serving:

```

Recent user actions (from Redis) → SASRec ONNX model → predicted next-item embedding

→ FAISS top-K → Learned ranker → Response

```

  

Can be combined with two-tower: use SASRec output as additional user tower input feature.

  

---

  

## 3. Serving Architecture Evolution

  

### Current State

```

SmartFlow (daily) → BigQuery → Cassandra (pre-computed lists)

↓

smarter-model-serving (pure lookup + light rerank)

↓

Response (top-N items)

```

  

### After Phase 2 (Learned Ranker)

```

SmartFlow (daily) → BigQuery → Cassandra (pre-computed lists)

→ GCS (ONNX ranker model)

↓

smarter-model-serving

├─ Cassandra lookup (candidates)

├─ Redis (user browsing metrics)

└─ ONNX ranker (reranking) ← NEW

↓

Response (top-N items)

```

  

### After Phase 3 (Two-Tower + Real-Time)

```

SmartFlow (daily) → Cassandra (item embeddings)

→ GCS (FAISS index + user tower ONNX)

  

Kafka (real-time) → Redis (user action history) ← NEW

  

smarter-model-serving

├─ Redis (recent user actions) ← NEW

├─ Cassandra (item embeddings) ← NEW

├─ FAISS (ANN search, in-process) ← NEW

├─ ONNX user tower (user embedding) ← NEW

└─ ONNX ranker (final reranking)

↓

Response (top-N items)

```

  

### Key Architectural Decisions

  

| Decision | Choice | Rationale |

|----------|--------|-----------|

| FAISS in-process vs external service | In-process | Avoids network hop. 500MB for 2.5M×64d index fits in pod memory. Reload daily from GCS. |

| User action store | Redis (existing infra) | Already used for UserBrowsing metrics. Add action list per user. Low latency. |

| Embedding store | Cassandra (existing infra) | Same Scylla cluster already serving recommendations. Add new table. |

| Model serving | ONNX Runtime in Go | Avoid separate model serving infra (TF Serving, Triton). Go bindings exist. <5ms inference. |

| A/B testing | BlockFlag mechanism (existing) | Already supports feature flags per block/widget. Add flags for each phase. |

  

---

  

## 4. Changes Summary Per Repository

  

### SmartFlow (training pipeline)

  

| Phase | New Files | Modified Files |

|-------|-----------|----------------|

| 1A | `train/params/train.yml` | `train/script/view_together_mf.py` |

| 1B | — | `train/sql/.../input_data_train_interaction.sql`, `train/script/view_together_mf.py` |

| 1C | — | `train/sql/train_user_favorite_cate/{purchase,view}_feature.sql`, `data_train.sql` |

| 2A | `train/script/title_similarity_sbert.py` | `requirements.txt` |

| 2B | `train/script/learn_to_rank.py`, `train/sql/ranking_training_data.sql` | — |

| 3A | `train/script/two_tower_model.py`, `train/script/export_embeddings.py` | — |

| 3C | New DAG: `reco_embedding_refresh.py` | — |

| 4A | `train/script/sasrec_model.py` | — |

  

### smarter-model-serving (inference)

  

| Phase | New Files | Modified Files |

|-------|-----------|----------------|

| 2B | `pkg/ranking/onnx_ranker.go`, `pkg/ranking/feature_builder.go` | `handlers/.../general_suggest_handler.go`, `pkg/coremetrics/metrics_ranking.go` |

| 3B-i | `pkg/embedding/embedding_store.go`, `pkg/embedding/user_embedding.go`, `pkg/embedding/faiss_index.go` | `service/data_service.go`, `handlers/.../general_suggest_handler.go`, `config/db.go` |

| 3B-ii | — | `pkg/embedding/user_embedding.go` (swap weighted-avg for ONNX) |

| 4A | `pkg/sequential/sasrec_runner.go` | `pkg/embedding/user_embedding.go` |

  

### New Infrastructure

  

| Phase | Component | Purpose |

|-------|-----------|---------|

| 3B | Kafka consumer (Go microservice or sidecar) | Ingest user actions → Redis |

| 3B | Redis: `user:{id}:actions` list | Store last 50 user actions per user |

| 3A | Cassandra: `item_embeddings_<version>` table | Store 64d item embeddings |

| 3B-i | FAISS index in serving pod memory (~500MB) | ANN search over item embeddings |

| 3B-ii | ONNX Runtime in serving pod | Run user tower model |

| 3B-iii (optional) | Flink job | Streaming user embedding pre-computation |

  

---

  

## 5. Evaluation Strategy

  

### Offline Metrics (Gate Each Phase)

  

| Metric | How | Target |

|--------|-----|--------|

| Recall@K | Hold-out last-day sessions, check if clicked/purchased items in top-K | +5% per phase |

| NDCG@K | Rank quality on held-out impression/click data | +3% per phase |

| Coverage | % of salable products in any user's top-120 | ≥ current |

| Latency (serving) | p50/p95/p99 of gRPC response time | p99 < 50ms (Phase 1-2), < 100ms (Phase 3+) |

  

### Online A/B Tests

  

- Use existing Tiki experiment platform + BlockFlag mechanism

- Minimum 7-day test per phase

- Primary metric: Widget CTR (Home "You May Like", PDP "Similar Products")

- Secondary: Conversion rate from reco widgets, GMV per session

- Ship if: CTR +2% AND conversion +1% (statistically significant)

  

### Monitoring

  

All phases feed into the existing `recommendation_monitoring_v2` DAG:

- Phase 1-2: Existing model output quality checks cover new model outputs (same table format)

- Phase 3: Add embedding quality checks (coverage, norm distribution, drift)

- Phase 3B: Add serving latency monitoring (p50/p95/p99 per endpoint)

  

---

  

## 6. Risk Mitigation

  

| Risk | Mitigation |

|------|------------|

| Two-tower model worse than LightFM | Keep LightFM running in parallel. A/B test before switching. BlockFlag to roll back instantly. |

| FAISS index too large for pod memory | Use IVF + PQ compression. 2.5M × 64d with PQ48 ≈ 500MB. If still too large, use FAISS server as sidecar. |

| ONNX Runtime instability in Go | Extensive load testing before production. Fallback to weighted-average (3B-i) if ONNX crashes. |

| Kafka consumer lag → stale user actions | Monitor consumer lag. Acceptable: up to 5 min lag. Set TTL on Redis action lists (24h). |

| Serving latency regression | All new computation paths behind BlockFlags. Monitor p99 latency. Auto-disable if p99 > 200ms. |

| Daily FAISS rebuild causes serving disruption | Atomic swap: build new index in background, swap pointer. No downtime. |

  

---

  

## 7. Feature Store Architecture

  

### 7.1 The Problem with Current Approach

  

Today, **all features are computed in BigQuery batch SQL** (daily aggregation DAG). This creates several issues:

  

1. **Training-serving skew**: Training uses BigQuery features computed at T-1. Serving uses pre-computed recommendation lists from T-1. But user behavior at serving time is T+0. The learned ranker (Phase 2B) and two-tower user embedding (Phase 3B) need features at request time that match what the model was trained on.

  

2. **Redundant computation**: The same features (e.g., product view count, user purchase count) are computed from scratch every day in BigQuery, even though only the delta changed. At ~24 SQL queries over 9 source tables, this is expensive.

  

3. **No point-in-time correctness**: Training on `product_features_v2_20260316` uses today's aggregated features as if they were the features at the time each interaction happened. A product that had 0 views 30 days ago but 10K views today gets the 10K-view feature for all historical training rows — this is label leakage.

  

4. **Feature duplication**: The same raw signal (e.g., "user X viewed product Y") is stored in `core_events`, aggregated into `pdp_view_raw`, re-aggregated into `product_features_pdp_view`, then joined into `product_features_v2`. Each layer recomputes from scratch daily.

  

### 7.2 What Features Exist and How Fast They Change

  

Every feature in the system falls into one of these freshness tiers:

  

#### Tier 1: Static / Slow (changes days to weeks)

These rarely change. Daily batch is more than sufficient.

  

| Feature | Current Source | Change Frequency |

|---------|---------------|-----------------|

| Product name, category (cate1-6), seller_id | `dim_product_full` | Days (catalog updates) |

| Product tier (price segment) | `personas.product_tier` | Weeks (manually maintained) |

| Category tree structure | `dim_product_full` | Months |

| Business type (1P/3P), brand, is_official_store | `dim_product_full` | Rarely |

| Complementary cate pairs | `vw_related_product_v2` | Manually, outdated |

  

**Collection**: Daily batch from BigQuery (keep current approach). Sync to online store once/day.

  

#### Tier 2: Medium (changes hours)

These accumulate through the day. Hourly or near-real-time incremental updates provide significant freshness improvement.

  

| Feature | Current Source | Change Frequency | Impact of Staleness |

|---------|---------------|-----------------|---------------------|

| Product sale_price, stock status, is_salable | `dim_product_full` | Hours (flash sales, stock-outs) | Recommending out-of-stock or wrong-price items |

| Product avg_rating, num_reviews | `ecom.review` | Hours (new reviews) | Low impact |

| Product num_sold_d30, num_sold_d7 | `nmv.nmv` | Hours (new orders) | Trending products missed until tomorrow |

| Product impression CTR (ctr_root_d30) | `product_impressions` | Hours | Moderate — CTR shifts with traffic patterns |

| Product view count (num_root_view_d7/d30) | `core_events` + `trackity_tiki_click` | Hours | High — new/trending products invisible until tomorrow |

| User propensity per category | `combine_propensity` | Daily (model output) | Moderate |

| Product embeddings (two-tower item tower) | Model output | Daily (model output) | Low for existing products; high for new products (no embedding) |

  

**Collection**: Incremental aggregation. Two approaches:

  

**Option A — Hourly micro-batch in BigQuery** (simpler):

```

New DAG: recommendation_feature_refresh (hourly)

→ Incremental SQL: only process events since last run

→ Update product_features_v2 for changed products

→ Sync changed features to online store (Cassandra/Redis)

```

  

**Option B — Streaming aggregation via Kafka + Flink** (more infra, lower latency):

```

Kafka (core_events, product_impressions, nmv)

→ Flink stateful aggregation (sliding windows: 1d, 7d, 30d)

→ Write to online store (Redis) in real-time

→ Periodically snapshot to offline store (BigQuery) for training

```

  

#### Tier 3: Fast (changes per-minute / per-request)

These change with every user action. Must be real-time.

  

| Feature | Current Source | Change Frequency | Used By |

|---------|---------------|-----------------|---------|

| User's last N viewed products | `core_events` (batch) / Redis `UserBrowsing` (partial) | Per-action | Two-tower user embedding, SASRec, learned ranker |

| User's last N clicked products | `trackity_tiki_click` (batch) / Redis (partial) | Per-action | Learned ranker, reranking |

| User's last N purchased products | `nmv` (batch) | Per-action | Filtering already-bought items |

| User's last N add-to-cart products | `core_events` (batch) | Per-action | Cart-based recommendations (currently disabled) |

| User's impression count per product | Redis `UserBrowsing` (already real-time) | Per-impression | Current reranking (good/bad split) |

| User's click count per product | Redis `UserBrowsing` (already real-time) | Per-click | Current reranking (CTR sort) |

| User's current session context | Not captured | Per-action | Session-based cold start |

  

**Collection**: Real-time via Kafka consumer → Redis (already partially exists for `UserBrowsing`).

  

### 7.3 Proposed Feature Store Design

  

```

┌─────────────────────────────────────────────────────────────────────┐

│ DATA SOURCES │

│ Kafka Topics: BigQuery Tables: │

│ • trackity.tiki.v2.core_events • dim_product_full │

│ • trackity_tiki.product_impr. • ecom.review │

│ • tiki_events_sessions • nmv.nmv │

│ • (new) order_events • personas.product_tier │

└────────┬──────────────────────────────────┬──────────────────────────┘

│ │

▼ ▼

┌─────────────────────┐ ┌─────────────────────────┐

│ STREAMING LAYER │ │ BATCH LAYER │

│ (Kafka Consumer / │ │ (Airflow DAGs) │

│ Flink / Go svc) │ │ │

│ │ │ Daily: │

│ Real-time: │ │ • Full feature rebuild │

│ • User action log │ │ • Model training │

│ → Redis │ │ • Embedding generation │

│ • User browsing │ │ │

│ metrics → Redis │ │ Hourly: │

│ (already exists) │ │ • Incremental feature │

│ │ │ refresh for changed │

│ Near-real-time: │ │ products │

│ • Product view/ │ │ • New product embedding │

│ sold counters │ │ computation │

│ → Redis │ │ │

└─────────┬───────────┘ └────────────┬────────────┘

│ │

▼ ▼

┌─────────────────────────────────────────────────────────────────────┐

│ FEATURE STORES │

│ │

│ ┌─────────────────────────────┐ ┌──────────────────────────────┐ │

│ │ ONLINE STORE │ │ OFFLINE STORE │ │

│ │ (Serving time features) │ │ (Training time features) │ │

│ │ │ │ │ │

│ │ Redis: │ │ BigQuery: │ │

│ │ • user:{id}:actions │ │ • product_features_v2_YMD │ │

│ │ (last 50 actions, RT) │ │ • customer_pdp_view_YMD │ │

│ │ • user:{id}:browsing │ │ • all 24 agg tables │ │

│ │ (impressions/clicks, RT) │ │ • training labels │ │

│ │ • user:{id}:propensity │ │ • feature snapshots (for │ │

│ │ (per-cate scores, daily) │ │ point-in-time training) │ │

│ │ • product:{id}:counters │ │ │ │

│ │ (views/sold today, NRT) │ │ GCS: │ │

│ │ │ │ • Model artifacts │ │

│ │ Cassandra (Scylla): │ │ • FAISS indices │ │

│ │ • item_embeddings │ │ • ONNX models │ │

│ │ (64d vectors, daily │ │ │ │

│ │ + hourly for new items) │ │ │ │

│ │ • product_features_online │ │ │ │

│ │ (price, rating, CTR, │ │ │ │

│ │ cate, seller — for │ │ │ │

│ │ learned ranker) │ │ │ │

│ │ • p_reco_<version> │ │ │ │

│ │ (pre-computed lists, │ │ │ │

│ │ kept as fallback) │ │ │ │

│ │ │ │ │ │

│ │ In-Process (serving pod): │ │ │ │

│ │ • FAISS index (~500MB) │ │ │ │

│ │ • ONNX models (<100MB) │ │ │ │

│ └─────────────────────────────┘ └──────────────────────────────┘ │

│ │

│ ┌─────────────────────────────────────────────────────────────────┐ │

│ │ SYNC MECHANISMS │ │

│ │ │ │

│ │ Online → Offline (for training): │ │

│ │ • Daily: Snapshot Redis counters → BigQuery staging table │ │

│ │ • Daily: Snapshot user action logs → BigQuery for training │ │

│ │ • Purpose: Point-in-time correct training data │ │

│ │ │ │

│ │ Offline → Online (for serving): │ │

│ │ • Daily: BigQuery product_features → Cassandra product_features│ │

│ │ • Daily: Model embeddings → Cassandra + GCS │ │

│ │ • Daily: User propensity scores → Redis │ │

│ │ • Hourly: Changed product features → Cassandra │ │

│ │ • Hourly: New product embeddings → Cassandra + FAISS │ │

│ └─────────────────────────────────────────────────────────────────┘ │

└─────────────────────────────────────────────────────────────────────┘

```

  

### 7.4 Feature Registry: What Goes Where

  

Complete mapping of every feature to its store, freshness, and consumer:

  

#### Product Features

  

| Feature | Offline (BigQuery) | Online (Cassandra) | Online (Redis) | Freshness | Training Consumer | Serving Consumer |

|---------|---|---|---|---|---|---|

| product_id, name, cate1-6 | `product_features_v2` | `product_features_online` | — | Daily | Two-tower item tower, title similarity | Learned ranker |

| seller_id, business_type, brand | `product_features_v2` | `product_features_online` | — | Daily | Two-tower item tower | Learned ranker |

| sale_price | `product_features_v2` | `product_features_online` | — | Hourly* | Two-tower item tower, ranker training | Learned ranker |

| is_salable, stock_status | `dim_product_full` | `product_features_online` | — | Hourly* | Filtering | Candidate filtering |

| avg_rating, num_reviews | `product_features_v2` | `product_features_online` | — | Daily | Two-tower item tower | Learned ranker |

| num_root_view_d7/d30 | `product_features_v2` | `product_features_online` | `product:{id}:counters` (today's delta) | Daily + NRT delta | Two-tower item tower | Learned ranker (daily base + NRT delta) |

| num_sold_root_d30 | `product_features_v2` | `product_features_online` | `product:{id}:counters` | Daily + NRT delta | Two-tower item tower | Learned ranker |

| ctr_root_d30 | `product_features_v2` | `product_features_online` | — | Daily | Two-tower item tower, ranker | Learned ranker |

| product_tier | `product_features_v2` | `product_features_online` | — | Weekly | Two-tower item tower | Learned ranker |

| item_embedding (64d) | GCS (model artifact) | `item_embeddings` + FAISS | — | Daily + hourly for new | — | FAISS ANN search, user embedding computation |

| title_embedding (384d) | GCS (model artifact) | — | — | Daily | — | Not served directly (baked into similarity scores) |

  

*Hourly refresh only for products with detected changes (price update, stock-out event).

  

#### User Features

  

| Feature | Offline (BigQuery) | Online (Redis) | Freshness | Training Consumer | Serving Consumer |

|---------|---|---|---|---|---|

| Last 50 actions (product_id, action_type, ts) | Daily snapshot → `user_action_log_YYYYMMDD` | `user:{id}:actions` | Real-time (Kafka → Redis) | Two-tower user tower, SASRec, ranker | User embedding computation (Phase 3B), SASRec (Phase 4) |

| Impression count per product | — | `browsing_{key}` → `m_p_{pid}` (already exists) | Real-time (already exists) | Ranker training (position bias) | Current reranking, learned ranker |

| Click count per product | — | `browsing_{key}` (already exists) | Real-time (already exists) | Ranker training | Current reranking, learned ranker |

| CTR per product | — | Derived from above (already exists) | Real-time | Ranker training | Current reranking, learned ranker |

| Impression count per category | — | `browsing_{key}` (already exists) | Real-time (already exists) | — | Category group reranking |

| Propensity per primary cate | `propensity_cate_YYYYMMDD` | `user:{id}:propensity` | Daily (model output) | Two-tower user tower input | Two-tower user tower, learned ranker |

| Purchase history (product_ids) | `customer_purchased_YYYYMMDD` | `user:{id}:actions` (filtered) | RT for new; daily for full history | — | Already-bought filtering |

| user_embedding (64d) | `user_embeddings` (warm cache) | `user:{id}:embedding` (Phase 3B-iii) or computed on-the-fly | RT (on-the-fly) or NRT (streaming) | — | FAISS ANN query |

| Customer-client mapping | `map_client_customer_YYYYMMDD` | Serving uses customer_id from auth | Daily | — | Key lookup priority |

  

#### Interaction Features (Training Only)

  

| Feature | Store | Freshness | Consumer |

|---------|-------|-----------|----------|

| Session co-views (product pairs in same session) | `sale_orders_YYYYMMDD` (BigQuery) | Daily | LightFM / Two-tower training |

| Session co-views with timestamp + weight | New: `weighted_session_coviews_YYYYMMDD` | Daily | Two-tower training (with temporal decay) |

| Impression → click pairs (for ranker) | New: `ranker_training_data_YYYYMMDD` | Daily | LambdaRank training |

| User action sequences (ordered by time) | New: `user_action_sequences_YYYYMMDD` | Daily | SASRec training |

  

### 7.5 Data Collection Strategy Per Signal

  

#### Real-Time Collection (Kafka → Redis)

  

Already exists for `UserBrowsing` metrics. Extend to cover:

  

```

┌─────────────────────────────────────────────────────┐

│ Kafka Consumer Service (Go, new or extend existing) │

│ │

│ Input Topics: │

│ • trackity.tiki.v2.core_events │

│ • trackity_tiki.product_impressions │

│ │

│ Outputs: │

│ │

│ 1. User Action Log (NEW) │

│ Redis: LPUSH user:{id}:actions │

│ → {product_id, action_type, timestamp} │

│ → LTRIM to keep last 50 │

│ → TTL: 7 days │

│ │

│ 2. User Browsing Metrics (ALREADY EXISTS) │

│ Redis: HSET browsing_{key} m_p_{pid} ... │

│ → impressions, clicks, CTR per product │

│ │

│ 3. Product Real-Time Counters (NEW) │

│ Redis: HINCRBY product:{id}:counters │

│ → views_today, clicks_today, sold_today │

│ → TTL: 25 hours (reset with daily batch) │

│ │

│ Event filtering: │

│ • view_pdp → user actions + product view counter │

│ • true_impression → user browsing + product counter │

│ • click → user browsing + product counter │

│ • add_to_cart → user actions │

│ • complete_purchase → user actions + product counter │

└─────────────────────────────────────────────────────┘

```

  

#### Near-Real-Time / Hourly Batch

  

New lightweight Airflow DAG: `recommendation_feature_refresh` (hourly)

  

```

Tasks:

1. Detect changed products (price, stock, salable status)

→ Query dim_product_full WHERE updated_at > last_run

→ Update Cassandra product_features_online for changed rows

  

2. Compute embeddings for new products

→ Query product_features_v2 WHERE product_id NOT IN item_embeddings

→ Run item tower ONNX → write to Cassandra item_embeddings

→ Rebuild FAISS index, upload to GCS

→ Signal serving pods to reload (or periodic reload every 1h)

  

3. Sync propensity scores (daily only, after train DAG)

→ Read combine_propensity from BigQuery

→ Write to Redis user:{id}:propensity (batch pipeline)

```

  

#### Daily Batch (Keep Existing, Enhance)

  

Current aggregation DAG stays for full feature rebuild. Add:

  

```

New tasks in recommendation_data_aggregation_daily_v2:

  

1. Feature snapshot for point-in-time training (NEW)

→ Snapshot product_features_v2 with date key

→ When training, join interactions with features AS OF interaction date

→ Prevents label leakage (product with 10K views today doesn't get

that feature for a training row from 30 days ago)

  

2. User action log snapshot (NEW)

→ Daily: dump Redis user:{id}:actions → BigQuery user_action_log_YYYYMMDD

→ Provides offline training data for SASRec and two-tower user tower

→ Alternative: read directly from core_events (current approach),

but Redis snapshot ensures training/serving feature parity

  

3. Ranker training data preparation (NEW, Phase 2B)

→ Join impressions with clicks to create (query, impression_list, click_list) tuples

→ Attach product features + user features at impression time

→ Output: ranker_training_data_YYYYMMDD

```

  

### 7.6 Online ↔ Offline Sync

  

The most critical design decision: **how to keep online and offline stores consistent**.

  

#### Offline → Online (Serving Gets Fresh Features)

  

| What | Frequency | Mechanism | Target |

|------|-----------|-----------|--------|

| Product features (full) | Daily after aggregation DAG | BigQuery → Cassandra bulk load (via K8s pod) | `product_features_online` table |

| Product features (changed) | Hourly | Incremental BigQuery query → Cassandra update | Same table, changed rows only |

| Item embeddings (full) | Daily after training DAG | Model output → Cassandra bulk load | `item_embeddings` table |

| Item embeddings (new products) | Hourly | ONNX item tower on new products → Cassandra | Same table, new rows only |

| FAISS index | Daily + hourly rebuild | GCS upload → serving pod reload | In-process memory |

| User propensity scores | Daily after train_user_favorite_cate DAG | BigQuery `combine_propensity` → Redis bulk load | `user:{id}:propensity` |

| ONNX models (ranker, user tower, SASRec) | Daily after training | GCS upload → serving pod reload | In-process memory |

| Pre-computed reco lists (fallback) | Daily after inference DAG | BigQuery → Cassandra (existing flow) | `p_reco_<version>` (keep as fallback) |

  

#### Online → Offline (Training Gets Real-Time Signals)

  

| What | Frequency | Mechanism | Target |

|------|-----------|-----------|--------|

| User action logs | Daily | Redis `user:{id}:actions` → BigQuery snapshot (K8s pod) | `user_action_log_YYYYMMDD` |

| Product real-time counters | Daily | Redis `product:{id}:counters` → BigQuery (append to feature snapshot) | Enrich `product_features_v2` with intra-day signals |

| User browsing metrics | Not needed offline | Already in impression/click logs in BigQuery | — |

  

**Why Online → Offline matters**: Without this, training and serving see different data. If the serving ranker uses real-time Redis counters but training only sees daily BigQuery aggregates, the model learns on different feature distributions than it serves on. The daily snapshot closes this gap.

  

### 7.7 Point-in-Time Feature Correctness

  

For training, we need features **as they were when the interaction happened**, not as they are today.

  

**Current problem** (label leakage example):

```

Day 1: Product X has 10 views, user A views it (training positive)

Day 30: Product X has 100,000 views

Training: Uses product_features_v2 from Day 30 → assigns 100K views to Day 1 interaction

Result: Model learns "high-view products get clicks" instead of learning real signal

```

  

**Solution**: Daily feature snapshots + temporal join

  

```sql

-- Training data preparation (new SQL)

SELECT

i.session_id,

i.user_id,

i.product_id,

i.interaction_date,

-- Join features AS OF interaction date, not today

f.num_root_view_d30,

f.sale_price,

f.avg_rating_root_all,

...

FROM interaction_training_data i

LEFT JOIN product_features_v2_{interaction_date_suffix} f

ON i.product_id = f.product_id

```

  

This requires keeping ~30 days of `product_features_v2_YYYYMMDD` tables (already the case with current `_YYYYMMDD` suffix convention).

  

### 7.8 Feature Store Implementation: Build vs Buy

  

| Option | Pros | Cons | Recommendation |

|--------|------|------|----------------|

| **Build on existing infra** (Redis + Cassandra + BigQuery) | No new infra cost; team already knows these systems; incremental migration | No feature registry/catalog; manual schema management; no built-in point-in-time joins | **Start here** — Phase 1-3 |

| **Feast** (open-source feature store) | Feature registry; point-in-time joins; online/offline sync built-in; BigQuery + Redis supported as backends | New system to operate; learning curve; may be over-engineered for current scale | Consider for Phase 4+ if complexity grows |

| **Vertex AI Feature Store** (GCP managed) | Fully managed; integrates with BigQuery; streaming ingestion; monitoring | Vendor lock-in; cost; may not support Cassandra/Scylla as online store | Evaluate if team prefers managed services |

  

**Recommended approach**: Build incrementally on existing infra (Phase 1-3), evaluate Feast adoption for Phase 4+ when the number of features and models grows beyond what manual management can handle.

  

### 7.9 Implementation Sequence

  

```

Phase 1 (Weeks 1-3): No feature store changes

└─ Training improvements only, existing BigQuery features sufficient

  

Phase 2 (Weeks 4-7): Minimal online feature additions

├─ Cassandra: Add product_features_online table (for learned ranker)

├─ Daily sync: BigQuery product_features_v2 → Cassandra product_features_online

└─ Redis: Add user:{id}:propensity (daily sync from BigQuery)

  

Phase 3 (Weeks 8-15): Full online/offline feature store

├─ Kafka consumer: user action log → Redis (real-time)

├─ Kafka consumer: product counters → Redis (real-time)

├─ Cassandra: item_embeddings table (daily + hourly for new)

├─ Hourly DAG: feature refresh for changed products

├─ Daily: Redis → BigQuery snapshots (online → offline sync)

├─ Daily: BigQuery → Cassandra/Redis bulk sync (offline → online)

└─ Point-in-time feature joins for training data

  

Phase 4 (Weeks 16+): Evaluate Feast / managed feature store

├─ Feature registry and catalog

├─ Automated feature monitoring and drift detection

└─ Standardized feature serving API

```

  

### 7.10 Summary: What Changes for Each Model

  

| Model/Component | Current Data Source | After Improvement | Collection Method |

|-----------------|--------------------|--------------------|-------------------|

| **Two-tower item tower** (training) | BigQuery `product_features_v2` (daily) | Same + point-in-time snapshots | Daily batch (existing) |

| **Two-tower user tower** (training) | BigQuery `sale_orders` sessions (daily) | BigQuery `user_action_log` (snapshotted from Redis) + point-in-time product features | Daily: Redis → BigQuery snapshot |

| **Two-tower user tower** (serving) | N/A (pre-computed) | Redis `user:{id}:actions` + Cassandra `item_embeddings` → on-the-fly computation | Real-time: Kafka → Redis |

| **Learned ranker** (training) | N/A (doesn't exist yet) | BigQuery: impression/click pairs + product features + user metrics (point-in-time) | Daily batch |

| **Learned ranker** (serving) | N/A | Cassandra `product_features_online` + Redis `user browsing` + Redis `user propensity` | Daily batch + real-time |

| **SASRec** (training) | N/A | BigQuery `user_action_log` (ordered sequences) | Daily: Redis → BigQuery snapshot |

| **SASRec** (serving) | N/A | Redis `user:{id}:actions` (last 50, ordered) | Real-time: Kafka → Redis |

| **Title similarity** (training) | BigQuery `product_features_v2` (titles) | Same | Daily batch (no change) |

| **User propensity** (training) | BigQuery `event_staging` + `nmv` (daily) | Same + enriched features from aggregation tables | Daily batch (no change, just more SQL joins) |

| **User propensity** (serving) | Baked into pre-computed reco lists | Redis `user:{id}:propensity` (for ranker + user tower input) | Daily: BigQuery → Redis sync |

| **Current reranking** (serving) | Redis `UserBrowsing` (real-time) | Same (kept as fallback alongside learned ranker) | Real-time (no change) |

| **Candidate retrieval** (serving) | Cassandra pre-computed lists (daily) | FAISS ANN on item embeddings + pre-computed lists as fallback | Daily FAISS + hourly update |


  **Core** **Insight**

  

  Features fall into **3** **freshness** **tiers**, each with a different collection strategy:

  

  ┌────────┬─────────────────────────────────┬────────────┬─────────────────────────────────────┐

  │  Tier  │            Examples             │ Freshness  │             Collection              │

  ├────────┼─────────────────────────────────┼────────────┼─────────────────────────────────────┤

  │ **Static** │ product name, category, seller, │ Days/weeks │ Daily BigQuery batch (keep current) │

  │        │  product_tier                   │            │                                     │

  ├────────┼─────────────────────────────────┼────────────┼─────────────────────────────────────┤

  │ **Medium** │ price, stock, rating, view/sold │ Hours      │ **Hourly** **micro-batch** (incremental SQL │

  │        │  counts, CTR                    │            │  on changed products → Cassandra)   │

  ├────────┼─────────────────────────────────┼────────────┼─────────────────────────────────────┤

  │ **Fast**   │ user's last 50 actions, session │ Per-action │ **Real-time** Kafka → Redis (extend     │

  │        │  context, impression counts     │            │ existing UserBrowsing consumer)     │

  └────────┴─────────────────────────────────┴────────────┴─────────────────────────────────────┘

  

  **Online** **↔** **Offline** **Sync** **(the** **key** **question** **you** **asked)**

  

  **Offline** **→** **Online** (daily/hourly): BigQuery features → Cassandra product_features_online table (for

  learned ranker) + Redis user:{id}:propensity (for user tower). This is straightforward — bulk load

   after each DAG run.

  

  **Online** **→** **Offline** (daily snapshot): Redis user action logs → BigQuery user_action_log_YYYYMMDD.

  This is critical because:

  - The two-tower user tower and SASRec need user action sequences for training

  - If training reads from BigQuery core_events but serving reads from Redis, you get

  **training-serving** **skew**

  - Daily Redis → BigQuery snapshots ensure the training data matches what serving sees

  

  **Point-in-Time** **Correctness**

  

  Current approach has **label** **leakage** — a product with 10 views on Day 1 gets today's 100K-view

  feature for historical training rows. Fix: join interactions with

  product_features_v2_{interaction_date} snapshots (tables already exist with _YYYYMMDD suffix).

  

  **Build** **vs** **Buy**

  

  Start by building on existing infra (Redis + Cassandra + BigQuery) for Phases 1-3. Evaluate **Feast**

  (open-source feature store) for Phase 4+ if the number of features/models grows beyond manual

  management.