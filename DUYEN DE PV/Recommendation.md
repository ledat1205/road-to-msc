**Purpose:** Single source of truth for the ~65–70 engineered features that power both production models. This document describes the feature mart tables, the observability layer, the automated protection mechanisms, and how everything stays trustworthy even during 3–5× traffic spikes (flash sales, Lunar New Year, etc.).

### 1. Overview & Architecture
We operate **two main production models** that retrain **daily** on the exact same feature mart tables:

- **Search Ranking Model** – LambdaMART / XGBoost + deep reranking layers (learning-to-rank)  
- **Product Recommendation Model** – Two-tower candidate generation + session-based reranker (collaborative filtering + deep learning)

**Training data volume:** ~1.5 billion query-item-user rows per day (joined from feature marts + click / add-to-cart / purchase labels).

**Core principle:**  
**No raw tables, no external signals.** Every feature used by ML comes from the dbt feature marts I own:  
`fct_search_features`, `fct_rec_features`, `dim_user`, `dim_item`.

**High-level architecture (hybrid batch + streaming):**
```
Raw sources + CDC
        ↓
Kafka (clicks, impressions, purchases)
        ↓
Flink (real-time cleaning + polluted-click guard)
        ↓
dbt (incremental, partitioned on event_date + user_id)
        ↓
BigQuery / ClickHouse feature marts
        ↓
Daily training DAG + shadow-model gate
        ↓
Production models
```

The **protection layer** (the “shield”) sits across the entire flow and guarantees zero bad models reach production.

### 2. Complete Feature Inventory (the only features ML uses)

#### 2.1 User Features (~12)
- `rfm_score` (7d / 30d / 90d)  
- `session_length_sec`, `sessions_7d`, `avg_session_depth`  
- `device_type`, `time_of_day_bucket`, `user_tier` (new/active/loyal)  
- `geo_region`  
- `historical_category_affinity` (128-dim embedding)

#### 2.2 Item Features (~15)
- `price`, `price_discount_pct`, `inventory_level`, `is_in_stock`  
- `category_id`, `category_path` (full hierarchy), `brand_id`  
- `rating_avg`, `review_count_30d`  
- `image_embedding` (128-dim MobileNet)  
- `popularity_score_7d` (views + sales)

#### 2.3 Query / Context Features (~10)
- `query_length`, `query_embedding` (BERT)  
- `query_item_text_match_score` (BM25 + dense)  
- `is_sale_period` (flag added after Feb incident)  
- `platform` (app/web), `referrer_type`

#### 2.4 Interaction / Behavioral Features (strongest signals – ~20)
- `ctr_7d`, `ctr_30d`, `ctr_mobile_7d`, `ctr_desktop_7d` ← critical feature that broke during flash sale  
- `NDCG_precursor` (position-discounted click probability)  
- `click_rate_position_1to5`, `add_to_cart_rate`  
- `query_item_affinity_score`  
- `session_click_sequence` (last 5 clicks → embedding)  
- `conversion_rate_7d`

#### 2.5 Temporal Freshness Features
Every numeric feature above exists in **7d / 14d / 30d / 90d** rolling windows.  
Additional freshness signals: `days_since_last_interaction`, `trend_velocity_24h`.

**Total per query-item pair:** 65–70 fully engineered, materialized features.  
All built as **incremental dbt models**, partitioned daily.

### 3. Daily Monitoring & Observability Layer (the shield that prevents bad data)

#### 3.1 Daily Distribution Monitoring (dbt + KS gate)
- Every dbt run runs **40+ dbt-expectations tests**.  
- After dbt finishes, `data_drift_gate` PythonOperator executes:  
  - Kolmogorov-Smirnov (numeric)  
  - Chi-square (categorical)  
  - Custom ±30 % bounds on high-importance features (e.g. `ctr_mobile_7d`)  
- If any test fails → **entire training DAG is blocked automatically**.

#### 3.2 Feature Importance Tracking (SHAP)
- After every retrain: SHAP explainer on 1 % sample.  
- Results stored in `monitoring.feature_importance_history`.  
- Weekly Monday review with ML team: “ctr_7d importance dropped from 0.18 → 0.09 — shall we retire/boost it?”

#### 3.3 Real-time Drift Dashboard (Flink + Looker)
- Flink job computes histograms every 5 minutes for streaming features.  
- Looker dashboard: live vs 7-day baseline (red/yellow/green) for top 15 features.  
- PagerDuty alerts on high-importance features.

