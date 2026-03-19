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


Here is a concise **dbt concepts cheat sheet** (focused on **dbt Core**, current as of 2025–2026 best practices). It covers the most important building blocks and when to choose each one.

### Core Building Blocks

| Concept          | What it is                                                                 | Key Jinja / syntax                          | When to use it                                                                 | Common folder / prefix     |
|------------------|----------------------------------------------------------------------------|---------------------------------------------|--------------------------------------------------------------------------------|----------------------------|
| **Model**        | SQL SELECT (or Python) transformation file — the heart of dbt                | `{{ config(...) }}` at top                  | Every transformation you want to version, test & document                       | models/                    |
| **Source**       | Raw / ingested tables you read from (not created by dbt)                    | `{{ source('source_name', 'table_name') }}` | Reference external tables safely + add freshness / loaded_at tests              | models/staging + sources.yml |
| **Seed**         | Static CSV files loaded as tables                                           | `{{ ref('seed_name') }}`                    | Lookup / mapping tables, small manual data, constants                           | seeds/                     |
| **Snapshot**     | SCD Type 2 slowly changing dimension capture (tracks history)               | `{% snapshot ... %}`                        | Dimension tables that change slowly and you need history (users, products…)     | snapshots/                 |
| **Exposure**     | Downstream consumer (dashboard, notebook, app)                              | exposures.yml                               | Document lineage to BI tools / ML models                                        | exposures.yml              |
| **Metric**       | (dbt Semantic Layer) – reusable business metric definition                   | metrics.yml                                 | Define "revenue", "active_users" once and reuse in many places                  | metrics/ (or metrics.yml)  |

### Materializations – The #1 decision in dbt modeling

| Materialization     | How data is stored / refreshed                                      | Build speed | Query speed | Storage cost | Best use cases                                                                                     | Default? | When **not** to use                          |
|---------------------|---------------------------------------------------------------------|-------------|-------------|--------------|----------------------------------------------------------------------------------------------------|----------|----------------------------------------------|
| **view**            | No table — just a saved SELECT query (recomputed on every read)     | Very fast   | Slower      | Almost none  | Small/medium data, staging layers, dev iteration, always-fresh data                                | Yes ✓    | Large tables, frequent BI queries            |
| **table**           | Full rebuild → CREATE OR REPLACE TABLE every run                    | Slow        | Fast        | High         | Medium-sized models, complex joins, models heavily queried by BI / downstream                      | —        | Very large append-only data (use incremental)|
| **incremental**     | Only process new/changed rows (insert/update/delete/merge)          | Fast after first run | Fast        | High         | Large event/log/fact tables, append-heavy data, expensive full refreshes, daily+ volume            | —        | Small tables, changing keys without history  |
| **ephemeral**       | No object created — SQL inlined as CTE/subquery into downstream models | Instant (no build) | Depends on parent | None         | Reusable logic, light cleaning, intermediate steps you don’t want to expose/query directly         | —        | Anything you want to query independently     |
| **materialized_view** (newer warehouses: BigQuery, Snowflake…) | Hybrid: precomputed + auto-refreshes (warehouse managed)           | Medium      | Very fast   | High         | Performance-critical dashboards on medium-large data without manual incremental logic              | —        | Warehouses without native support            |

**Quick materialization decision tree** (most common 2025–2026 advice):

1. Start with **view** (default) — almost everything in staging & intermediate.
2. Switch to **table** when BI / downstream tools complain about speed.
3. Switch to **incremental** when full table rebuild > 5–15 min **and** data is append-mostly or has reliable unique_key + timestamp.
4. Use **ephemeral** for shared utility logic (e.g. clean_url macro logic, currency conversion) that multiple models need but nobody queries alone.
5. Consider **materialized_view** only if your warehouse supports it well and you want warehouse-managed refresh.

### Quick Reference – Common Patterns & When to Use

