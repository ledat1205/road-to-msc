

**DAG ID**: `recommendation_monitoring_v2`

**Location**: `bitbucket/smartflow/dags/reco_monitoring_v2.py`

**Schedule**: Daily at **10:00 AM** (Asia/Ho_Chi_Minh), after inference DAGs complete

**Output dataset**: `tiki-dwh.recommendation_monitoring`

  

---

  

## 11 Monitoring Pillars

  

### Pillar 1 — Input Source Freshness

  

**What**: Verify the 9 upstream source tables have fresh partitions and aren't empty.

  

| Source Table | Partition Column | Max Stale Hours |

|---|---|---|

| `tiki-dwh.trackity_v2.core_events` | `date_key` | 24 |

| `tiki-dwh.dwh.dim_product_full` | `DATE(last_updated_at)` | 48 |

| `tiki-dwh.trackity_tiki.product_impressions` | `date_key` | 24 |

| `tiki-dwh.cdp.datasources` | `DATE(event_time)` | 48 |

| `tiki-dwh.olap.trackity_tiki_click` | `date_key` | 24 |

| `tiki-dwh.ecom.review` | `MONTH(created_at)` | 72 |

| `tiki-dwh.nmv.nmv` | `date` | 24 |

| `tiki-analytics-dwh.personas.product_tier` | none (unpartitioned) | 168 |

| `tiki-dwh.trackity.tiki_events_sessions_cross_devices` | `date_key` | 24 |

  

**How**: Query `INFORMATION_SCHEMA.PARTITIONS` for latest partition timestamp. Compare to `CURRENT_TIMESTAMP()`.

**Alert**: CRITICAL if stale beyond max hours; WARNING if within 80% of max hours.

  

---

  

### Pillar 2 — Row Count Day-over-Day Change

  

**What**: Compare today's table row count vs yesterday's. Detect unexpected drops/spikes instead of hard min_rows thresholds (row counts naturally vary day-to-day).

  

**Thresholds**:

| Level | Drop | Spike |

|---|---|---|

| WARNING | >20% decrease | >100% increase |

| CRITICAL | >50% decrease | >300% increase |

  

**Tables monitored** (37 total):

- 23 intermediate tables (root_product through client_cate_impression_ctr)

- 2 model tables (prediction_title_similar, input_data_train_interaction)

- 12 inference tables (product_weight through infer_pdp_widget_complementary)

  

**Exempt from day-over-day** (non-daily cadence):

- `product_weight` — always exactly 1 row (config table)

- `propensity_cate` — runs monthly on the 1st

- `prediction_interaction_similar` — runs every 2-3 days, not daily

- `product_similar_prediction_v2` — runs every 2-3 days, not daily

  

**Special alert**: Any inference `infer_*` table with 0 rows triggers CRITICAL regardless of day-over-day change. Currently `infer_pdp_widget_similar`, `infer_pdp_widget_hero`, `infer_pdp_widget_infinity_first_tab` have 0 rows — investigate.

  

---

  

### Pillar 3 — Null Rate Monitoring

  

**What**: Check key columns for unexpected NULL values. These indicate broken joins or upstream data quality issues.

  

**Critical columns per table** (from actual BigQuery schemas):

  

| Table | Null-check Columns | Why |

|---|---|---|

| `product_features_v2` | product_id, root_product_id, num_root_view_d30, sale_price, primary_cate_id, ctr_root_d30 | Master table — nulls propagate to all downstream |

| `root_product` | product_id, root_product_id, primary_cate_id, sale_price | Base product mapping |

| `product_impression_ctr` | root_product_id, product_id, ctr_root_d30 | CTR used in ranking |

| `map_client_customer` | client_id, customer_id | Broken mapping = wrong user recommendations |

| `customer_pdp_view` | customer_id, root_product_id, product_id | User history for personalization |

| `product_features_sold` | root_product_id, product_id, num_sold_root_d30 | Sales features |

| `data_home_list_personalize` | customer_id, score, product_group | Direct input to KVDal-ready tables |

  

