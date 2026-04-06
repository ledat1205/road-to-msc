Let me pull the full conversation to make sure nothing is missed.Good — I have everything. Let me build this properly from scratch, incorporating everything we've covered across the whole conversation.

---

## The Full Interview Frame

### Opening (30 seconds — memorize word for word)

> "I own the end-to-end recommendation system at Tiki — raw event data through to what users see on their homepage and product pages. The core engineering challenge is that we have two surfaces answering fundamentally different questions. Home asks _what does this user want right now_. PDP asks _what's related to this product_. Because the questions are different, everything — data pipelines, models, serving, evaluation, monitoring — is architected differently for each. Let me walk you through it."

---

## The Five Acts

---

### Act 1 — Data Foundation (2–3 min)

**Hook to open with:**

> "Before any model trains, you need to decide: what signal actually changes fast enough to justify real-time infrastructure, and what's stable enough to compute in batch? Getting that boundary wrong is expensive either way."

**What you explain:**

**Batch layer** — dbt computes features daily over 7/30/90-day windows, materialized to BigQuery, served via Feast online store at ~5–10ms per lookup. 18 user features, 15 item features. Feast exists for one specific reason: BigQuery point lookups take 200–500ms — too slow for serving. Feast bridges the analytical world and the serving world.

**Streaming layer** — Spark Structured Streaming on Kafka computes exactly 10 features into Redis: 6 user session features + 4 product trending features. Deliberately narrow scope. The question you ask for every feature: does it change fast enough that batch cadence hurts the user experience? Session context shifts every pageview — yes. Trending scores change hourly — yes. Brand affinity, ratings, sales velocity — too sparse or too stable. Batch is sufficient.

**The insight to land:**

> "A common mistake is to stream everything because it feels more sophisticated. We asked which features actually change fast enough to matter for recommendations. The answer was 10. Everything else is batch. That keeps the streaming job simple, observable, and cheap to operate."

---

### Act 2 — Training Pipelines (3 min)

**Hook:**

> "Training is where the interesting DE problem lives — specifically how you ensure the model trains on the same signal it will see at serving time, when the two sources are fundamentally different systems."

**Home — Two-Tower:**

- Dataset built entirely in BigQuery: session reconstruction via window functions, point-in-time feature joins, time-based train/eval split (never random — random splits leak future signal)
- The train/serve consistency problem: session features come from Redis at serving time, but training has no Redis. Solution: reconstruct session features from raw event logs using window functions. Statistically consistent — same distributions, same meaning — without any streaming dependency in training
- Model: 28 features → user tower → 64-dim embedding. 15 features → item tower → 64-dim embedding. In-batch negative loss — batch size 256 gives 65,000 implicit negatives per step, no explicit negative sampling needed
- Export: ONNX user tower + FAISS item index. Item tower discarded after export — it only exists to build the index
- Infrastructure: PyTorch training on GCP VM with T4 GPU. MLflow logs every run — hyperparameters, metrics, artifacts. Gate: if Recall@50 doesn't meet threshold vs production model, DAG fails and nothing ships

**PDP — LightFM + Sentence-BERT:**

- Two models because they learn orthogonal things. LightFM sees sessions, never product names. SBERT sees product names, never sessions.
- LightFM: session × product interaction matrix, WARP loss (optimizes ranking directly, not just positive/negative margin). Captures behavioral co-occurrence and complementary relationships.
- SBERT: fine-tuned on (anchor, positive, hard negative) triplets from co-view pairs joined to product names. MNR loss — in-batch negatives are implicit, hard negatives mined from same-category never-co-viewed pairs. Handles cold start and long-tail products LightFM can't embed reliably.
- Why both: LightFM knows "laptop bags co-occur with laptops" but doesn't know which bag. SBERT knows semantic similarity but misses complementary relationships entirely. Combined score is strictly more robust than either alone.
- Output: pre-computed similarity lists written to KVDal. No embedding index needed at serving time.

---

### Act 3 — Serving Pipelines (2–3 min)

**Hook:**