- **Staging layer** → `materialized='view'` or `'ephemeral'` (clean + rename columns, add metadata)
- **Intermediate / business logic** → `view` or `table` (joins, filters, window functions)
- **Marts / fact / wide tables** → `table` or `incremental` (optimized for BI)
- **Large append-only events** → `incremental` + `is_incremental()` macro + `merge` strategy
- **Type 2 dimensions** → `snapshot` (not a normal model)
- **Static lookups** → `seed`
- **Reusable snippets** → macros/ or **ephemeral** models
- **Tests + docs** → `schema.yml` next to models (don't forget `description`, `tests: unique, not_null, relationships`)

### Most Useful Jinja / Macros Snippets

```sql
-- Top of file
{{ config(
    materialized = 'incremental',
    unique_key = 'order_id',
    incremental_strategy = 'merge',   -- or 'insert_overwrite', 'delete+insert'
) }}

-- Incremental logic
{% if is_incremental() %}
  WHERE event_ts > (SELECT MAX(event_ts) FROM {{ this }})
{% endif %}

-- Safe ref / source
{{ ref('stg_orders') }}
{{ source('raw', 'payments') }}
```

This cheat sheet covers ~80% of day-to-day decisions in modern dbt projects.


 ---

 #Plan: **Migrate** **Airflow** **BigQuery** **DAGs** **to** **dbt** **Models**

  

  **Context**

  

  - **330** **DAGs**, **2,290** **tasks** in serialized-dags.csv

  - Focus operators: BigQueryExecuteQueryOperator (1,103 tasks) +

  SQLToBigQueryOperator (504 tasks) = **1,607** **tasks** to migrate

  - Projects: tiki-dwh, tnsl-dwh across ~20+ datasets

  

  ---

  **Phase** **1** **—** **Parse** **&** **Inventory** **(Foundation)**

  

  **Goal:** Extract all SQL from the CSV into a structured inventory before touching

   dbt.

  

  Steps:

  1. Write a Python script to parse serialized-dags.csv → extract per-task:

  dag_id, task_id, operator_type, sql, destination_table, write_disposition,

  depends_on (upstream tasks), schedule, owner

  2. Identify **write** **dispositions**: WRITE_TRUNCATE → full refresh candidates,

  WRITE_APPEND → incremental candidates

  3. Map **table-level** **lineage**: which task writes to which project.dataset.table,

  which SQL reads from which tables → build a dependency graph

  4. Flag complexity: tasks with {{ macros... }} template vars, multi-statement

  SQL, partitioned tables, cross-project references

  

  **Output:** A spreadsheet/JSON of all 1,607 tasks ranked by migration complexity

  (simple → complex).

  

  ---

  **Phase** **2** **—** **dbt** **Project** **Scaffold**

  

  **Goal:** Set up the dbt project structure aligned to your existing dataset

  organization.

  

  dbt/

  ├── dbt_project.yml

  ├── profiles.yml          # BigQuery connection (tiki-dwh, tnsl-dwh)

  ├── models/

  │   ├── staging/          # Raw source → typed, renamed (maps to staging_v2

  dataset)

  │   ├── intermediate/     # Complex joins, not exposed externally

  │   ├── marts/

  │   │   ├── ecom/         # tiki-dwh.ecom

  │   │   ├── nmv/          # tiki-dwh.nmv

  │   │   ├── commercial/   # tiki-dwh.commercial

  │   │   ├── fna/          # tiki-dwh.fna

  │   │   ├── marketplace/  # tiki-dwh.marketplace

  │   │   └── ...

  ├── macros/

  │   └── localtz.sql       # Replicate Airflow's macros.localtz.ds()

  └── tests/

  

  Dataset → dbt folder mapping follows existing tiki-dwh.* structure for

  zero-disruption rollout.

  

  ---

  **Phase** **3** **—** **Macro** **Translation**

  

  **Goal:** Replace Airflow-specific Jinja with dbt equivalents.

  

  ┌──────────────────────────┬──────────────────────────────────────────────┐

  │         Airflow          │                dbt equivalent                │

  ├──────────────────────────┼──────────────────────────────────────────────┤

  │ {{ macros.localtz.ds(ti) │ {{ var('ds') }} or {{ run_started_at |       │

  │  }}                      │ convert_timezone(...) }}                     │

  ├──────────────────────────┼──────────────────────────────────────────────┤

  │ {{ ds }}                 │ {{ var('execution_date') }}                  │

  ├──────────────────────────┼──────────────────────────────────────────────┤

  │ {{ next_ds }}            │ custom macro                                 │

  ├──────────────────────────┼──────────────────────────────────────────────┤

  │ Dynamic partition        │ dbt is_incremental() block                   │

  │ filters                  │                                              │

  └──────────────────────────┴──────────────────────────────────────────────┘

  

  Create a macros/airflow_compat.sql file to centralize these translations.

  

  ---

  **Phase** **4** **—** **Model** **Conversion** **(Prioritized** **Batches)**

  

  Convert in order of complexity:

  

  **Batch** **A** **—** **Simple** **truncate/replace** **(no** **dependencies)**

  - WRITE_TRUNCATE tasks with self-contained SQL → materialized='table'

  - Easiest wins, validates the pipeline

  

  **Batch** **B** **—** **Chain** **dependencies** **within** **a** **single** **DAG**

  - Tasks where Task B reads from Task A's output → convert both, use ref()

  instead of hardcoded table names

  - This is where lineage starts appearing in dbt docs

  

  **Batch** **C** **—** **Cross-DAG** **dependencies**

  - Tasks using ExternalTaskSensor → convert to dbt source() declarations or

  dbt's cross-project ref()

  

  **Batch** **D** **—** **Incremental** **candidates**

  - WRITE_APPEND tasks or partitioned table writes → materialized='incremental'

  with unique_key and partition filter

  - Apply is_incremental() block to filter only new rows

  

  **Batch** **E** **—** **Complex** **/** **skip** **for** **now**

  - KubernetesPodOperator, PythonOperator, SparkKubernetesOperator → keep in

  Airflow, use dbt source() to reference their outputs

  

  ---

  **Phase** **5** **—** **Lineage** **&** **Orchestration**

  

  **Goal:** Replace Airflow task dependencies with dbt's native DAG.

  

  - Replace {{ project.dataset.table }} hardcoded refs with {{ ref('model_name')

   }} — dbt infers execution order automatically

  - For tables still owned by Airflow (Phase E), declare them as sources: in

  schema.yml

  - Use dbt's --select and --defer for selective runs

  - Orchestrate dbt runs from Airflow using BashOperator or

  DbtCloudRunJobOperator — one Airflow DAG triggers dbt instead of 100s of SQL

  tasks

  

  ---

  **Phase** **6** **—** **Testing** **&** **Validation**

  

  For each converted model:

  1. **Row** **count** **check**: compare dbt output vs. existing BigQuery table

  2. **Column-level** **diff**: sample 1000 rows, assert no difference

  3. Add dbt native tests: not_null, unique on key columns

  4. Add freshness checks on sources

  

  ---

  **Phase** **7** **—** **Cutover**

  

  1. Run dbt models in **parallel** with existing Airflow DAGs for 1–2 weeks

  2. Validate parity daily

  3. Pause Airflow DAGs one domain at a time (start with ecom, then nmv, etc.)

  4. Keep Airflow as the scheduler, now calling dbt run --select tag:domain

  

  ---

  **Recommended** **Starting** **Point**

  

  Start with the **datateam_etl_aff** or **fna_bi_tiki_monthly_report** DAGs — they have

   a bounded number of tasks, clear domain ownership, and heavy

  BigQueryExecuteQueryOperator usage. Proving the pattern on one full DAG gives

  you a repeatable template for the rest.