**Alert**: WARNING if null rate >1% on any key column; CRITICAL if >5%.

  

---

  

### Pillar 4 — Duplicate Key Detection

  

**What**: Verify primary key uniqueness. Duplicates cause inflated recommendations or double-counting in aggregations.

  

| Table | Primary Key | Expected |

|---|---|---|

| `product_features_v2` | `product_id` | 1:1 |

| `root_product` | `product_id` | 1:1 |

| `product_impression_ctr` | `product_id` | 1:1 |

| `product_features_pdp_view` | `product_id` | 1:1 |

| `product_features_rating` | `product_id` | 1:1 |

| `product_features_sold` | `product_id` | 1:1 |

| `cate_features_v2` | `cate_id` | 1:1 |

| `map_client_customer` | `client_id` | 1:1 (latest customer_id per client) |

| `infer_home_widget_you_may_like` | `key` | 1:1 |

| `infer_home_widget_infinity_first_tab` | `key` | 1:1 |

  

**Alert**: WARNING if any duplicates found.

  

---

  

### Pillar 5 — Feature Drift Detection

  

**What**: Track daily statistics of key numerical features from `product_features_v2`. Detect distribution shifts that may degrade model performance.

  

**Monitored features** (from actual schema):

| Feature | Type | Why Monitor |

|---|---|---|

| `num_root_view_d30` | INT64 | View volume — seasonal, campaign-driven |

| `num_sold_root_d30` | NUMERIC | Sales volume — most volatile |

| `avg_rating_root_all` | FLOAT64 | Rating — should be stable |

| `ctr_root_d30` | FLOAT64 | CTR — sensitive to UI changes |

| `sale_price` | FLOAT64 | Price — sensitive to promotions |

| `num_impress_root_d30` | INT64 | Impression volume — traffic-dependent |

  

**Method**:

1. **SQL z-score**: Daily mean/stddev snapshot. Compare today vs 7-day rolling baseline. Z-score >2 = WARNING, >3 = CRITICAL.

2. **PSI (Population Stability Index)**: 10-bin histogram comparison via scipy in K8s pod. PSI >0.1 = WARNING, >0.2 = CRITICAL.

3. **KS test**: Kolmogorov-Smirnov test for distribution shift confirmation.

  

**Output**: `feature_stats_snapshot_YYYYMMDD` (daily stats), `drift_results_YYYYMMDD` (PSI/KS results).

  

---

  

### Pillar 6 — Model Output Quality

  

**What**: Validate model prediction outputs have correct score ranges, no nulls, and expected distributions.

  

**Key insight from actual schemas**: `prediction_title_similar` and `prediction_interaction_similar` are **pair tables** with columns `(mpid INT64, spid INT64, similar INT64)` — they have NO score column. Only `product_similar_prediction_v2` has actual scores.

  

| Model Table | Columns | Validation |

|---|---|---|

| `prediction_title_similar` (133M rows) | mpid, spid, similar | No nulls; no duplicate (mpid, spid) pairs |

| `prediction_interaction_similar` (156M rows) | mpid, spid, similar | Same; runs every 2-3 days — check staleness |

| `product_similar_prediction_v2` (195M rows) | mpid, spid, max_score_model, score_interaction, score_title, score_cate | All scores in [0, 1]; score distribution stability |

| `propensity_cate` (165K rows) | customer_id, primary_cate_id, propensity | propensity in [0, 1]; runs monthly |

  

**Score distribution checks for `product_similar_prediction_v2`**:

- Track p5/p25/p50/p75/p95 daily for each of: `max_score_model`, `score_interaction`, `score_title`, `score_cate`

- Alert if p25 ≈ p75 (distribution collapse — model may be outputting constant scores)

- Alert if out-of-range count > 0

  

**Coverage check**:

- % of salable products (from `product_features_v2 WHERE is_salable = TRUE`) that appear as `mpid` in prediction tables

- Alert if coverage < 70%

  

---

  

### Pillar 7 — Inference Output Completeness

  