#### 3.4 Model Impact Simulation (pre-retrain gate)
- Shadow model runs on new day’s features vs previous day.  
- If predicted NDCG or CTR changes >2 % → gate fails + Slack report listing the exact culprit features.

#### 3.5 Post-Training Validation
- Compare offline NDCG@10 / AUC vs previous 7 days.  
- Drop >1.5 % (even if drift tests passed) → deep investigation.

### 4. Key Protection Components (detailed implementation)

#### 4.1 Polluted Clicks Detection & Deduplication
**Real-time (Flink):**  
- 60-second tumbling window, keyed by `click_id + user_id`.  
- Duplicate `click_id` or click/impression ratio > 3× rolling median → dead-letter topic + metric.  

**Batch (dbt):**  
```sql
QUALIFY ROW_NUMBER() OVER (PARTITION BY click_id ORDER BY timestamp) = 1
AND click_timestamp > impression_timestamp + 200ms
AND session_duration < 3600s
```

**Scalability trick at 5× peak:** RocksDB state backend + exactly-once + 10 % sampling.  
**Result:** polluted clicks dropped from ~11 % to 0.4 %.

#### 4.2 Multi-Type Drift Detection Engine
- **Covariate drift** – feature distributions (KS)  
- **Label drift** – conversion rate  
- **Concept drift** – shadow-model prediction vs actual CTR  

**Daily gate** uses Alibi Detect-style logic on versioned 7-day baseline (GCS).  
**Seasonal adaptation:**  
- Auto baseline refresh every Sunday OR when `is_sale_period = true`.  
- Dynamic rules: allow drift if sale flag is active.  

**False-positive solution:** started in “warning mode” for 2 weeks, tuned p-value per SHAP importance, added business rules.

#### 4.3 Integration & Observability
- Everything version-controlled in Git (dbt + Airflow + Flink).  
- Looker dashboard: p-value heatmap, polluted-click %, drift incidents.  
- Code review enforced across 300+ DAGs.

### 5. Results & Business Impact
**During 3 major events (Feb flash sale, Lunar New Year, April promotion):**  
- Zero bad models reached production.  
- Polluted clicks: 11 % → 0.4 %.  
- Online NDCG@10 + CTR variance reduced ~65 % week-over-week.  
- ML team moved from “daily manual inspection” to “trust the pipeline”.

This is the exact **end-to-end feature store + data-quality shield** that turned chaotic peak-season data into trustworthy training data every single day.

### Next Steps for You (how to turn this into your final docs)
1. Copy this structure into Notion / Confluence / GitHub wiki.  
2. Add a **Feature Dictionary** page with one row per feature (name, description, table, owner, importance rank, drift threshold).  
3. Add **ER diagram** of the 4 main marts (I can help sketch it).  
4. Add **Runbook** section for “What to do when the drift gate fires”.  




**Here’s a deep dive into the 3 main types of drift** that every production ML system (like our Tiki search & recommendation models) must monitor. These are the ones that directly impact daily retraining and online metrics (NDCG, CTR, conversion).

I’ll cover each one with:

- Clear definition
- Real-world use case in e-commerce recsys (Tiki-style)
- Concrete example
- Business & model implications
- How we detect + resolve it (linking back to the tools we discussed)

### 1. Covariate Drift (also called Data Drift or Feature Drift)

**Definition**: The input features change their distribution (P(X) shifts), but the relationship between features and target stays the same. The model is now seeing data it has never seen in training.