> "The design principle at serving time is: do as little computation as possible. Front-load everything offline. The only things you recompute live are the ones where staleness directly hurts the user."

**Home API (~50–80ms P95):**

Walk it as a sequence:

```
1. Parallel fetch
   Feast  → 18 batch customer features (~5–10ms)
   Redis  → 6 session features (~3ms)
   HTTP   → platform, hour, day, is_weekend (0ms)

2. ONNX user tower inference
   28 features → 64-dim user embedding (~5ms)
   Only neural network call in the entire serving path

3. Parallel retrieval — two channels answering different questions
   FAISS ANN(user_embedding, k=1000) → personalized candidates
   Redis ZREVRANGE(user's top cates) → trending candidates
   FAISS knows long-term preferences. Redis knows what's hot right now.
   They are additive, not redundant.

4. Merge + rank
   Cross-features computed here — first moment you know
   both user and specific candidates simultaneously

5. Filter + diversify → top K
```

**PDP API (~5–10ms P95):**

```
1. KVDal lookup(product_id)
   → pre-computed similar + complementary lists

2. Redis ZREVRANGE
   → trending in same + complementary categories

3. Merge + filter → ranked list
```

No Feast. No ONNX. No FAISS. Pure lookups.

**The contrast to land:**

> "PDP is 10x faster than Home because product-product similarity is stable enough to pre-compute entirely offline. The serving layer has nothing to compute — just two lookups and a merge."

---

### Act 4 — Evaluation as a Gate System (2 min)

**Hook:**

> "Evaluation isn't a step that happens once before deployment. It's a gate system — each gate is automated, logged in MLflow, and blocks promotion if thresholds aren't met."

**As a solo DE you run a simplified but principled version:**

**Gate 1 — Offline eval (automated, every training run):**

- Recall@K, NDCG@K, MRR on time-based holdout
- Must not regress vs current production model — relative comparison, not absolute threshold
- Train/serve skew check: compare session feature distributions between BigQuery reconstruction and Redis. Compare Feast offline vs online store. Flag null rate spikes. Uses KS test and PSI — PSI > 0.2 triggers alert.
- If gate fails → DAG raises exception, nothing ships, Slack alert fires

**Gate 2 — A/B test:**

- Start 5% traffic, watch hard guardrails 48 hours (latency, null rate, error rate), expand to 10%
- Metric hierarchy: GMV/session (primary) → add-to-cart rate → CTR (never primary, easy to game). Return rate as quality guardrail — lags 2–3 weeks, run experiment long enough.
- Segment Home by session depth — streaming session features should only outperform batch-only at session depth > 5. No difference there means streaming infrastructure isn't justified.
- Segment PDP by catalog tier — SBERT win concentrates in long-tail. If improvement is uniform across head/torso/tail, something unexpected is happening.
- Minimum 2 weeks. Weekly seasonality in e-commerce is significant enough to invalidate shorter experiments.

**Gate 3 — Continuous monitoring:**

- Six daily checks → Slack alert: Feast materialization completed, Redis session hit rate, KVDal coverage, null recommendation rate vs 7-day baseline, P95 latency vs budget, CTR vs 7-day baseline
- Automated rollback on latency breach or null rate spike
- PSI monitoring on 6 key features: `session_view_count`, `session_top_cate`, `num_pdp_views_d30`, `customer_category_affinity`, `days_since_last_purchase`, `trending_score_1h`
- Retraining triggered by signal — sustained metric drop, PSI > 0.2, catalog structure change — not just weekly schedule

---

### Act 5 — What I'd Do Next (1 min)

Shows you think beyond current state:

**GNN for PDP** — LightFM captures only direct co-view pairs (1 hop). A GNN propagates signal across multiple hops during training — product 111 influences product 123's embedding even if they were never directly co-viewed, via shared neighbors. Shopee uses this approach. Infrastructure cost is significant so the pragmatic intermediate step is Node2Vec on the co-view graph: simpler than full GNN, captures 2–3 hop relationships, same serving architecture.