**What**: The final inference tables are what gets synced to KVDal and served to users. These are the most critical tables to monitor.

  

**Home widgets**:

| Table | Latest Rows | Key Format | Check |

|---|---|---|---|

| `infer_home_widget_you_may_like` | 168K keys | customer_id / trackity_id / "_" | Key count stability; products array not empty |

| `infer_home_widget_infinity_first_tab` | 191K keys | same | same |

| `infer_home_widget_topdeal` | 124K keys | same | same |

| `infer_home_widget_import` | 36K keys | same | same |

| `infer_home_widget_repurchase` | 57K keys | same | same |

  

**PDP widgets**:

| Table | Latest Rows | Key Format | Check |

|---|---|---|---|

| `infer_pdp_widget_similar` | **0 rows** | "_{mpid}" | **CRITICAL — currently broken** |

| `infer_pdp_widget_complementary` | 821K | "_{mpid}" | Key count stability |

| `infer_pdp_widget_hero` | **0 rows** | "_{mpid}" | **Investigate if expected** |

| `infer_pdp_widget_infinity_first_tab` | **0 rows** | "_{mpid}" | **Investigate if expected** |

  

**Specific checks**:

1. **Popular key exists**: Every `infer_home_*` table must have key `"_"` (the popular/cold-start fallback). If missing, new/anonymous users get nothing.

2. **Products array not empty**: Sample 100 random keys, verify `ARRAY_LENGTH(products) > 0`.

3. **Products array size distribution**: Track avg/min/max products per key. Alert if avg drops >30% day-over-day (truncated recommendations).

4. **Rank values reasonable**: `rank` field in products array should be > 0.

  

---

  

### Pillar 8 — Propensity Model Health

  

**What**: The propensity model (`recommendation_train_user_favorite_cate_v2`) runs monthly and feeds into Home personalization via `combine_propensity`. Monthly cadence means problems persist for 30 days before the next refresh.

  

**Tables to monitor**:

| Table | Rows | Cadence | Check |

|---|---|---|---|

| `event_staging` | partitioned daily | daily | Row count per event_type (view_pdp, add_to_cart, complete_purchase, true_impression) |

| `purchase_feature` | 7.8M | monthly | Features: eventLast3M, eventLast1M — check for all-zero distributions |

| `view_feature` | 343M | monthly | Same |

| `data_train` | partitioned | monthly | Label balance: % of label=1 vs label=0. Alert if <1% positive or >50% positive |

| `propensity_cate` | 165K | monthly | Score distribution: alert if mean propensity >0.5 (model too optimistic) or <0.01 (too conservative) |

| `combine_propensity` | non-partitioned | monthly | Check staleness: compare `final_weight` distribution vs last month. Alert if >30 days since last update |

  

**Calibration check**: For propensity score buckets (0-0.1, 0.1-0.2, ..., 0.9-1.0), actual 30-day conversion rate should increase monotonically. If not, the model is miscalibrated.

  

---

  

### Pillar 9 — Training Data Quality

  

**What**: Monitor the training inputs to catch problems before they propagate to model outputs.

  

**Interaction model training data** (`input_data_train_interaction`):

| Metric | How | Alert |

|---|---|---|

| Session count | `COUNT(DISTINCT session_id)` | Day-over-day drop >20% |

| Products per session | `AVG(products_per_session)` | Should be 2-15 (filtered in pipeline). Alert if avg <3 |

| Unique products | `COUNT(DISTINCT product_id)` | Drop >10% = catalog shrinkage |

| Category diversity | `COUNT(DISTINCT cate)` per session | If avg drops below 1.5 = sessions becoming too narrow |

  

**Session co-view data** (`sale_orders`, 42K rows):

| Metric | How | Alert |

|---|---|---|

| Row count | Direct count | Very small table — any significant drop is concerning |

| Sessions with >15 products | Count | Should be 0 (filtered). If >0, filter is broken |

| Customer coverage | `COUNT(DISTINCT customer_id)` | Tracks active user base |

  

