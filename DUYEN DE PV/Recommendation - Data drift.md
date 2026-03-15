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

I’m happy to walk through any section in more depth, share GitHub structure examples, dbt model YAML snippets, Flink job logic, or the exact Looker dashboard queries. This project best represents my ability to deliver production-grade, reliable data platforms that directly drive business outcomes.

Ready for questions!