![Multivariate drift detection using Domain Classifier with a Python tutorial](https://usenotioncms.com/proxy/block/37282503-e443-4f17-8f67-f6ba9cd9709a%2Fdb33a217-8812-4b9f-a623-d33b0faeac96%2F63e4d430549d4a26879bdad9_Frame_2-p-800.jpg)

[nannyml.com](https://www.nannyml.com/blog/data-drift-domain-classifier)

![Navigating Data Drift in the AI Ecosystem — A Practical Guide to Detection  Methods and Monitoring Tools | by Ravi Shankar Shukla | Medium](https://miro.medium.com/v2/resize:fit:2000/1*RES-1YX16AnbwWy6h2Lcug.png)

[medium.com](https://medium.com/@shukla.shankar.ravi/navigating-data-drift-in-the-ai-ecosystem-a-practical-guide-to-detection-methods-and-monitoring-cc82a3911885)

**Use case in recsys**: User behavior or catalog changes (new categories, mobile traffic spikes, pricing experiments).

**Tiki example (the one we discussed)**: Flash-sale day → ctr_mobile_7d distribution suddenly shifts right (mean 0.082 → 0.119). The feature table that feeds ranking changes, but the true relevance logic is the same.

**Implications**:

- Model overfits to the temporary distribution → wrong feature weights (e.g., mobile CTR becomes over-weighted).
- Offline NDCG looks fine, but online CTR/conversion drops 2–5 % within hours.
- Revenue loss: we calculated ~1.4 billion VND in one morning from the Feb 2025 incident.

**Detection & Resolution**:

- Detect: KS test / Alibi Detect KSDrift on every engineered feature (exactly what we built).
- Resolve:
    - Short-term: block training DAG (our Airflow gate).
    - Long-term: add new features (e.g., is_sale_period flag), refresh baseline weekly, or use domain-adaptation techniques.

### 2. Concept Drift

**Definition**: The relationship between features and target changes (P(Y|X) shifts) while the feature distribution may stay stable. The “rules of the game” have changed — what used to be a strong signal is no longer predictive.

![An introduction to Model drift in machine learning - UbiOps - AI model  serving, orchestration & training](https://sp-ao.shortpixel.ai/client/to_webp,q_glossy,ret_img,w_828,h_466/https://ubiops.com/wp-content/uploads/2022/03/What-is-concept-drift.png)

[ubiops.com](https://ubiops.com/an-introduction-to-model-drift-in-machine-learning/)

An introduction to Model drift in machine learning - UbiOps - AI model serving, orchestration & training

**Use case in recsys**: Sudden external events or UI/product changes that alter user intent (seasonal trends, competitor actions, algorithm updates on the platform).

**Tiki example**: After a major app redesign (new recommendation carousel layout in March 2025), users started clicking differently. The same query_item_affinity_score now mapped to much lower conversion. The feature distribution stayed identical, but the true probability P(click | affinity) dropped 18 %.

**Implications**:

- Model predictions become systematically wrong (high confidence on items that no longer convert).
- Offline metrics may stay stable for days (because training data also slowly drifts), but live A/B tests crash.
- Worst case: entire ranking model becomes useless until retrained on new concept — we saw 4–7 % drop in site-wide conversion until we reacted.

**Detection & Resolution**:

- Detect: Monitor prediction vs actual labels (e.g., predicted CTR vs real CTR on live traffic) or use drift detectors on residuals. We added a daily “shadow model” comparison in Airflow.
- Resolve:
    - Immediate: emergency retrain with higher weight on recent data.
    - Long-term: shorter retrain windows (we moved from daily to 6-hour micro-batches during high-volatility periods), or switch to online learning (Flink-based incremental updates).

### 3. Label Drift (also called Prior Drift or Target Drift)

**Definition**: The distribution of the target variable itself changes (P(Y) shifts). The overall “positive rate” in the world changes, even if features and relationships stay the same.

**Use case in recsys**: Seasonal or event-driven changes in overall user behavior (Black Friday, new user influx, economic downturn).

**Tiki example**: During Lunar New Year 2025, overall purchase rate (the label for our conversion models) dropped from 3.2 % to 1.8 % across the entire site because users were window-shopping more. The feature distributions (ctr_7d, price, etc.) were normal, and the feature-to-label mapping was unchanged — but there were simply far fewer positive labels.

**Implications**:

- Model calibrated on old prior over-predicts positives → too many items ranked high that don’t convert.
- Precision drops dramatically; recall may look okay.
- Business impact: wasted impressions, lower ad revenue, user frustration (too many irrelevant recommendations). We saw a 12 % drop in add-to-cart rate until we adjusted.

**Detection & Resolution**:

- Detect: Simple statistical tests on the label distribution (KS or PSI on the target column itself) + monitoring of global metrics (site-wide CTR, conversion rate).
- Resolve:
    - Re-calibrate the model ( Platt scaling or isotonic regression on new prior).
    - Adjust negative sampling ratio in training.
    - Add a global “seasonality_factor” feature that the model can learn.

### Quick Comparison Summary

|Type|What changes|Typical trigger at Tiki|Detection method we used|Most dangerous because…|
|---|---|---|---|---|
|Covariate|P(X)|Traffic spike, new categories|KS / Alibi Detect on features|Breaks feature importance instantly|
|Concept|P(Y|X)|UI change, competitor move|Prediction vs actual monitoring|
|Label / Prior|P(Y)|Holidays, economic events|Label distribution + global KPIs|Model becomes poorly calibrated|

All three types can happen at the same time (we saw covariate + label drift together during the flash sale).

This is why we built the multi-layer monitoring we discussed earlier — one simple KS gate on features only catches #1. We layered on prediction monitoring and label checks to cover the full picture.