---

  

### Pillar 10 — Cross-Table Consistency

  

**What**: Verify referential integrity and join consistency across the pipeline.

  

| Check | SQL Logic | Alert |

|---|---|---|

| product_features_v2 ⊇ root_product | All product_ids in root_product should exist in product_features_v2 | Mismatch >1% = join failure |

| product_features_v2.row_count ≈ root_product.row_count | Both should be ~85M | Difference >1% |

| product_features_v2_with_weights.row_count = product_features_v2.row_count | Exact match expected | Any difference = weight join dropped rows |

| prediction mpids ⊂ product_features_v2 product_ids | All mpids in prediction tables should be valid products | >5% orphan mpids = stale predictions |

| infer_home keys ⊂ customer_pdp_view customer_ids ∪ {"_"} | Home widget keys should be known users + popular fallback | >10% unknown keys |

| combine_propensity customer_ids ⊂ customer_pdp_view customer_ids | Propensity users should have browsing history | >20% mismatch = stale propensity |

| cate_pair cate_ids ⊂ cate_features_v2 cate_ids | All cate pairs should be valid categories | Any orphan cate_id |

  

---

  

### Pillar 11 — SHAP / Model Explainability (Weekly)

  

**What**: Weekly feature importance analysis to understand what drives model predictions. Runs on Sundays only.

  

**LightGBM (propensity_cate model)**:

- `shap.TreeExplainer` on 10K sampled rows from `data_train`

- Output: top-20 feature importances

- Alert: If a single feature contributes >80% of importance (model is essentially a single-rule classifier)

- Currently only 4 features: eventView3M, eventView1M, eventPurchase3M, eventPurchase1M — track if relative importance shifts significantly

  

**Matrix factorization (interaction/title models)**:

- Factor norm analysis as proxy for feature importance

- Track top-20 products by embedding magnitude (these dominate recommendations)

- Alert: If top-20 products are all from same category (model is category-biased)

  

---

  

## Additional Monitoring Suggestions

  

Beyond the 11 implemented pillars, these are valuable to add over time:

  

### A. Recommendation Diversity Metrics

  

**Why**: Even if models are "correct", low diversity leads to filter bubbles and poor user experience.

  

| Metric | SQL | Target |

|---|---|---|

| **Catalog coverage** | `COUNT(DISTINCT spid in all infer_* tables) / COUNT(DISTINCT product_id WHERE is_salable)` | >30% of catalog appears in at least one recommendation |

| **Gini coefficient** | Measure how evenly products are distributed across recommendations | <0.8 (1.0 = one product dominates) |

| **Intra-list diversity** | For each user's recommendation list, avg pairwise category distance | Track trend, alert on sustained drop |

| **Category concentration** | `% of recommendations from top-3 categories` per user | <60% ideal |

| **Price range diversity** | Stddev of `sale_price` within each user's recommendation list | Alert if approaching 0 (all same price) |

  

### B. Cold Start Performance

  

**Why**: The popular fallback (key="_") serves all new/anonymous users. Its quality directly impacts first impressions.

  

| Metric | How | Alert |

|---|---|---|

| Popular key freshness | Check if products in "_" key are still salable | >10% unsalable = stale |

| Popular key category spread | Categories represented in popular list | <5 categories = too narrow |

| % traffic hitting popular | From serving logs: requests with key="_" / total requests | Track trend; if >40%, personalization reach is low |

| Popular vs personalized CTR gap | Compare CTR of "_" recommendations vs personalized | If gap >50%, cold-start experience is significantly worse |

  

### C. Serving-Side Monitoring (requires smarter-model-serving logs)

  

**Why**: Batch pipeline may be healthy but serving layer may fail silently.

  

| Metric | Source | Alert |

|---|---|---|

| Cassandra lookup latency p99 | Serving metrics | >100ms |

| Cache miss rate | Serving metrics | >20% = KVDal sync issue |

| Empty response rate | Serving logs | >5% of requests return 0 products |

