I owned and built the end-to-end data reliability layer that protects Tiki’s search ranking and product recommendation models during normal operation **and** extreme seasonal events (flash sales, Lunar New Year, Black Friday).  

This system directly addressed the biggest risk in e-commerce ML: **bad or drifted data reaching daily model retraining**, causing immediate online metric degradation and revenue loss.

### 1. Executive Summary
- **Scope**: 300+ Airflow DAGs, 65+ engineered features, ~1.5–2 billion rows/day  
- **Core Achievement**: Built a hybrid batch + real-time “data shield” using dbt + Alibi Detect + Flink that automatically detects polluted clicks, covariate drift, concept drift, and label drift.  
- **Key Outcome**: Zero bad models deployed during three major 2025 seasonal peaks; polluted clicks reduced from ~11% → 0.4%; online NDCG@10 and CTR variance dropped ~65%.

### 2. Data Landscape – What Kinds of Data I Handled

**Raw Transactional Sources (MySQL / PostgreSQL + CDC via Debezium)**
- Orders, users, products, inventory, cart, search queries, payments
- Slowly-changing dimensions (product catalog, pricing, categories)

**Real-time Event Streams (Kafka – 15–20 high-volume topics)**
- Clickstream events (`impression`, `click`, `add_to_cart`, `purchase`, `search_query`)
- Recommendation feedback (impression → action pairs)
- A/B test exposure logs
- User session events (page views, scroll depth, device, geo)

**Feature Store & Warehouse Data (BigQuery + ClickHouse)**
- Aggregated user features (RFM, session stats, tier)
- Item features (price, inventory, image embeddings, brand, category hierarchy)
- Interaction features (`ctr_7d`, `ctr_mobile_7d`, `NDCG_precursor`, `query_item_affinity_score`, `session_click_sequence`)
- Temporal rolling windows (7d / 30d / 90d)

**Label Data for Models**
- Positive labels: actual clicks, add-to-cart, purchases
- Negative labels: impressions without clicks (critical for ranking)

**External / Seasonal Data**
- Marketing campaign flags, sale periods, flash-sale metadata

This data is extremely high-velocity, high-cardinality, and highly seasonal — exactly the conditions that create polluted clicks and drift.

### 3. Key Problems Before My Work

**Problem 1 – Polluted Clicks**  
Duplicate events, bot clicks, test traffic, and double-clicks inflated CTR features by 10–30% during peaks. Models learned wrong probabilities → over-boosting irrelevant items.

**Problem 2 – Severe Drift During Seasonal Events**  
- Covariate drift: mobile CTR distribution shifted 38–45% in hours during flash sales.  
- Label drift: overall conversion rate dropped 40% during Lunar New Year.  
- Concept drift: after UI redesign, same features predicted completely different user intent.  
Models retrained daily on drifted data → immediate 3–8% drop in online metrics.

**Problem 3 – Brittle Legacy Pipelines**  
- 300+ scattered BigQuery SQL transformations (no version control, no tests).  
- Batch-only recommendation platform (hours of latency).  
- No systematic data quality gates → ML team manually inspected distributions every morning.

**Problem 4 – No Protection at Scale**  
During 5× traffic spikes, simple checks would fail or cause OOM; false positives would block legitimate sales.

### 4. Solution Architecture (High-Level)

I designed a **Lambda-style hybrid architecture** with protection at every layer:

![[Pasted image 20260315171825.png]]


![[Pasted image 20260315171938.png]]


**Protection layers I added**:
- Real-time (Flink) → polluted click deduplication + intra-day drift guard
- Batch (dbt + Airflow) → full statistical drift detection + quarantine
- Observability (Looker + monitoring tables) → automatic alerts and incident logging

![[Pasted image 20260315171915.png]]


### 5. Detailed Technical Solutions I Implemented

