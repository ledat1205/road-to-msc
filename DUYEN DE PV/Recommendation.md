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