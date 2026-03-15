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