**A. Polluted Clicks Detection & Deduplication (Real-time + Batch)**
- Flink job (keyed by `click_id + user_id`, 60-second tumbling window + RocksDB state) detects duplicates and extreme ratios.
- dbt incremental staging model with `ROW_NUMBER()` + business filters (`click_timestamp > impression_timestamp + 200ms`, session duration bounds).
- If KS test on `click_count` or `ctr_*` fails → entire partition quarantined to `quarantine_clicks` table.

**B. Multi-Type Drift Detection Engine (Alibi Detect + Custom Gates)**
- **Covariate drift**: KSDrift (Alibi Detect) on 65 features using 7-day stratified 10% baseline stored in GCS.
- **Label drift**: KS/PSI on global conversion rate and purchase distribution.
- **Concept drift**: Shadow-model comparison (predicted CTR vs actual live CTR) inside Airflow.
- Seasonal adaptation: automatic baseline refresh every Sunday + dynamic `is_peak_season` feature.

**C. dbt Migration & Standardization**
- Migrated hundreds of legacy SQL transformations into version-controlled, tested dbt models (staging → intermediate → mart).
- Added 200+ schema + statistical tests (dbt-expectations) running on every execution.

**D. Real-time Recommendation Re-architecture**
- Switched feature generation from batch to Flink + ScyllaDB serving.
- ClickHouse migrated to native Kafka Table Engines for streaming ingestion.

**E. Airflow Gating & Observability**
- Data-quality gate task after every dbt run → blocks ML training DAG via ExternalTaskSensor.
- Full incident logging + Looker dashboard showing p-value heatmaps and polluted-click percentages.

### 6. Major Challenges & How I Overcame Them

**Challenge 1**: False positives during real seasonal spikes → would have blocked legitimate traffic.  
**Solution**: Started in “warning mode”, tuned per-feature thresholds using SHAP importance, added business-rule overrides (`is_sale_period = true`).

**Challenge 2**: Performance at 5× peak traffic.  
**Solution**: 10% stratified sampling in BigQuery + Flink exactly-once + RocksDB state backend. Detection latency < 4 minutes.

**Challenge 3**: Maintaining baselines when feature list grows.  
**Solution**: Versioned GCS artifacts + automated weekly refresh job + metadata table.

**Challenge 4**: Team adoption (engineers attached to hand-written SQL).  
**Solution**: Lineage graphs, knowledge-sharing sessions, and demonstrated zero manual inspection after go-live.

### 7. Results & Business Impact (Quantified)

- Polluted clicks: 11% → 0.4%  
- Online NDCG@10 & CTR variance: reduced ~65% week-over-week  
- During three major seasonal events (Feb, Lunar New Year, April): **zero** bad models reached production  
- ML team time saved: from 2 hours daily manual checks → 5-minute review of auto-generated report  
- Estimated revenue protected: >5 billion VND across peak periods

### 8. Key Highlights & Learnings

- This is a pure **Data Engineering** success story: I protected models without writing any model code.
- Combined streaming (Flink) + batch (dbt) + observability (Alibi Detect) into one cohesive shield.
- Proved that robust data pipelines are the #1 prerequisite for reliable ML in high-velocity e-commerce.

This system is now the standard for all new ML feature pipelines at Tiki.



**My Understanding of the Drift Detection Techniques & How I Fix Them**  
*(Real-Time Recommendation Platform – Tiki)*

These four techniques were deliberately chosen because they cover **all three types of drift** that kill recommendation models in e-commerce (especially during seasonal spikes). I implemented them as a multi-layer “hard gate” inside the Flink + ScyllaDB system so that **no bad data ever reaches model training or online inference**.

Here’s exactly how each technique works, why I trust it, and — most importantly — **how I actually fix the drift** once it is detected.

### 1. Covariate Drift (Feature Distribution Shift) – KSDrift in Alibi Detect
**What it detects**: P(X) changes while P(Y|X) stays the same.  
Example: During a flash sale, `ctr_mobile_7d` suddenly jumps from mean 0.082 → 0.119.