| Rerank demotion rate | `ReRank` metrics | % of products demoted (≥2 impressions, 0 clicks). If >50%, recommendations are mostly stale |

| FlatBuffer decode errors | Serving error logs | Any = data format mismatch |

  

### D. Temporal Staleness Tracking

  

**Why**: The interaction model uses 90 days of session data equally weighted. Products' recommendation relevance decays over time.

  

| Metric | SQL | Alert |

|---|---|---|

| Avg age of recommended products | For products in `infer_home_*`, avg `DATE_DIFF(CURRENT_DATE(), created_at, DAY)` from root_product | If avg age >180 days, recommendations are stale |

| % of reco products with 0 views in last 7 days | Join infer products with product_features_v2 WHERE num_root_view_d7 = 0 | >30% = recommending dead products |

| % of reco products out of stock | Join with product_features_v2 WHERE is_salable = FALSE | >5% = not filtering properly |

| Newest product in reco lists | MAX(created_at) from recommended products | If >7 days old, new products aren't entering recommendations |

  

### E. User-Level Recommendation Freshness

  

**Why**: Pre-computed recommendations are frozen for the entire day. Users who browse heavily in the morning still see last night's recommendations in the afternoon.

  

| Metric | SQL | Alert |

|---|---|---|

| Users with >10 daily actions | Count from pdp_view_raw WHERE date_key = today | Track volume of "power users" affected by staleness |

| Overlap: morning reco vs afternoon clicks | For users with morning + afternoon activity, % of afternoon clicks that were in their infer_home list | If <20%, recommendations are largely irrelevant by afternoon |

| Recommendation refresh lag | Time between pipeline completion and next user session | Track: if most users open app before pipeline finishes (~noon), they see 2-day-old recommendations |

  

### F. Feedback Loop Monitoring

  

**Why**: The models train on organic user behavior, not on recommendation performance. Bad recommendations persist until features naturally decay.

  

| Metric | SQL | Alert |

|---|---|---|

| Zero-click products | Products with >1000 impressions (from product_impression_ctr) and 0 clicks, that appear in infer_* tables | Count; track "bad recommendation half-life" |

| Impression-to-click ratio by source | From product_impressions.reco.metadata.widget_code, calculate CTR per widget | Widget CTR drop >20% week-over-week |

| Recommendation churn rate | % of products in today's infer_home vs yesterday's that are different | <5% = recommendations barely changing; >50% = too volatile |

  

---

  

## DAG Task Graph

  

```

ExternalTaskSensor(wait for recommendation_infer_home_v2)

├── [input_freshness] >> [input_schema_check]

├── [intermediate_row_counts] >> [intermediate_null_rates] >> [intermediate_duplicate_check]

├── [feature_stats_snapshot] >> [feature_drift_comparison_sql, drift_detector_pod]

├── [model_score_distribution, model_coverage, model_output_validation] (parallel)

└── [shap_analyzer_pod] (weekly only, via BranchPythonOperator)

All terminal tasks >> [alert_evaluator_pod]

```

  

---

  

## Consolidated Alerting

  

The `alert_evaluator.py` (K8s pod) runs as the final task. It:

  

1. Reads all monitoring output tables for today's date

2. Compares against thresholds in `params/thresholds.yml`

3. Evaluates 6 check categories:

- Input Data Quality (freshness)

- Row Count Changes (day-over-day %)

- Intermediate Tables (null rates + duplicates)

- Feature Drift (z-score + PSI)

- Model Output Quality (score validation + coverage)

4. Sends single Slack summary with RED/YELLOW/GREEN per category

  

**Slack message format**:

```

*Recommendation Monitoring Report — 20260317*

🔴 *Input Data Quality*: core_events stale (26h > 24h max)

🟢 *Row Count Changes*: All tables within normal range

🟡 *Intermediate Tables*: product_features_v2.null_rate_ctr_root_d30=6.2%

🟢 *Feature Drift*: No significant drift detected

🔴 *Model Output Quality*: infer_pdp_widget_similar has 0 rows

  

*Overall Status*: 🔴 RED

```

  

