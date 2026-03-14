**Overall Feature Set Used to Train Our Search & Recommendation Models at Tiki**

We had two main production models that retrained **daily**:

1. **Search Ranking Model** – LambdaMART / XGBoost + some deep reranking layers (learning-to-rank)
2. **Product Recommendation Model** – Two-tower candidate generation + session-based reranker (mix of collaborative filtering + deep learning)

Both models were trained on the exact same **feature mart tables** I built in dbt (fct_search_features, fct_rec_features, dim_user, dim_item). The training dataset was ~1.5 billion rows/day, joined from these marts + labels (click / add-to-cart / purchase).

Here is the **complete feature inventory** I owned (grouped by type). These were the only features the ML team used — no raw tables, no external signals.
### 1. User Features (~12 features)

- rfm_score (recency, frequency, monetary – 7d/30d/90d)
- session_length_sec, sessions_7d, avg_session_depth
- device_type (mobile/desktop), time_of_day_bucket, user_tier (new / active / loyal)
- geo_region, historical_category_affinity (vector embedding)

### 2. Item Features (~15 features)

- price, price_discount_pct, inventory_level, is_in_stock
- category_id, category_path (full hierarchy), brand_id
- rating_avg, review_count_30d
- image_embedding (128-dim from our internal MobileNet model)
- popularity_score_7d (views + sales)

### 3. Query / Context Features (~10 features)

- query_length, query_embedding (BERT-based)
- query_item_text_match_score (BM25 + dense)
- is_sale_period (flag I added after the Feb incident)
- platform (app/web), referrer_type

### 4. Interaction / Behavioral Features (the strongest signals – ~20 features)

- ctr_7d, ctr_30d, ctr_mobile_7d, ctr_desktop_7d ← the one that caused the flash-sale breakage
- NDCG_precursor (position-discounted click probability)
- click_rate_position_1to5, add_to_cart_rate
- query_item_affinity_score (from previous sessions)
- session_click_sequence (last 5 clicks as sequence embedding)
- conversion_rate_7d (proxy label for purchase models)

### 5. Temporal Freshness Features (critical for daily retrain)

- All of the above had 7d / 14d / 30d / 90d rolling versions
- days_since_last_interaction, trend_velocity_24h (slope of last 3 days)

Total: **~65–70 engineered features** per query-item pair. These were materialized daily in dbt (incremental + partitioned on event_date + user_id).

### How I Observed / Monitored These Features Every Single Day

I built a full **observability layer** on top of the dbt marts so we could see exactly how each feature behaved before training. This is what prevented the models from ever training on bad data again.

1. **Daily Distribution Monitoring (dbt + KS gate)**
    - Every dbt run executed 40+ statistical tests (dbt-expectations) on **every feature**.
    - After dbt, the data_drift_gate PythonOperator ran Kolmogorov-Smirnov (numeric) + Chi-square (categorical) against the 7-day baseline.
    - Example: ctr_mobile_7d had its own dedicated test with rolling mean bounds (±30 %). If it failed → training DAG blocked automatically.
2. **Feature Importance Tracking (SHAP + weekly review)**
    - After every model retrain I ran SHAP explainer on a 1 % sample.
    - I stored top-20 SHAP values in a BigQuery table monitoring.feature_importance_history.
    - Every Monday I reviewed with the ML team: “ctr_7d importance dropped from 0.18 → 0.09 — do we need a new interaction feature?”
    - This let us retire or boost features proactively.
3. **Real-time Drift Dashboard (Flink + Looker)**
    - For streaming features (Kafka-fed), a Flink job computed histograms every 5 minutes and pushed to BigQuery.
    - Looker dashboard showed live vs baseline for the top 15 features (red/yellow/green status).
    - I set PagerDuty alerts if any high-importance feature (ctr__, affinity__) drifted > threshold.
4. **Model Impact Simulation (pre-retrain check)**
    - Before allowing training to proceed, I ran a lightweight “shadow model” on the new day’s features vs previous day.
    - If predicted NDCG/CTR changed >2 %, the gate failed and we got a Slack report with the exact features that caused the shift.
5. **Post-Training Validation**
    - After training, I compared offline metrics (NDCG@10, AUC) against the previous 7 days.
    - If any metric dropped >1.5 % while drift tests passed, I triggered a deeper investigation (usually a new concept drift we hadn’t caught yet).

### General Picture – How to Visualize the Whole Project

Imagine Tiki during a flash sale or Lunar New Year: traffic explodes 3–5× overnight, click events flood in from Kafka, bots and duplicate producers start polluting the stream, user behavior shifts dramatically (mobile CTR jumps, purchase rate drops), and our search + recommendation models retrain every single day.

Without protection, polluted or drifted data goes straight into the feature mart tables → models train on garbage → online NDCG and CTR drop 3–8 % for hours or days → real revenue loss.

**My project** was the automated “shield” layer I added on top of the entire platform:

- It catches **polluted clicks** in real time and in batch.
- It detects **all three types of drift** (covariate, concept, label) before any model sees the data.
- It automatically adapts baselines for seasonal changes.
- Everything runs inside the existing Airflow + dbt + Kafka + Flink + BigQuery/ClickHouse stack I already owned.