**How the technique works** (my implementation):
- 7-day stratified 10% baseline (BigQuery → GCS artifact).
- Every day after dbt run + every 5 min in Flink: `KSDrift.predict(X_test)` on all 65 features.
- Returns per-feature p-values + overall `is_drift` flag.
- Decision: p < 0.05 on any high-SHAP feature → gate fails.

**Why I chose KSDrift (Alibi Detect)**:
- Non-parametric (works on any distribution shape).
- Multivariate + per-feature breakdown (I can tell exactly which feature drifted).
- Baseline is versioned in GCS → I can roll back instantly.
- Fast even on millions of rows (10% sample + KeOps backend).

**How I fix it**:
- **Immediate fix** (0–5 min): Airflow gate blocks the ML training DAG + Flink pauses the bad partition and routes events to `quarantine_clicks` topic.
- **Short-term fix** (same day): Add `is_sale_period` flag as a new feature in Flink (stateful) and retrain only after the baseline is refreshed.
- **Long-term fix**: Automatic weekly baseline refresh (Sunday job) + dynamic rolling bounds (±30% on high-importance features like `ctr_mobile_7d`). This is the same pattern Netflix and Uber use for holiday traffic.

### 2. Label Drift (Prior / Target Shift) – KS + PSI on Conversion Rate
**What it detects**: P(Y) changes (e.g., overall purchase rate drops from 3.2% → 1.8% during Lunar New Year).

**How the technique works**:
- Daily KS test on the global conversion rate distribution.
- PSI (Population Stability Index) on binned purchase labels (10–20 buckets).
- PSI > 0.25 or KS p < 0.05 → drift flagged.

**Why this combination**:
- KS gives statistical rigor.
- PSI is business-friendly (easy to explain to PMs: “the positive rate moved too much”).

**How I fix it**:
- **Immediate fix**: Block training + send Slack with exact PSI value and before/after distribution charts.
- **Short-term fix**: Adjust negative sampling ratio in the training dataset (increase negatives if positives dropped) and apply Platt scaling / isotonic regression on the model predictions.
- **Long-term fix**: Add a global `seasonality_factor` feature (computed in Flink from Druid rollups) so the model learns to down-weight predictions automatically during low-conversion periods.

### 3. Concept Drift (Relationship Change) – Shadow-Model Comparison
**What it detects**: P(Y|X) changes (e.g., after UI redesign, the same `query_item_affinity_score` now predicts much lower conversion).

**How the technique works**:
- A lightweight “shadow model” (same XGBoost/LambdaMART weights) runs in parallel in Airflow on the new day’s features.
- Compare predicted CTR vs actual live CTR (from ScyllaDB feedback logs).
- If divergence > 2% (or NDCG drop > 1.5%) → concept drift.

**Why this is the strongest signal**:
- It directly measures model performance degradation in production — the ultimate ground truth.
- Works even when feature distributions look normal.

**How I fix it**:
- **Immediate fix**: Block the main training DAG; promote the shadow model only if it passes.
- **Short-term fix**: Trigger an emergency micro-retrain (6-hour window instead of 24-hour) with higher weight on the last 3 hours of data (Flink state TTL adjustment).
- **Long-term fix**: Shorten the retrain window during detected high-volatility periods and add more interaction features (e.g., `post_ui_change_flag`) to let the model adapt faster.

### 4. Seasonal Adaptation (The Glue That Makes Everything Work)
**Technique**: 
- Automatic baseline refresh every Sunday night (or on `is_sale_period = true` trigger).
- Dynamic feature `is_peak_season` (computed in Flink from traffic velocity + Druid 5-min rollups).
- Two baselines maintained: 7-day normal + 30-day peak-adjusted.

**How I fix seasonal drift proactively**:
- When `is_peak_season` flips to true, the system automatically switches to the peak baseline and relaxes some thresholds (e.g., allow 40% shift on CTR features).
- After the event ends, a one-click “baseline rollback” job in Airflow restores the normal 7-day reference.
- This prevented any false-positive blocks during three major 2025 events.