**Reranking layer for Home** — current merge scoring is lightweight arithmetic. A small cross-encoder reranking model on the top-50 candidates could improve final ranking quality without blowing the latency budget, since it only scores 50 items not thousands.

**MLflow model registry formalization** — currently using GCS path conventions for production/candidate/archive. As the team grows beyond solo, formal stage transitions in MLflow registry become worth the overhead.

---

## Questions They'll Ask — Your Exact Answers

**"Why not BigQuery ML for training?"**

> "BQML doesn't support dual-encoder architectures with custom loss functions, WARP loss, or transformer fine-tuning. BigQuery is the data preparation layer for all three pipelines — the actual training runs on a GCP VM with PyTorch and a T4 GPU. For POC scale that's sufficient and keeps infrastructure simple."

**"How do you handle cold start for new products?"**

> "Two mechanisms. New products get SBERT title embeddings immediately from their name alone — no interaction data needed — so they enter KVDal from day one. They also surface through the Redis trending channel as soon as they accumulate views, which happens within one hour of going live. The head/torso catalog handles itself through LightFM interaction signal. Cold start is fundamentally a long-tail problem and SBERT plus trending covers it adequately."

**"What's the hardest data engineering problem?"**

> "Train/serve consistency on session features. At serving time they come from Redis. During training we reconstruct them from BigQuery window functions. The values aren't byte-identical but they need to be statistically consistent — same distributions, same meaning — otherwise the model learns signals it won't see at inference time. Getting that reconstruction right, and then continuously monitoring for drift between the two sources, is the subtlest ongoing problem in the system."

**"Why two retrieval channels for Home?"**

> "FAISS and Redis answer fundamentally different questions. FAISS knows what the user loves based on 30 days of history — it can't know what's trending right now because it was trained in the past. Redis trending knows what's spiking in the last hour — but it has no personalization signal. A viral laptop bag won't appear in FAISS if the user has never browsed bags. A niche product the user would love won't show in trending. The channels are additive, not redundant. You need both."

**"Why keep LightFM if you have SBERT?"**

> "They learn orthogonal things. LightFM sees sessions, never product names — it captures behavioral co-occurrence and complementary relationships like laptop → laptop bag. SBERT sees product names, never sessions — it captures semantic similarity and handles cold start. Neither alone is sufficient. LightFM struggles on new and long-tail products with sparse interactions. SBERT misses complementary relationships entirely because semantically a laptop and a laptop bag are quite different. The combined score is strictly more robust."

**"What would you cut as a solo DE?"**

> "Shadow mode and formal canary as separate stages — they collapse into a single A/B at low initial traffic with hard guardrail monitoring for 48 hours before expanding. Segmented monitoring narrows to six key features and six daily checks rather than full PSI across all 40 features. Human SBERT evaluation becomes monthly spot checks rather than a formal annotation pipeline. The one thing I'd never cut is the offline eval gate comparing against the production model — without that there's no automated check that a new model doesn't regress before it ships."

**"What is MLflow doing in your system?"**

> "MLflow is the audit trail for the training pipeline. Every run logs hyperparameters, metrics, and artifacts. If something breaks in production I can identify exactly which model version caused it, compare its metrics against the previous version, and restore the previous ONNX file and FAISS index from GCS within minutes. Without it you have model files with no provenance — you don't know what training run produced them or what they achieved."

---

## The Meta-Point to Internalize

Every answer frames decisions as **tradeoffs**, not just choices:

|Decision|Tradeoff|
|---|---|
|Batch vs streaming|Staleness vs latency vs operational cost|
|Two retrieval channels|Personalization vs freshness — additive not redundant|
|LightFM + SBERT|Behavioral signal vs semantic signal — orthogonal not redundant|
|Pre-compute vs live compute|Speed vs freshness — front-load what's stable|
|Simplified MLOps solo|Reliability vs sophistication — operate what you can maintain|
|Time-based eval split|Correctness vs convenience — random splits leak future signal|

Senior interviewers aren't checking if you know the right answer. They're checking if you know _why_ it's right given specific constraints. Tradeoff framing is what separates a senior DE answer from a junior one.