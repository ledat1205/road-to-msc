Airflow ecosystem ownership & dbt migration at Tiki (Python, Go – April 2025–present) At Tiki I took full ownership of an Airflow deployment with 300+ DAGs that orchestrated end-to-end analytics and ML pipelines. I identified that our legacy BigQuery/SQL transformations were scattered and hard to maintain, so I migrated everything into dbt. This standardized feature logic, added version-controlled models, and made it trivial for the ML team to iterate on new features without breaking downstream models. I also introduced data validation checks and quality monitors inside the DAGs (using Great Expectations-style tests) so we could catch drift before models were retrained.

**What datasources were involved?** Tiki’s data platform is a classic e-commerce mix. The raw sources feeding the 300+ DAGs were:

- Transactional MySQL/PostgreSQL databases (orders, users, products, inventory, search queries, cart events)
- Kafka topics (real-time user events, clickstreams, recommendation feedback, A/B test traffic — about 15–20 high-volume topics)
- CDC streams via Debezium (from the above DBs into Kafka)
- BigQuery itself (some legacy tables that were already loaded via previous pipelines)
- A few external sources (marketing platforms, payment gateways — ingested via API + batch files)

All of these ultimately landed in BigQuery as the central analytics warehouse for both analytics dashboards and ML feature stores.

**What SQL did we need to unify into dbt?** Before the migration, almost every ML and analytics pipeline had its own hand-written SQL transformations directly inside Airflow DAGs. Typical examples I migrated:

- Raw event → sessionized user features (hundreds of lines of window functions, JSON parsing, deduplication)
- Product catalog + inventory → enriched item features (joins across 8+ tables, slowly-changing dimension logic)
- Search & recommendation features (CTR, NDCG precursors, query-item affinity, category embeddings)
- User behavioral aggregates (daily/weekly rolling windows for recency, frequency, monetary)
- A/B test exposure + conversion tables

These were scattered as:

- BigQuery views (hundreds of them)
- Raw SQL tasks in Airflow (PythonOperator or BigQueryOperator with inline SQL)
- Some materialized tables that were manually refreshed

The goal was to bring every single one of these into dbt models so the ML team could treat features as code.

**Why dbt? What features made it the obvious choice?** I pushed for dbt because the legacy setup was becoming a maintenance nightmare for ML iteration:

- No version control — changes to a feature broke models silently
- No built-in testing — we only discovered bad data after a model retrained and metrics dropped
- Massive duplication (same “active user” logic copied in 20+ DAGs)
- No lineage or documentation — new ML engineers spent days just figuring out “where does this feature come from?”
- Hard to do incremental loads at scale (Tiki has billions of rows)

dbt solved all of that with:

- Git-native workflow (models in .sql + .yml files, PR reviews, CI/CD)
- Built-in testing framework (schema tests, row-count tests, custom SQL tests, dbt-expectations package)
- Incremental models + materializations (table, incremental, view) with automatic partitioning on date/user_id
- Macros + Jinja for DRY code (one “sessionize” macro used everywhere)
- dbt docs + lineage graph (super useful when ML team asks “why did CTR drop?”)
- Exposures + sources.yml so downstream ML pipelines could declare dependencies cleanly
- Easy integration with Airflow (dbt run via BashOperator or official dbt-airflow plugin)

Once I saw the lineage graph and the test coverage jump from ~0% to >95%, it was clear this was the right move.

**How did the migration actually happen?** I did it in three phases over ~4 months (while still keeping all 300 DAGs running):

1. **Inventory & extraction (2 weeks)**
    - Wrote a Python script (using BigQuery INFORMATION_SCHEMA) to discover every view and every SQL task in Airflow DAG code.
    - Grouped them into logical domains (user, item, session, search, recsys).
    - Created dbt sources.yml for all raw tables.
2. **Model conversion (6 weeks)**
    - One-by-one, I extracted the SQL, refactored it into dbt models (staging → intermediate → mart).
    - Added incremental logic everywhere possible (using dbt’s is_incremental() + merge strategy).
    - Created shared macros for common logic (e.g., one macro for “calculate NDCG precursors”).
    - Added 200+ tests (not_null, unique, accepted_values, custom statistical checks).
3. **Cutover & parallel run (6 weeks)**
    - Deployed dbt models side-by-side with legacy SQL.
    - Created “shadow” DAGs that ran both and compared output row-by-row using a custom PythonOperator (hash comparison on sample partitions).
    - Once confidence was 100% for 7 days, I switched the downstream ML pipelines to point to dbt mart tables and deprecated the old SQL tasks.

Everything stayed in Git, with proper branching (feature/dbt-model-xxx) and code reviews.

**Biggest difficulties I hit**

- Performance surprises: Some complex window functions that ran fine as one-off views exploded when turned into incremental dbt models on 10B+ rows. I had to rewrite several using partitioned incremental + clustering on user_id/date.
- Dependency hell: A few legacy views depended on other legacy views in circular ways — had to break cycles manually.
- Data type drift: BigQuery was forgiving with implicit casts; dbt is strict, so I had to add explicit CASTs in 50+ places.
- Testing at scale: Running full dbt test on the entire warehouse took >2 hours initially — I optimized with dbt’s --select and sample-based tests for CI.
- Team buy-in: Some engineers were attached to their hand-crafted SQL; I had to do knowledge-sharing sessions and show the lineage graph to win them over.

**Why catch drifts before model retraining — and how I did it** At Tiki, our recommendation and search models retrain daily. If feature distribution drifts (e.g., sudden spike in mobile traffic, new category launch, or upstream bug), the model can degrade silently before the next training cycle. Catching it early prevents “garbage in, garbage out” and lets us either fix the pipeline or pause/retrain immediately.

How I implemented it inside the dbt + Airflow setup:

- Added dbt tests that run on every model execution (not just schema — statistical too):
    - Expected value ranges for CTR, NDCG precursors
    - z-score checks on numeric features
    - cardinality checks on categorical columns
    - freshness tests on sources (dbt source freshness)
- Used dbt-expectations package for advanced tests (e.g., expect_column_values_to_be_between with dynamic bounds based on 30-day rolling stats).
- In Airflow, I added a “data-quality-gate” task right after dbt run:
    - If any test fails, the DAG fails and sends Slack alert + blocks the downstream ML training DAG.
    - Also built a lightweight Great-Expectations-style custom check in Python that compares current day distribution vs 7-day baseline using Kolmogorov-Smirnov test (threshold 0.05).
- For real-time features (Kafka-fed), I added a streaming monitor in Flink that publishes drift metrics to a separate BigQuery table, which then triggers Airflow sensors.

This caught three major incidents in the first two months (one upstream schema change, one traffic anomaly during a sale, one duplicate event bug) before any model was retrained on bad data. The ML team loved it because their offline metrics stayed stable.


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