### Overall Fix Workflow (The “Data Shield” I Built)
1. Drift detected (any of the three) → Airflow/Flink gate fails.
2. Event routed to quarantine + detailed incident logged in BigQuery (feature name, p-value, before/after charts).
3. Slack/PagerDuty alert + automatic Jira ticket.
4. Fix applied (flag/feature addition or emergency retrain).
5. Post-fix validation: shadow model must show <1% metric degradation before re-opening the gate.

This exact combination (KSDrift + PSI + shadow model + seasonal baselines) is what big tech uses to keep models stable at 100× scale. I implemented it entirely as a Data Engineer layer — no model code changes were required.

The result: during the highest-traffic periods in 2025, the recommendation platform stayed 100% trustworthy, and the ML team could focus on new architectures instead of firefighting bad data.


**Here’s exactly how I perform the KS (Kolmogorov-Smirnov) drift detection in the real-time recommendation platform I built at Tiki.**

I’ll break it down into the four concrete questions you asked: **what data**, **how much data**, **which window I compare**, and **how it actually runs** in the Flink + Airflow system.

### 1. What data is used for the KS test?
The KS test is run **only on the 65 engineered features** that actually feed the search and recommendation models (never on raw events).

These features come from the final **feature mart tables** in BigQuery (and the same features are also materialized in Flink state for real-time):

**Core 65 columns** (grouped):
- User behavioral (12): `rfm_score`, `session_length_sec`, `sessions_7d`, `device_type`, `user_tier`
- Item features (15): `price`, `price_discount_pct`, `inventory_level`, `category_id`, `brand_id`, `image_embedding` (128-dim)
- Interaction features (20): `ctr_7d`, `ctr_mobile_7d`, `ctr_desktop_7d`, `NDCG_precursor`, `query_item_affinity_score`, `click_rate_position_1to5`
- Contextual & temporal (18): `is_sale_period`, `time_of_day_bucket`, `trend_velocity_24h`, `days_since_last_interaction`, etc.

These are the **exact same features** the XGBoost/LambdaMART models consume. That’s why I test them — if any of them drifts, the model will be wrong.

### 2. How much data is used?
I never use the full 1.5–2 billion rows per day — that would be too slow and too sensitive.

**Baseline (reference) data**:
- 7 full days
- 10 % stratified sample (by `event_date` + `category_id` + `device_type`)
- Typical size: **1.2 – 1.8 million rows** (enough for stable CDFs, but fast to load)

**Test (current) data**:
- Batch mode (daily Airflow gate): 10 % stratified sample of **today’s partition only** (~150k–250k rows)
- Streaming mode (Flink sidecar): 5-minute rolling window sample (~20k–40k rows)

This 10 % stratified sampling is the key technique I used (exactly as we discussed earlier) to keep distributions representative without bias.

### 3. Which window I compare (the rolling baseline)
**Batch (Airflow) comparison**:
- Reference window: last **7 days** (rolling)
- Test window: **current day** (00:00 – 23:59)
- Run once per day right after dbt materialization

**Streaming (Flink) comparison** (intra-day protection):
- Reference window: last **7 days** (still the same GCS baseline)
- Test window: last **5 minutes** (sliding window)
- Run every 5 minutes inside the Flink job

This gives me both **daily safety** and **intra-day early warning** during flash sales (when drift can happen in minutes).

The baseline is stored as a versioned artifact in GCS (`gs://tiki-detector-artifacts/..._v2025-03-14/`), so I can roll back instantly if needed.

### 4. How the KS actually runs in the system
**Batch path (Airflow PythonOperator)**:
1. Load baseline detector from GCS
2. Query today’s 10 % stratified sample from BigQuery
3. `detector.predict(X_test)` → get p-values for all 65 features
4. If any high-importance feature (e.g. `ctr_mobile_7d`, `query_item_affinity_score`) has p < 0.05 → gate fails → block ML training DAG

