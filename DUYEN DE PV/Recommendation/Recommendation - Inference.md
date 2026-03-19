This document details the complete re-architecture I owned — converting the legacy Airflow batch pipeline into a production-grade, low-latency streaming system. It directly addresses the CV bullets:

- Designed data pipelines and feature schemas supporting **real-time model evaluation**.
- Defined **canonical metric definitions** (NDCG, CTR) used consistently across search and recommendation model iterations.
- Rearchitected the product recommendation platform from batch to real-time using **Flink and ScyllaDB**, reducing latency from hours to seconds.

The new system now supports sub-second feature freshness, real-time model evaluation, polluted-click protection, and seasonal drift detection — patterns widely used by Netflix, Uber, Alibaba Taobao, and LinkedIn.

### 1. Problems with the Old Batch Inference System (Airflow + BigQuery)

Before the migration, the entire recommendation platform relied on nightly Airflow DAGs:

- **Stale features**: User intent and category trends were computed only once every 4–8 hours. During flash sales or Lunar New Year, user behavior changed in minutes — models evaluated on 6-hour-old data.
- **High inference latency**: Online inference required heavy BigQuery joins across billions of rows → 200–500 ms per request (far above the <15 ms target).
- **Inconsistent model evaluation**: Different teams defined CTR and NDCG differently in scattered SQL transformations. Offline NDCG@10 and CTR could not be reliably compared across model iterations.
- **No real-time A/B testing**: New model versions could only be evaluated the next day.
- **Pollution & drift vulnerability**: Duplicate clicks and seasonal spikes (covariate/label/concept drift) went undetected until after training — causing 3–8 % online metric drops and revenue loss.
- **Scalability ceiling**: At 5× peak traffic, batch jobs OOM-ed or took >12 hours.

These are the exact pain points Netflix, Uber, and Alibaba solved by moving to Lambda + real-time feature stores.

### 2. Migration Strategy: From Airflow Batch to Flink Streaming + ScyllaDB (Lambda Architecture)

I designed and implemented a **Netflix/Uber-style Lambda architecture**:

- **Speed Layer (Flink + Kafka)**: Real-time feature computation and user intent tracking.
- **Batch Layer (dbt + Airflow)**: Nightly backfills and historical training data (kept for consistency).
- **Serving Layer (ScyllaDB + Redis + Go)**: Sub-second lookups for online inference.

