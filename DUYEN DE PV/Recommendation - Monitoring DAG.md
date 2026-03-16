
  

**DAG ID**: `recommendation_monitoring`

**Location**: `bitbucket/smartflow/dags/reco_monitoring.py`

**Schedule**: Daily at **08:00 AM** (Asia/Ho_Chi_Minh)

**Dependencies**: Runs after aggregation_daily_v2 + train DAGs complete

  

---

  

## Overview

  

Comprehensive monitoring DAG for the recommendation pipeline. Tracks:

- **Input data health**: Row counts, unique values, coverage across all 24 aggregated tables

- **User feature quality**: Customer/client behavioral signal distributions (views, clicks, purchases, cart)

- **Prediction output**: Prediction row counts, score distributions, coverage ratios

- **Day-over-day changes**: Anomaly detection via % changes across all tables

  

---

  

## DAG Flow

  

```

wait_for_pipeline_outputs

(sensors: root_product + product_similar_prediction_v2)

↓

[input_data_health, user_feature_quality, prediction_health] (parallel)

↓

day_over_day_comparison

```

  

---

  

## Output Tables (all date-partitioned: `_YYYYMMDD`)

  

### 1. `monitoring_input_data_health_YYYYMMDD`

**File**: `sql/monitoring/input_data_health.sql`

**Metrics**: Row counts + unique value counts for all tables

  

| Category | Tables | Metrics |

|----------|--------|---------|

| Product Features (9) | root_product, product_sold_raw, product_features_sold, product_features_rating, pdp_view_raw, product_features_pdp_view, product_impression_raw, product_impression_ctr, product_features_v2 | count, unique_root, salable_count, avg_metrics |

| Customer Data (7) | map_client_customer, customer_purchased, customer_cart_estimated, customer_pdp_view, customer_cate_pdp_view, customer_impression_ctr, customer_cate_impression_ctr | count, unique_customer |

| Client Data (4) | client_pdp_view, client_cate_pdp_view, client_impression_ctr, client_cate_impression_ctr | count, unique_client |

| Cate Features (2) | cate_features_v2, cate_pair | count, unique_cate |

| Model Inputs | sale_orders, input_data_train_interaction, event_staging | count, unique_session, unique_product |

  

**Example columns**:

- `root_product_count`, `root_product_unique_root`, `root_product_salable_count`

- `avg_sold_root_d30`, `avg_root_view_d30`, `avg_ctr_root_d30`

- `product_features_v2_count` (should match salable count)

- `sale_orders_count`, `sale_orders_unique_session`, `sale_orders_unique_product`

  

---

  

### 2. `monitoring_user_feature_quality_YYYYMMDD`

**File**: `sql/monitoring/user_feature_quality.sql`

**Metrics**: Behavioral signal distributions for customer vs client users

  

| Signal Type | Customer | Client | Metrics |

|------------|----------|--------|---------|

| PDP Views | customer_pdp_view (90/30/7d) | client_pdp_view (90/30/7d) | unique_users, avg/median/p99 views |

| Cate Views | customer_cate_pdp_view | client_cate_pdp_view | avg views, users_per_cate |

| Impression/CTR | customer_impression_ctr (30d) | client_impression_ctr (30d) | unique_users, overall_ctr |

| Purchase | customer_purchased (365d) | - | unique_customers, avg_purchases, avg_unique_products |

| Cart | customer_cart_estimated (365d) | - | unique_customers, avg_cart_items |

| Mapping Quality | map_client_customer | - | unique_customer, unique_client, client_to_customer_ratio |

  

**Example columns**:

- `customer_pdp_unique_customer`, `avg_customer_view_root_d30`, `median_customer_view_root_d30`, `p99_customer_view_root_d30`

- `customer_cate_view_unique_customer`, `avg_cate_per_customer` (diversity)

- `overall_customer_ctr_d30` (aggregate CTR for all customers)

- `purchased_unique_customer`, `avg_unique_root_per_customer` (purchase diversity)

- `cart_unique_customer`, `avg_cart_items_per_customer`

- `client_to_customer_ratio` (multi-device users)

  

---

  

### 3. `monitoring_prediction_health_YYYYMMDD`

**File**: `sql/monitoring/prediction_health.sql`

**Metrics**: Model output quality and coverage

  

| Model | Metrics |

|-------|---------|

| Interaction (view-based co-purchase) | total_rows, unique_mpid, unique_spid, avg/p25/p50/p75/p99_score |

| Title (product name similarity) | total_rows, unique_mpid, unique_spid, avg/p25/p50/p75/p99_score |

| Combined | total_rows, unique_mpid, **product_coverage_ratio** |

  

**Example columns**:

- `interaction_pred_total_rows`, `interaction_pred_avg_score`, `interaction_pred_p50_score`

- `title_pred_total_rows`, `title_pred_unique_mpid`

- `combined_pred_total_rows`, `product_coverage_ratio` (% salable products with recommendations)

  

**Alert thresholds**:

- Coverage < 70%: many products without recommendations

- Score distribution collapse (p25 ≈ p75): model may be stuck

- Total rows drop >20%: pipeline may have failed upstream

  

---

  

### 4. `monitoring_day_over_day_YYYYMMDD`

**File**: `sql/monitoring/day_over_day_comparison.sql`

**Metrics**: Today vs yesterday row counts + % change

  

Contains 3 column sets:

- `today_*`: Row counts for today's date partition

- `yday_*`: Row counts for yesterday's date partition

- `*_dod_pct`: Day-over-day % change (e.g., `root_product_dod_pct`, `sale_orders_dod_pct`)

  

**Tables tracked** (26 total):

