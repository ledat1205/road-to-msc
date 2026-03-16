

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