![Lambda Architecture: A Big Data processing framework | by Abhinav Vinci |  Medium](https://miro.medium.com/v2/resize:fit:1200/1*Nn-rjIpPVCYgw4-LvCkK8A.png)

[medium.com](https://medium.com/@vinciabhinav7/lambda-architecture-a-big-data-processing-framework-introduction-74a47bc88bd3)

![Lambda Architecture Explained: A Simple Guide for Beginners](https://www.guvi.in/blog/wp-content/uploads/2026/01/The-Three-Layers-of-Lambda-Architecture-1200x630.webp)

[guvi.in](https://www.guvi.in/blog/what-is-lambda-architecture/)

![Lambda Architecture in Banking Systems – Balancing Batch and Stream](https://media.licdn.com/dms/image/v2/D4E12AQHbp4pMPXS1Ng/article-cover_image-shrink_720_1280/B4EZgZT3JOHIAI-/0/1752771297592?e=2147483647&v=beta&t=-VFOLxvixHXndMEswI3lwULiGGGS8eAsackeqkwxP6w)

[linkedin.com](https://www.linkedin.com/pulse/lambda-architecture-banking-systems-balancing-batch-sedaghatbin-kqxje)

The migration was done in three phases while keeping the old batch system running in parallel (shadow mode for 6 weeks).

### 3. System Goal & Scale

- Target: <15 ms p95 inference latency at 1,000+ QPS.
- Scale: 5 M master_ids (semantic products) + 30 M pids (seller-specific items).
- Freshness: Shifted from hours → seconds for user intent and category trends.
- Hard constraints: Master_id-level deduplication + Exactly-Once semantics for all state (Flink checkpoints + Kafka offsets).

### 4. Tiered Data Strategy (The “Storage Matrix”)

To balance speed, cost, and durability:

|Component|Storage Backend|Logic / Use Case|
|---|---|---|
|User Session Intent|Flink (RocksDB)|Decaying category scores based on live clicks (millions of active users)|
|Hot Serving Data|Redis Cluster|Recent spids, category best-sellers, master:pid maps|
|Persistent Dimensions|ScyllaDB|Global semantic relations, historical reco_by_recency_long|
|Real-Time Analytics|Apache Druid|5-minute rollups for “Top Sold” items per category|

### 5. Real-Time Personalization Pipeline (Flink) – The “Brain”

![Real-Time ETL with Flink and Kafka: Trends | by Sanket Nadargi | Medium](https://miro.medium.com/v2/resize:fit:1400/0*mbDYK233IETufI3X)

[medium.com](https://medium.com/@sanket.nadargi1/real-time-etl-with-flink-and-kafka-trends-ebb3d9dff635)

![Trillions of Bytes of Data Per Day! Application and Evolution of Apache  Flink in Kuaishou - Alibaba Cloud Community](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcScXizP-Uiy1__4yCwBczM7lTvsY64NjUqtEA&s)

[alibabacloud.com](https://www.alibabacloud.com/blog/trillions-of-bytes-of-data-per-day-application-and-evolution-of-apache-flink-in-kuaishou_596675)

**A. Internal Keyed State (Why RocksDB)**

- EmbeddedRocksDBStateBackend chosen for scale (millions of users exceed heap RAM).
- Flink checkpoints RocksDB state + Kafka offsets together → true Exactly-Once.
- State TTL + incremental compaction handles memory pressure during peaks.

**B. User Intent & Category Decay** We track live user intent with a MapState<String, Double> keyed by user_id:

1. Score(A) = Score(A) + 1.0 on interaction
2. Score(Others) = Score(Others) × 0.8 (decay factor)

This enables instant pivots (e.g., “Shoes” → “High Heels” in seconds).

**C. Polluted-Click Protection & Drift Detection**

- Stateful deduplication (click_id + user_id) in 60-second windows.
- Intra-day KS drift checks (Alibi Detect) on CTR features → route bad events to dead-letter topic.
- Canonical NDCG/CTR precursors computed in real time for immediate model evaluation.

### 6. Recommendation Logic & Data Modeling

**Semantic & Category Expansion** Three signals combined in Flink:

- Semantic similarity (pre-computed in ScyllaDB).
- Real-time co-view relations (Flink window aggregates).
- User recency (last 15 min from state).

**Master-to-Pid Resolution**

- Redis Hash: master:pids:{mid} (price, rating, stock, seller_id).
- Go layer picks the best PID per master_id (cheapest/highest-rated).

### 7. Serving Layer (Go Backend)

Final rerank & filter in <15 ms:

- Parallel fetches (Redis + ScyllaDB).
- Scoring: FinalScore = BaseScore × IntentScore_category.
- Business filters (promo, stock) via Redis.
- Master_id deduplication with Go map.

### 8. Integration with Real-Time Model Evaluation & Canonical Metrics

- All features now flow through the same dbt-defined schemas.
- Canonical NDCG@10 and CTR macros are computed in Flink and stored as side outputs → models can evaluate new versions in real time (A/B testing in seconds, not days).
- Drift gates block training if polluted clicks or seasonal drift detected.

### 9. Major Challenges & How I Overcame Them

- **Exactly-Once at scale**: Solved with Flink checkpoints + RocksDB + idempotent Kafka producers.
- **Memory pressure during peaks**: RocksDB spill-to-SSD + 10 % stratified sampling for drift checks.
- **False positives on seasonal events**: Dynamic baselines + is_peak_season flag + warning mode first.
- **Consistency between Flink & batch**: Shared dbt macros + dual-write to ScyllaDB.
- **Master_id deduplication**: Handled at both Flink and Go layers.

### 10. Results & Business Impact

- Inference latency: hours → <2 seconds (p95 <15 ms).
- Polluted clicks: 11 % → 0.4 %.
- Model evaluation freshness: daily → real-time (NDCG/CTR now consistent across iterations).
- Seasonal peaks: zero bad models deployed.
- Online NDCG@10 & CTR variance: ↓65 %.
- Revenue protected: >5 billion VND across 2025 events.

### 11. Big-Tech Techniques Applied

- Netflix/Uber Lambda + RocksDB state.
- Alibaba Taobao real-time feature serving with ScyllaDB.
- LinkedIn/Netflix observability (Alibi Detect drift gating).
- Airbnb-style canonical metric layer.
In Flink, a **Window** is just a way to **group events together** so you can do calculations on them (like sum, average, count, deduplication, etc.).

Think of it like putting events into different “buckets” based on time or count.

Here are the **main types** of windows in Flink, explained simply:

### 1. Tumbling Window (Most Common)

- **What it is**: Fixed, non-overlapping buckets.
- Example: Every 5 minutes → events from 00:00-00:05, 00:05-00:10, 00:10-00:15…
- No overlap.

**How we used it at Tiki**:

- 5-minute tumbling window for real-time drift checking (KS test every 5 minutes).
- 60-second tumbling window for click-to-impression ratio check (to detect polluted clicks).

### 2. Sliding Window (Overlapping)

- **What it is**: Buckets that overlap.
- Example: Every 5 minutes, but the window slides every 1 minute. → Window 1: 00:00-00:05 → Window 2: 00:01-00:06 → Window 3: 00:02-00:07

**How we used it**:

- Calculating rolling CTR (ctr_7d, ctr_mobile_7d) in real time.
- Session-based metrics that need to update frequently.

### 3. Session Window (Activity-based)

- **What it is**: Groups events by user activity with a **gap**.
- Example: If a user is inactive for more than 30 minutes → close the session window and start a new one.

**How we used it**:

- Tracking user intent and session_click_sequence.
- If user is idle for 30 minutes → we decay their intent scores and close the session.

### 4. Count Window

- **What it is**: Groups by number of events, not time.
- Example: Group every 100 clicks.

**How we used it**:

- Rarely, but we used it for sampling (e.g., process every 1000 events for heavy calculations to reduce load).

### 5. Global Window (Advanced)

- **What it is**: One single big window for all events (never closes automatically).
- You must manually define when to trigger calculation using **Triggers**.

Used when you want full control (we used it for some global metrics).