- Product: root_product, product_sold_raw, product_features_sold/rating/pdp_view/impression_raw/impression_ctr, product_features_v2, sale_orders

- Customer: map_client_customer, customer_purchased/cart_estimated/pdp_view/cate_pdp_view/impression_ctr/cate_impression_ctr

- Client: client_pdp_view/cate_pdp_view/impression_ctr/cate_impression_ctr

- Cate: cate_features_v2, cate_pair

- Predictions: interaction/title/combined

  

**Alert thresholds**:

- Any `*_dod_pct` outside [-0.15, +0.15] (±15%): potential anomaly

- `product_sold_raw_dod_pct` < -30%: significant sales drop

- `sale_orders_dod_pct` < -20%: session data issue

- Predictions `*_dod_pct` < -20%: model quality regression

  

---

  

## Implementation Details

  

### Config

- **Schedule**: `0 8 * * *` (8 AM daily, adjustable in `params/general.yml`)

- **Timeouts**: 600 minutes (10 hours, adjustable)

- **Max active runs**: 1 (runs one at a time)

- **Concurrency**: 3 parallel tasks

  

### Dependencies

- **Wait sensors**: Polls BigQuery for `root_product_YYYYMMDD` and `product_similar_prediction_v2_YYYYMMDD` existence

- Ensures aggregation_daily_v2 and train DAGs completed first

- Adjustable timeout if needed

  

### Timezone

- All dates use `macros.localtz` = Asia/Ho_Chi_Minh timezone

- Jinja templating: `{{macros.localtz.ds(ti)}}` → `YYYY-MM-DD`, `{{macros.localtz.ds_nodash(ti)}}` → `YYYYMMDD`

  

### Error Handling

- Pre-existing slack alert on failure (`alert_reco` connection)

- Retries: 3 attempts with 5-minute backoff

  

---

  

## How to Use

  

### 1. Query monitoring data in Looker Studio / BI tool

  

```sql

-- Daily snapshot of input data health

SELECT run_date, root_product_count, product_features_v2_count, sale_orders_count

FROM `tiki-dwh.recommendation.monitoring_input_data_health_*`

ORDER BY run_date DESC

LIMIT 30;

  

-- Day-over-day anomalies (>20% change)

SELECT run_date,

ABS(root_product_dod_pct) as root_change_pct,

ABS(sale_orders_dod_pct) as session_change_pct,

ABS(combined_pred_dod_pct) as pred_change_pct

FROM `tiki-dwh.recommendation.monitoring_day_over_day_*`

WHERE ABS(root_product_dod_pct) > 0.20

OR ABS(sale_orders_dod_pct) > 0.20

ORDER BY run_date DESC;

  

-- User feature diversity

SELECT run_date,

avg_cate_per_customer,

avg_unique_root_per_customer,

overall_customer_ctr_d30,

client_to_customer_ratio

FROM `tiki-dwh.recommendation.monitoring_user_feature_quality_*`

ORDER BY run_date DESC;

```

  

### 2. Set up alerts

  

**BigQuery scheduled queries** on the monitoring tables:

- Flag `root_product_dod_pct < -0.15` or `product_coverage_ratio < 0.7`

- Post to Slack via webhook

  

**Airflow alert** (pre-configured):

- DAG failure → slack to `alert_reco` connection

  

### 3. Dashboard examples

  

**Input Data Health**

- Time series: root_product_count, product_features_v2_count, sale_orders_count

- Check: Are row counts stable day-over-day?

  

**Prediction Quality**

- Time series: product_coverage_ratio, interaction/title/combined score distributions

- Check: Are models producing output? Scores in expected range?

  

**User Behavior**

- Time series: avg_customer_view_root_d30, overall_customer_ctr_d30, avg_unique_root_per_customer

- Check: User engagement trends, diversity of interactions

  

**Day-over-Day Anomalies**

- Heatmap or threshold alerts: highlight rows where `|*_dod_pct| > 0.20`

- Check: Sudden drops/spikes in any table

  

---

  

## Notes

  

1. **Prev_ds_nodash macro**: Uses Airflow's `macros.localtz.prev_ds_nodash(ti)` to get yesterday's date. Verify your Airflow version supports this (generally available in Airflow 2.x+).

  

2. **Test run**: First run will fail on day-over-day if yesterday's partitions don't exist. This is expected — all future runs will work once 2 days of data exist.

  

3. **Schedule tuning**: Current schedule is `8 AM`. Adjust in `params/general.yml` based on when upstream DAGs complete:

- aggregation_daily_v2: `15 1,5,7 * * *` (1:15 AM, 5:15 AM, 7:15 AM)

- interaction/title training: ~4:30 AM - 1:00 AM (check `general.yml` in train module)

  

4. **Cost optimization**: All queries use BigQuery's cost estimation. For high-volume monitoring, consider:

- Running daily summary instead of per-table snapshots

- Sampling large tables (pdp_view_raw, product_impression_raw)

- Moving day-over-day logic to Dataflow if needed

  

---

  

## Files Created

  

```

Olympus/recommendation/monitoring/

├── config.py # Config loader

├── params/general.yml # Schedule, BQ dataset, pool

└── pipeline/

├── monitor_input_data.py # Input data health task

├── monitor_prediction.py # Prediction health task

├── monitor_user_features.py # User feature quality task

└── monitor_dod.py # Day-over-day comparison task

  

Olympus/recommendation/monitoring/sql/monitoring/

├── input_data_health.sql # All aggregation table row counts

├── prediction_health.sql # Prediction score distributions + coverage

├── user_feature_quality.sql # Customer/client behavioral signals

└── day_over_day_comparison.sql # Today vs yesterday % changes

  

dags/

└── reco_monitoring.py # DAG entry point

```