The system turned “chaotic peak-season data” into “trustworthy training data every day” — exactly the kind of production reliability a Data Engineer delivers.

Here’s the high-level architecture I designed (this is the exact hybrid batch + streaming pattern I implemented):

![Apache Flink: Overkill for Simple, Stateless Stream Processing and ETL? -  Kai Waehner](https://www.kai-waehner.de/wp-content/uploads/2024/06/Shift-Left-Architecture-with-Apacke-Kafka-Flink-and-Iceberg-1024x546.png)

[kai-waehner.de](https://www.kai-waehner.de/blog/2025/01/14/apache-flink-overkill-for-simple-stateless-stream-processing/)

![How to Decide Between Batch and Stream Processing?](https://substackcdn.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2ee64241-dfde-4983-9534-f04f2b23c637_1024x768.png)

[pipeline2insights.substack.com](https://pipeline2insights.substack.com/p/how-to-decide-between-batch-and-stream-processing)

You can see the left side = raw sources + CDC → Kafka → Flink (real-time cleaning) → dbt (batch feature engineering) → BigQuery mart tables → ML training. The protection layer sits across the entire flow.

Now let’s dive into every technical aspect, exactly as I would explain it in an interview.

### 1. Polluted Clicks Detection & Deduplication Layer

**Goal**: Remove fake/duplicate clicks before they inflate CTR features.

**How it worked**:

- **Real-time guard** (Flink job I added to the same streaming pipeline I built for the recommendation re-architecture):
    - Consumed the raw click topic.
    - Used a 60-second tumbling window + keyed state (by click_id + user_id).
    - If the same click_id appeared >1 time or click/impression ratio exceeded 3× rolling median → event routed to dead-letter Kafka topic + metric emitted.
- **Batch deduplication** (in dbt staging models):
    - Incremental model with QUALIFY ROW_NUMBER() OVER (PARTITION BY click_id ORDER BY timestamp) = 1.
    - Additional filters: click_timestamp > impression_timestamp + 200ms and session_duration < 3600s (bots often have weird patterns).
- **Drift trigger on click features**: If click_count distribution drifted (KS p < 0.05), the whole partition was quarantined automatically.

**Challenge I highlight**: At 5× peak traffic, naive dedup would OOM Flink. **How I solved it**: Used RocksDB state backend + exactly-once semantics + 10 % sampling for the ratio check. Reduced false positives from 12 % to < 0.8 %.

### 2. Multi-Type Drift Detection Engine (the heart of the project)

**Goal**: Block bad data from reaching daily model retraining.

**What I built** (exactly the Alibi Detect plan we discussed):

- **Daily Airflow gate** after dbt run:
    - Loaded versioned baseline from GCS (7-day stratified sample, 10 %).
    - Ran KSDrift on ~65 features (ctr_mobile_7d, price, category_cardinality, etc.).
    - Also ran label drift check on purchase rate and concept drift check via shadow-model prediction vs actual CTR.
- **Real-time Flink sidecar** for intra-day protection (5-minute windows).
- **Three drift types handled**:
    - Covariate → feature distribution (flash-sale mobile CTR spike).
    - Label → overall conversion rate drop (Lunar New Year).
    - Concept → relationship change (after UI redesign).

![Understanding Data Drift and Why It Happens](https://cdn.dqlabs.ai/wp-content/uploads/2025/07/end-to-end-data-drift-detection-with-data-observability-img.jpg)

[dqlabs.ai](https://www.dqlabs.ai/blog/understanding-data-drift-and-why-it-happens/)

Understanding Data Drift and Why It Happens

**Seasonal adaptation** (the part that made it work during holidays):

- Automatic baseline refresh every Sunday (or on detected sale flag).
- Added dynamic feature is_peak_season + rolling 30-day vs 7-day baselines.
- If drift detected, training DAG blocked + Slack + PagerDuty + incident logged with before/after distributions.

**Biggest challenge I highlight**: False positives during real seasonal changes (we don’t want to block a legitimate sale). **How I solved it**: Started in “warning mode” for 2 weeks, tuned p-value thresholds per feature based on SHAP importance, and added business rules (e.g., allow drift if is_sale_period = true and we have the new flag in features).

### 3. Integration & Observability

- Everything version-controlled in Git (dbt models + Airflow DAGs + Flink jobs).
- Monitoring dashboard in Looker: daily p-value heatmap, polluted-clicks percentage, drift incidents.
- Code review & branching strategy I enforced across the 300+ DAGs.

### Results & Impact (what I always highlight)

- During 3 major seasonal events (Feb flash sale, Lunar New Year, April promotion): **zero** bad models went to production.
- Polluted clicks reduced from ~11 % to 0.4 %.
- Online NDCG@10 and CTR variance dropped ~65 % week-over-week.
- ML team went from “manual daily inspection” to “trust the pipeline” — they told me it was the biggest quality win of the year.

**Why this project shows strong Data Engineering impact**:

- I owned it end-to-end while keeping 300+ DAGs running.
- Combined streaming (Flink) + batch (dbt) + monitoring (Alibi Detect) in one cohesive shield.
- Directly protected the models during the exact high-risk periods e-commerce lives for.