**Streaming path (Flink sidecar)**:
- Lightweight Python process consumes the same Kafka topic
- Maintains 5-min histograms in Flink state
- Every 5 min: runs KS on the current 5-min sample vs 7-day baseline
- If drift → fires Pub/Sub message → Airflow sensor blocks downstream

### Why this specific setup works so well
- 7-day window gives stable “normal” behavior
- 10 % stratified sample prevents bias during category imbalance
- Dual batch + streaming gives both daily and intra-day protection
- Exactly the pattern used by Netflix, Uber, and Alibaba for their real-time recsys platforms

This is the exact mechanism that protected us during the three major seasonal events in 2025 — it caught the mobile CTR spike within 9 minutes and prevented any bad data from reaching the models.

**Critical detail — Stratified Sampling (the technique I used to make KS reliable at scale)**: I never use simple random sampling (RAND() < 0.1 or TABLESAMPLE). Instead, I perform **true stratified sampling directly inside BigQuery** to guarantee that the reference and test distributions perfectly preserve the original joint structure of high-cardinality and business-critical columns.

**How the stratified sampling is implemented** (the exact method I built and still run today):

SQL

```
-- Stratified 10% sample (run once per day for baseline & daily test)
SELECT *
FROM (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY event_date, category_id, device_type 
               ORDER BY RAND()
           ) AS rn,
           COUNT(*) OVER (PARTITION BY event_date, category_id, device_type) AS stratum_size
    FROM `bigquery-public-data.tiki.fct_search_features`
    WHERE event_date BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY) 
          AND CURRENT_DATE()
)
WHERE rn <= stratum_size * 0.10
```

**Why this stratified approach is essential**:

- category_id has thousands of values (some very rare but high-impact).
- device_type (mobile/desktop) is the #1 source of covariate drift during sales.
- event_date ensures temporal balance.
- Without stratification, rare categories or mobile traffic would be under-sampled → false drift alerts or missed real drift.
- Result: ~1.2–1.8 million rows for the 7-day baseline and ~150k–250k rows for today’s test — statistically representative yet fast to process.

### 1. Immediate Fix (0–5 minutes) – Hard Gate & Quarantine

When KSDrift detects covariate drift (p < 0.05 on any high-SHAP feature like ctr_mobile_7d, query_item_affinity_score, or price_discount_pct):

**Airflow side**:

- The data_drift_gate PythonOperator fails immediately after the daily dbt run.
- This failure propagates to an ExternalTaskSensor in the ML training DAG → the entire training job is blocked before it even starts.
- At the same time, a Slack alert + PagerDuty page is triggered with:
    - Exact feature(s) that drifted
    - p-value + before/after distribution charts
    - Link to the quarantined partition

**Flink side** (real-time protection):

- The Flink job receives a control signal via a dedicated Kafka control topic (or Pub/Sub trigger).
- It immediately switches the affected partition (keyed by category_id or event_date) to a side-output stream.
- All events for that partition are routed to the quarantine_clicks Kafka topic instead of the main feature pipeline.
- The Flink job continues processing healthy partitions without interruption (no full restart needed).

**What happens in quarantine**:

- Events land in a BigQuery table monitoring.quarantine_events.
- An automated Airflow DAG analyzes them (duplicate ratio, CTR spike, etc.) and creates a Jira ticket for the on-call engineer (usually me in the first month).

This whole loop from detection to quarantine completes in **under 5 minutes** — even at 5× peak traffic — because the KS gate runs on a 10% stratified sample and the Flink side-output is zero-copy.

### 2. Short-Term Fix (Same Day) – Add Contextual Flag + Emergency Retrain

Once the immediate gate is triggered, I (or the on-call engineer) apply a targeted fix within the same day:

**Step 1: Add is_sale_period flag as a new feature**

