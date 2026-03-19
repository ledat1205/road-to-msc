


This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

  

## Repository Overview

  

SmartFlow is Tiki's ML pipeline platform for recommendation, personalization, and product scoring. It runs on **Apache Airflow** with **Kubernetes** executors, **BigQuery** for data warehousing, and **KVDal** for serving predictions.

  

## Repository Layout

  

Two copies of the smartflow repo exist side-by-side:

- **bitbucket/smartflow/** — Production codebase with core recommendation, flash_deal, cate_listing, trending, rfm, repeatable_product, view_complementary modules

- **github/smartflow/** — Extended version with ~70+ additional experimental modules (deep_repr, gcn, random_walk, personalize, etc.) and a separate `Everest/` project

  

Key directories:

- `Olympus/` — All ML pipeline modules live here

- `Olympus/airflow/` — Shared K8s pod configs, Docker image definitions

- `Olympus/recommendation/` — The main recommendation system (aggregation, training, inference)

- `dags/` — Airflow DAG definition files (e.g., `reco_data_agg_daily_v2.py`, `reco_train_interaction.py`)

- `dags_conf/` — YAML config for specific DAG deployments

  

## Olympus Module Pattern

  

Every module follows this standard structure:

```

module_name/

├── pipeline/ # Airflow task builders (train.py, predict.py, agg_*.py)

├── params/ # YAML configs (general.yml, train.yml, prediction.yml, cate.yml, feature.yml, meta.yml)

│ └── features/ # Feature selection JSON

├── sql/ # BigQuery SQL templates (Jinja-templated)

├── train/ # Training scripts (run inside K8s pods)

├── predict/ # Prediction scripts

├── evaluate/ # Evaluation scripts

├── config.py # Loads params/*.yml via yaml.SafeLoader

└── utils/ # Shared utilities (airflow_utils.py, utils.py)

```

  

### Pipeline builder pattern

  

Files in `pipeline/` export functions that construct Airflow task chains:

```python

def agg_product_data(dag):

# Creates BigQueryExecuteQueryOperator tasks chained with >>

# Returns final DummyOperator

```

  

Helper function `sql_task(dag, task_id, sql_path, table_destination, key_id)` is the core abstraction for BigQuery tasks.

  

Training/prediction tasks use `KubernetesPodOperator` running Docker images from `asia-southeast1-docker.pkg.dev/tikivn/tikivn/smartflow:release`.

  

### DAG construction pattern

  

```python

with DAG(dag_id="...", default_args=cfg.default_args, ...) as dag:

start = DummyOperator(task_id="start")

with TaskGroup("product_features") as product_section:

product_features = agg_product_data(dag)

start >> product_section >> ...

```

  

## Recommendation System DAG Chain & Data Lineage

  

The recommendation pipeline flows through these DAGs in order. Models used: **LightFM** (matrix factorization for interaction/title similarity), **LightGBM** (classification for user propensity per category), **pysparnn** (approximate nearest neighbors for title similarity).

  

---

  

### 1. DAG: `recommendation_data_aggregation_daily_v2`

  

**Input (9 source tables):**

  

| # | Table | Description |

|---|-------|-------------|

| 1 | `tiki-dwh.dwh.dim_product_full` | Product attributes (from DAG `dwh_etl_dim_product_full`, aggregated from many BQ tables) |

| 2 | `tiki-dwh.trackity_v2.core_events` | User action logs on the platform |

| 3 | `tiki-dwh.trackity_tiki.product_impressions` | Product impression logs |

| 4 | `tiki-dwh.cdp.datasources` | User/session product view logs |

| 5 | `tiki-dwh.olap.trackity_tiki_click` | User click logs |

| 6 | `tiki-dwh.ecom.review` | Product rating/review logs |

| 7 | `tiki-dwh.nmv.nmv` | Net merchandise value (sales) |

| 8 | `tiki-analytics-dwh.personas.product_tier` | Product price segment (manually maintained by commercials team) |

| 9 | `tiki-dwh.trackity.tiki_events_sessions_cross_devices` | Cross-device user action/session logs |

  

**Output (24 intermediate tables in `tiki-dwh.recommendation.*`):**

  

| # | Table | Description |

|---|-------|-------------|

| 1 | `root_product_YYYYMMDD` | Basic product info (name, code, category, seller_id) + root_product_id mapping |

| 2 | `map_client_customer_YYYYMMDD` | client_id → customer_id mapping from 365-day logs |

| 3 | `event_staging` (in `tiki-analytics-dwh.personas`) | Staged view_pdp, add_to_cart, complete_purchase, true_impression events for favorite cate pipeline |

| 4 | `sale_orders_YYYYMMDD` | Products viewed in same session (product_id = root_product_id) |

| 5 | `pdp_view_raw_YYYYMMDD` | Raw PDP view logs (90 days) from trackity_tiki_click + core_events, joined with root_product and map_client_customer |

| 6 | `product_features_pdp_view_YYYYMMDD` | PDP view aggregates by root/product_id (1, 7, 30 days) |

| 7 | `product_features_rating_YYYYMMDD` | Rating aggregates (avg rating, count, comments) by root/product_id |

| 8 | `product_impression_raw_YYYYMMDD` | Raw impression+click data (30 days) joined with root_product and map_client_customer |

| 9 | `product_impression_ctr_YYYYMMDD` | Product CTR from impressions/clicks (1, 30 days) |

| 10 | `product_sold_raw_YYYYMMDD` | Order data from nmv (365 days) with shipping day calc |

| 11 | `product_features_sold_YYYYMMDD` | Sales aggregates (quantity, customers, orders, price, shipping time) by root/product_id (30, 90 days) |

| 12 | `product_features_v2_YYYYMMDD` | **Master product features table** — joins root_product, pdp_view, rating, sold, impression_ctr, and product_tier |

| 13 | `cate_features_v2_YYYYMMDD` | Category features (cate 1-6) aggregated from product_features_v2 |

| 14 | `cate_pair_YYYYMMDD` | Primary cate co-occurrence from sessions in pdp_view_raw (used for PDP infinity widget) |

| 15 | `customer_cart_estimated_YYYYMMDD` | Estimated cart contents from add/remove/purchase logs (365 days) — coded but not yet active |

| 16 | `customer_purchased_YYYYMMDD` | Products purchased per user (365 days) — used for filtering |

| 17 | `customer_pdp_view_YYYYMMDD` | User PDP views by customer_id × product_id (7, 30, 90 days) |

| 18 | `customer_cate_pdp_view_YYYYMMDD` | User PDP views by primary cate (7, 30, 90 days) — not yet used |

| 19 | `client_pdp_view_YYYYMMDD` | PDP views by client_id for non-logged-in users (7, 30, 90 days) — not yet used |

| 20 | `client_cate_pdp_view_YYYYMMDD` | PDP views by primary cate for client_id (7, 30, 90 days) — not yet used |

| 21 | `customer_impression_ctr_YYYYMMDD` | User impression/click aggregates by customer_id (30 days) — not yet used |

| 22 | `customer_cate_impression_ctr_YYYYMMDD` | User impression/click by primary cate (30 days) — not yet used |

| 23 | `client_impression_ctr_YYYYMMDD` | Impression/click by client_id (30 days) — not yet used |

| 24 | `client_cate_impression_ctr_YYYYMMDD` | Impression/click by primary cate for client_id (30 days) — not yet used |

  

---

  

### 2. DAG: `recommendation_train_title_v2`

  

**Model:** LightFM / pysparnn — product-product similarity by product title (Vietnamese tokenized via pyvi)

  

| Direction | Table | Description |

|-----------|-------|-------------|

| Input | `tiki-dwh.recommendation.product_features_v2_YYYYMMDD` | Product features with title text |

| Output | `tiki-dwh.recommendation.prediction_title_similar_YYYYMMDD` | Product-product similarity scores by title (product_id level) |

  

### 3. DAG: `recommendation_train_interaction_v2`

  

**Model:** LightFM — matrix factorization on session co-view data

  

| Direction | Table | Description |

|-----------|-------|-------------|

| Input | `tiki-dwh.recommendation.root_product_YYYYMMDD` | Product mapping |

| Input | `tiki-dwh.recommendation.sale_orders_YYYYMMDD` | Session co-view data |

| Input | `tiki-dwh.recommendation.product_features_v2_YYYYMMDD` | Product features for filtering |

| Input | `tiki-dwh.recommendation.prediction_title_similar_YYYYMMDD` | Title similarity (for combining) |

| Output | `tiki-dwh.recommendation.input_data_train_interaction_YYYYMMDD` | Training input: sessions with 1-6 cates, 2-15 products |

| Output | `tiki-dwh.recommendation.prediction_interaction_similar_YYYYMMDD` | Product-product similarity by session co-views (root_product_id level) |

| Output | `tiki-dwh.recommendation.product_similar_prediction_v2_YYYYMMDD` | **Combined similarity** (score_interaction, score_title, max_score_model, score_cate) for reco widgets |

  

### 4. DAG: `recommendation_train_user_favorite_cate_v2`

  

**Model:** LightGBM — binary classification for user propensity per primary category

  

| Direction | Table | Description |

|-----------|-------|-------------|

| Input | `tiki-dwh.dwh.dim_product_full` | Product attributes |

| Input | `tiki-dwh.nmv.nmv` | Sales data |

| Input | `tiki-analytics-dwh.personas.event_staging` | Staged user events from aggregation DAG |

| Output | `tiki-analytics-dwh.personas.purchase_feature` | Purchase count features by primary_cate_id (30, 90 days) |

| Output | `tiki-analytics-dwh.personas.view_feature` | CTR features by primary_cate_id (30, 90 days) |

| Output | `tiki-analytics-dwh.personas.purchase_label` | Label: user purchased in cate during month N-1 |

| Output | `tiki-analytics-dwh.personas.view_label` | Label: user had ATC/view/impression in cate during month N-1 |

| Output | `tiki-analytics-dwh.personas.data_train` | Combined features + labels for model training |

| Output | `tiki-analytics-dwh.personas.propensity_cate_YYYYMMDD` | Per-user propensity scores by primary_cate_id |

| Output | `tiki-analytics-dwh.personas.combine_propensity` | Weighted propensity (6 months, time-decay weights) → determines favorite cates for Home personalization |

  

---

  

### 5. DAG: `recommendation_infer_data_agg`

  

| Direction | Table | Description |

|-----------|-------|-------------|

| Input | `tiki-dwh.dwh.dim_product_full` | Product attributes |

| Input | `tiki-dwh.recommendation.product_features_v2_YYYYMMDD` | Product features |

| Output | `tiki-dwh.recommendation.product_weight_YYYYMMDD` | Boosting weights (tikinow, next day, brand, etc.) |

| Output | `tiki-dwh.recommendation.product_features_v2_with_weights_YYYYMMDD` | Product features + dim_product_full + weights joined |

| Output | `tiki-dwh.recommendation.top_view_d7_product_weight_YYYYMMDD` | Products with views in 7 days + view share per cate (for Home widgets) |

  

### 6. DAG: `recommendation_infer_area_home_p1_v2 / p2_v2`

  

| Direction | Table | Source DAG | Description |

|-----------|-------|------------|-------------|

| Input | `tiki-dwh.recommendation.top_view_d7_product_weight_YYYYMMDD` | infer_data_agg | Products with 7-day views + cate weights |

| Input | `tiki-dwh.recommendation.customer_pdp_view_YYYYMMDD` | aggregation_daily_v2 | User product views (latest 7 days of tables) |

| Input | `tiki-dwh.recommendation.client_pdp_view_YYYYMMDD` | aggregation_daily_v2 | Client product views — coded but not yet active |

| Input | `tiki-analytics-dwh.personas.combine_propensity` | train_user_favorite_cate_v2 | User favorite cate propensity scores |

| Input | `tiki-dwh.recommendation.product_features_v2_with_weights_YYYYMMDD` | infer_data_agg | Product features with weights |

| Input | `tiki-dwh.recommendation.product_similar_prediction_v2_YYYYMMDD` | train_interaction_v2 | Combined product similarity (latest in 3 days) |

| Input | `tiki-dwh.recommendation.customer_cart_estimated_YYYYMMDD` | aggregation_daily_v2 | Estimated cart — coded but not yet active |

| Input | `tiki-dwh.recommendation.customer_purchased_YYYYMMDD` | aggregation_daily_v2 | Purchased products (latest in 3 days) |

| Input | `tiki-dwh.recommendation.map_client_customer_YYYYMMDD` | aggregation_daily_v2 | Client-customer mapping (latest in 3 days) |

  

### 7. DAG: `recommendation_infer_area_pdp_v2`

  

| Direction | Table | Source DAG | Description |

|-----------|-------|------------|-------------|

| Input | `tiki-dwh.recommendation.product_features_v2_with_weights_YYYYMMDD` | infer_data_agg | Product features with weights |

| Input | `tiki-dwh.smarter.vw_related_product_v2` | Manual (view from `tiki-analytics-dwh.personas.related_product_v2`) | Complementary cate pairs — manually maintained, may be outdated |

| Input | `tiki-dwh.recommendation.product_similar_prediction_v2_YYYYMMDD` | train_interaction_v2 | Combined product similarity |

| Input | `tiki-dwh.recommendation.cate_pair_YYYYMMDD` | aggregation_daily_v2 | Primary cate co-occurrence (for infinity widget) |

  

### 8. DAG: `recommendation_monitoring_v2` (new)

  

Runs daily after inference completes. Validates all 5 pillars: input freshness, intermediate table quality, feature drift (PSI + z-score), model output quality, and weekly SHAP explainability. Writes to `tiki-dwh.recommendation_monitoring.*` and sends consolidated Slack alerts.

  

## Improvement Plan

  

Full training + serving improvement roadmap: **[RECOMMENDATION_IMPROVEMENT_PLAN.md](./RECOMMENDATION_IMPROVEMENT_PLAN.md)**

  

Summary of 4 phases:

1. **Quick wins** (weeks 1-3): Optuna hyperparameter tuning, temporal decay, propensity feature enrichment — training only, no serving changes

2. **Semantic title + learned ranker** (weeks 4-7): Sentence-BERT replaces TF-IDF; LambdaRank ONNX model replaces rule-based reranking in smarter-model-serving

3. **Two-tower + real-time user embeddings** (weeks 8-15): Neural two-tower replaces LightFM; FAISS + ONNX user tower in serving for real-time personalization; Kafka → Redis for user action ingestion

4. **Sequential recommendation** (weeks 16+): SASRec transformer for session-aware Home page recommendations

  

## Serving Layer (smarter-model-serving)

  

Go gRPC service at `/Users/lap02399/smarter/services/models_serving/`. Pure lookup + light reranking.

  

**Home widget** (user_id → recommendations):

- Keys: `[customer_id, trackity_id, "_"]` (priority order) → Cassandra multi-get → FlatBuffer decode → dedup → CTR-based rerank from Redis

- Cold start: falls back to popular ("_") list

  

**PDP widget** (product_id → similar products):

- Keys: `["{customer_id}_{mpid}", "{trackity_id}_{mpid}", "_{mpid}"]` → same Cassandra flow

  

**Reranking** (`pkg/coremetrics/metrics_ranking.go`): Binary good/bad split by impression count (≥2 impressions + 0 clicks = bad), CTR sort. Controlled via `BlockFlag`.

  

## Tech Stack

  

- **Orchestration:** Apache Airflow 2.x with TaskGroups

- **Compute:** Kubernetes pods (namespace: `airlock`)

- **Data warehouse:** BigQuery (datasets: `tiki-dwh.recommendation`, `tiki-analytics-dwh.personas`)

- **ML libraries:** LightGBM, scikit-learn, LightFM, pysparnn (in Docker image based on miniconda3)

- **NLP:** pyvi (Vietnamese tokenization)

- **Serving:** KVDal (key-value store) via BigqueryToKVDalOperator

- **Storage:** GCS for model artifacts

- **Alerts:** Slack via alertlock.dynamic.alert

  

## Configuration

  

- `params/general.yml` — DAG-level config: schedule_interval, retries, BigQuery dataset, GCP connection ID, pool

- K8s resource configs defined in `Olympus/airflow/k8s.py` (DEFAULT_POD_CONFIG, COMMON_K8S_HIGHMEM for training)

- GCP connection: `gcp_tiki_datateam_ds`

- BigQuery tables use date-suffixed naming: `table_name_YYYYMMDD`

  

## Docker Build

  

```bash

# Dockerfile at bitbucket/smartflow/Dockerfile (base: continuumio/miniconda3)

# Requirements: bitbucket/smartflow/requirements.txt

# Timezone: Asia/Ho_Chi_Minh, workdir: /home/Olympus

```