---

  

## File Structure

  

```

Olympus/recommendation/monitoring/

├── config.py # Config loader, K8s configs, BQ defaults

├── params/

│ ├── general.yml # Schedule (10am), concurrency, BQ config

│ └── thresholds.yml # Day-over-day %, null rate limits, drift thresholds, score ranges

├── pipeline/

│ ├── input_validation.py # TaskGroup: source table freshness + schema

│ ├── intermediate_checks.py # TaskGroup: row counts → null rates → duplicates

│ ├── feature_drift.py # TaskGroup: stats snapshot → z-score + K8s PSI/KS

│ ├── model_output_quality.py # TaskGroup: score distribution, coverage, validation

│ └── shap_explainability.py # TaskGroup: weekly SHAP via K8s pod

├── sql/monitoring/

│ ├── input_freshness.sql # Source table partition recency

│ ├── input_schema_check.sql # Expected columns exist

│ ├── intermediate_row_counts.sql # Today vs yesterday row counts + pct_change

│ ├── intermediate_null_rates.sql # Null rates on key columns (actual column names)

│ ├── intermediate_duplicate_check.sql # Primary key uniqueness

│ ├── feature_stats_snapshot.sql # Mean/stddev/percentiles for 6 features

│ ├── feature_drift_comparison.sql # Z-score vs 7-day rolling baseline

│ ├── model_score_distribution.sql # Score stats for product_similar_prediction_v2 (4 score columns) + propensity_cate

│ ├── model_coverage.sql # % salable products with predictions (uses mpid not product_id)

│ └── model_output_validation.sql # Null/range/duplicate checks (pair tables vs score tables)

└── script/

├── drift_detector.py # PSI (10-bin) + KS-test via scipy

├── shap_analyzer.py # SHAP TreeExplainer + factor norm analysis

└── alert_evaluator.py # Reads all tables, day-over-day check, Slack summary

  

dags/

└── reco_monitoring_v2.py # DAG with ExternalTaskSensor → 5 TaskGroups → alert pod

```

  

---

  

## Key Findings from Schema Analysis (2026-03-16)

  

These observations informed the monitoring design:

  

1. **prediction_title_similar and prediction_interaction_similar have NO score column** — they are pair tables `(mpid, spid, similar)`. Previous monitoring assumed a `score` column that doesn't exist.

  

2. **prediction_interaction_similar runs every 2-3 days**, not daily. Latest partition is 20260310 while today is 20260317. Day-over-day row count checks must exempt this table.

  

3. **product_weight is a single-row config table** with 13 FLOAT64 weight columns. It must always have exactly 1 row.

  

4. **3 PDP inference tables have 0 rows**: `infer_pdp_widget_similar`, `infer_pdp_widget_hero`, `infer_pdp_widget_infinity_first_tab`. This is likely a pipeline issue or these widgets were disabled.

  

5. **Tables 18-24 ("not yet used") are actually still computed daily** with significant row counts (e.g., client_impression_ctr: 9.8M rows). They consume compute daily.

  

6. **propensity_cate runs monthly** (partitions: 20260201, 20260101, etc.) with declining row counts (165K → 513K over 14 months). The shrinking user base warrants investigation.

  

7. **sale_orders is very small** (42K rows) relative to pdp_view_raw (12.9M). This is the session co-view data that feeds the interaction model — its small size may limit model quality.

  

8. **product_features_v2 has 120+ columns** but only 6 are monitored for drift. Consider adding: `num_root_view_d7` (more volatile), `num_cus_root_d30` (customer engagement), `price_segment_id` (distribution shift).

  

9. **combine_propensity is not partitioned** — it gets overwritten monthly. There's no historical record of propensity changes. Consider writing to a date-suffixed table instead.

  

10. **vw_related_product_v2 is a VIEW** over `related_product_v2` with `UNNEST(complementary_categories)`. If the source table is stale, PDP complementary recommendations are stale — but there's no freshness check on it.