- I compute the flag in Flink using a broadcasted state (updated from Druid 5-min rollups or a marketing API).
- Logic: is_sale_period = true if traffic velocity > 2.5× 7-day median OR marketing campaign flag is active.
- I add this as a new column in the Flink MapState and in the dbt mart schema.
- Because Flink supports savepoints, I take a savepoint, update the job graph (add the new state field), and resume from the savepoint — **zero data loss**, no full restart.

**Step 2: Refresh baseline and retrain**

- I manually trigger the baseline refresh job (re-run the 7-day stratified sample query and save a new GCS artifact).
- The ML training DAG is then unblocked (I override the gate temporarily with a comment + approval).
- We run a **same-day emergency micro-retrain** using only the last 12–24 hours of data (weighted higher) so the model quickly learns the new distribution.
- The new is_sale_period feature is injected into the model so it can down-weight or boost features accordingly.

This short-term fix typically restores online NDCG/CTR within 4–6 hours of the drift event.

### 3. Long-Term Fix – Automatic Weekly Refresh + Dynamic Rolling Bounds

To stop fighting the same fires every sale, I built two automated improvements that run forever:

**A. Automatic baseline refresh (Sunday job)**

- A dedicated Airflow DAG (refresh_drift_baseline) runs every Sunday at 02:00.
- It re-executes the exact stratified sampling query over the last 7 days.
- Saves a new versioned GCS artifact (..._vYYYY-MM-DD/).
- Updates the metadata table so all detectors automatically pick up the new baseline on Monday.
- If is_peak_season was active during the week, it creates two baselines (normal + peak-adjusted).

**B. Dynamic rolling bounds instead of fixed p=0.05**

- For the top 10 high-importance features (identified by SHAP), I no longer use a hard p < 0.05.
- Instead, I compute **dynamic bounds** every day:
    - Rolling 30-day mean and std of each feature
    - Allowed range = mean ± 30% (or ± 2.5 std, whichever is wider)
- These bounds are stored in a BigQuery metadata table and injected into the Alibi Detect config at runtime.
- This is the exact pattern Netflix and Uber use for holiday traffic — it allows legitimate seasonal shifts while still catching real pollution.

**Result of the long-term fix**:

- False-positive rate during real sales dropped from ~35% to < 3%.
- The system now self-adapts to seasonality without manual intervention.
- During the three major 2025 events, we never once had to manually block training — the dynamic bounds + is_sale_period flag handled everything automatically.

This layered fix strategy (immediate hard gate → same-day contextual feature → weekly self-healing baseline) is what made the real-time recommendation platform actually reliable at scale. It protected model evaluation, kept canonical NDCG/CTR definitions trustworthy, and prevented any revenue loss from drifted features.


### 1. How Flink Handles Partial Consumption (The Built-in Isolation via Checkpoints & Watermarks)

Flink is designed for this scenario with its **checkpointing system** (which I configured with RocksDB backend for state persistence):

- Events are processed in **event-time windows** with watermarks (I set a 30-second bounded out-of-orderness to handle late events during peaks).
- Every 60 seconds (configurable at peak), Flink takes a **checkpoint** — a consistent snapshot of all state (user intent maps, CTR aggregates, etc.) + consumed Kafka offsets.
- If drift is detected (via the sidecar KS check every 5 minutes), I send a control signal to the Flink job via a dedicated Kafka control topic.
- The job then **rolls back to the last clean checkpoint** (before the bad partition started) and resumes processing — but with the bad partition flagged for side-output.

**What this means for already-consumed events**:

- Any events processed after the last clean checkpoint but before detection are **discarded from the main output stream** and replayed into the quarantine path.
- Flink's exactly-once semantics ensure no duplicates: the rollback resets the state to the clean snapshot, and the Kafka consumer offsets are adjusted to re-consume only the unflagged events.
- In practice, during the April 2025 incident (the duplicate-click bug), this isolated ~15% of the bad events that had been partially consumed — all without restarting the job or losing healthy partitions.