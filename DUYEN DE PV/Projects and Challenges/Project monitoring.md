**The three-layer architecture**

```
Layer 1 — Kafka-side:     consumer lag, offset progress, per-partition skew
Layer 2 — BigQuery-side:  table-level SQL checks via DataQualityOperator
Layer 3 — Statistical:    baseline-derived anomaly detection, drift, coherence
```

Each layer catches failures the others miss. Layer 1 catches consumer health before data reaches BigQuery. Layer 2 catches data correctness after it arrives. Layer 3 catches slow degradation that never crosses a hard threshold.

---

**The tables**

**`monitoring.checks_config`** — the central config registry. Every check definition lives here. No check logic is hardcoded in any DAG.

sql

```sql
CREATE TABLE monitoring.checks_config (
  table_name    STRING,      -- fully qualified: dataset.table
  check_type    STRING,      -- FRESHNESS | ROW_COUNT | NULL_RATE | DUPLICATE |
                             -- BUSINESS_RULE | RECONCILIATION | REF_INTEGRITY |
                             -- PARTITION_COMPLETE | LATE_DATA | UPSTREAM_READY |
                             -- TASK_DURATION | SPIKE_DIAGNOSIS
  severity      STRING,      -- WARN | ERROR
  params        JSON,        -- check-specific parameters
  level         STRING,      -- TABLE_SQL | SENSOR | INFRA
  run_context   STRING,      -- AFTER_LOAD | AFTER_TRANSFORM | PRE_SERVE
  enabled       BOOL,
  added_by      STRING,
  added_at      TIMESTAMP
)
```

**`monitoring.pipeline_quality_log`** — every check result from every run. This is both the audit trail and the source for dashboards and trending.

sql

```sql
CREATE TABLE monitoring.pipeline_quality_log (
  checked_at      TIMESTAMP,
  table_name      STRING,
  check_type      STRING,
  status          STRING,      -- OK | WARN | ERROR
  metric_value    FLOAT64,     -- observed value
  baseline_value  FLOAT64,     -- what was expected
  deviation       FLOAT64,     -- z-score, pct_change, or ratio
  detail          STRING,      -- human-readable explanation
  dag_id          STRING,
  task_id         STRING,
  kafka_partition INT64        -- populated for partition-level checks
)
PARTITION BY DATE(checked_at)
```

**`monitoring.task_duration_baseline`** — per-task runtime statistics, recomputed monthly from Airflow metadata.

sql

```sql
CREATE TABLE monitoring.task_duration_baseline (
  dag_id          STRING,
  task_id         STRING,
  baseline_month  DATE,
  p50_sec         FLOAT64,
  p75_sec         FLOAT64,
  p95_sec         FLOAT64,
  p99_sec         FLOAT64,
  mean_sec        FLOAT64,
  stddev_sec      FLOAT64,
  sample_count    INT64
)
```

**`monitoring.null_rate_config`** — per-column null rate thresholds, because critical ID columns (order_id) have a 1.1× tolerance while optional fields (gift_message) have a 10× tolerance.

sql

```sql
CREATE TABLE monitoring.null_rate_config (
  table_name      STRING,
  col_name        STRING,
  baseline_rate   FLOAT64,   -- 30-day rolling average null rate
  threshold_ratio FLOAT64,   -- multiplier before alerting (1.1 to 10)
  updated_at      TIMESTAMP
)
```

**`monitoring.schema_snapshot`** — daily snapshot of INFORMATION_SCHEMA for drift detection.

sql

````sql
CREATE TABLE monitoring.schema_snapshot (
  snapshot_date   DATE,
  table_schema    STRING,
  table_name      STRING,
  column_name     STRING,
  data_type       STRING,
  ordinal_position INT64
)
PARTITION BY snapshot_date
```

**`monitoring.rfm_segments`** and **`monitoring.task_duration_baseline`** feed into `pipeline_quality_log` indirectly — RFM feeds the CRM action queue, task duration feeds the SLA miss callback.

---

**The schedules**

Every schedule is separated by its required freshness and computational cost.
```
Every 15 minutes:
  - TABLE_SQL checks on CRITICAL tables (financial reconciliation,
    ad spend, session events)
  - Watchdog DAG: query INFORMATION_SCHEMA.JOBS_BY_PROJECT,
    cancel runaway queries (> 45 min runtime or > 5 TB scanned)

Every hour:
  - TABLE_SQL checks on STANDARD tables
  - Kafka lag summary: aggregate per-partition lag into pipeline_quality_log

Nightly at 01:00:
  - Schema drift detection: snapshot INFORMATION_SCHEMA,
    diff against yesterday's snapshot, alert on DROPPED columns,
    query lineage graph for affected DAGs
  - Null rate baseline refresh: recompute 30-day rolling null rate
    per column, update null_rate_config
  - RFM scoring: recompute all seller segments, write to rfm_segments,
    compute migration matrix vs last week

Monthly on the 1st:
  - Task duration baseline: read task_instance from Airflow PostgreSQL,
    compute p50/p75/p95/p99 per (dag_id, task_id) over last 30 days,
    write to task_duration_baseline
  - Trend model refresh: refit linear regression baselines for
    trend-adjusted Z-score checks, update stored slope and intercept
    per table per metric
```

---

**How a check actually executes — the full flow**

Taking a `ROW_COUNT` check on `mkt_mart.extract_cost_campaign_facebook` as the concrete example:
```
1. Airflow DAG runs the main SQL task (BigQueryExecuteQueryOperator)
   → writes to mkt_mart.extract_cost_campaign_facebook

2. DataQualityOperator runs next in the DAG
   → reads checks_config WHERE table_name = 'mkt_mart...' AND level = 'TABLE_SQL'
   → finds ROW_COUNT check with params: {lookback_days: 14, warn_zscore: 2.0, error_zscore: 3.0}

3. Operator executes the check SQL:
   → computes today's row count
   → computes 14-day same-DOW rolling mean and stddev from pipeline_quality_log
     (or directly from the table if first run)
   → z_score = (today_count - mean) / stddev

4. If promotional day (from calendar.promotional_days):
   → uses promotional baseline instead of standard 14-day window

5. If z_score > 3.0:
   → status = ERROR
   → auto-triggers SPIKE_DIAGNOSIS check:
     decomposes delta across [_kafka_partition, ad_platform, seller_tier]
     returns top-k contributing dimensions with absolute contribution and pct_of_delta

6. Results written to pipeline_quality_log:
   {checked_at, table_name, check_type='ROW_COUNT', status='ERROR',
    metric_value=today_count, baseline_value=mean, deviation=z_score,
    detail='142K rows, z=3.4 (baseline avg 51K)'}

7. Slack alert fires to #data-incidents with:
   - Table name, check type, severity
   - Observed vs baseline values
   - SPIKE_DIAGNOSIS output:
     "Top driver: _kafka_partition — partition 7 is +890% vs baseline"

8. AirflowException raised → downstream mart tasks blocked
```

---

**The three execution primitives and when each is used**
```
DataQualityOperator  →  TABLE_SQL checks
  Runs inline after the main task in the DAG.
  One shot: executes SQL, gets scalar result, passes or fails.
  Raises AirflowException on ERROR to block downstream.

UpstreamReadySensor  →  SENSOR checks (UPSTREAM_READY only)
  Runs before the main task, not after.
  Polls every 5 minutes (mode=reschedule — releases worker slot between polls).
  Returns True when upstream is fresh, unblocking the main task.
  Times out after 1 hour and raises exception if upstream never refreshes.
  mode=reschedule is critical: 300+ DAGs polling in poke mode
  would exhaust the worker pool.

SLA miss callback     →  INFRA checks (TASK_DURATION)
  Fires externally after a task exceeds its declared SLA timedelta.
  Not inline — fires from the Airflow scheduler, not the worker.
  Loads the task's monthly baseline from task_duration_baseline.
  Computes z-score of elapsed time vs baseline p95/p99.
  Writes result to pipeline_quality_log.
  Routes to #data-quality-alerts (WARN) or #data-incidents (ERROR).

Separate monitoring DAGs  →  INFRA checks (SCHEMA_DRIFT, COST_ANOMALY)
  Run on their own schedule, independent of any pipeline DAG.
  Schema drift: nightly, covers all 5,400 tables simultaneously,
  queries lineage graph for downstream impact of each dropped column.
  Cost anomaly watchdog: every 15 minutes, cancels runaway BQ queries.
```

---

**The baseline types and where each lives**
```
Baseline type              Where stored              Recomputed
─────────────────────────────────────────────────────────────────────
14-day rolling row count   pipeline_quality_log       On each check run
  same-hour same-DOW       (queried dynamically)

30-day null rate           null_rate_config            Nightly

30-day distribution        Computed inline in check    On each check run
  (KL divergence, JSD)     SQL (no stored state)

p99 freshness latency      Computed inline             On each check run
  per Kafka topic

Monthly task duration      task_duration_baseline      1st of each month
  p50/p75/p95/p99

Trend-adjusted baseline    monitoring.trend_params     Monthly
  (slope + intercept)      (dag_id, metric, slope,
                            intercept, resid_stddev)

Promotional calendar       calendar.promotional_days   Manually maintained
  (11.11, Tet, flash sale)  + pre_sale_window_days

RFM segment baseline       rfm_segments (historical    Weekly
  (migration tracking)      snapshots by scored_at)
```

---

**The alert routing**
```
Status   Channel                  Action
──────────────────────────────────────────────────────────────────
WARN     #data-quality-alerts     Slack only. No human action
                                  required immediately. Engineers
                                  monitor the trend.

ERROR    #data-incidents          Slack + PagerDuty on-call.
         + PagerDuty              Downstream tasks blocked by
                                  AirflowException. Incident
                                  ticket opened automatically.

COST     #data-costs              Slack only. BigQuery job
ANOMALY                           cancelled by watchdog. Cost
                                  anomaly written to quality log.

SCHEMA   #data-schema-changes     Slack with: table name, change
DRIFT    + affected team channels type (ADDED/DROPPED/TYPE_CHANGED),
                                  list of downstream DAGs from
                                  lineage graph.
````

---

**How to frame this in the interview**

> "The monitoring system has three layers: Kafka-side lag metrics for consumer health, BigQuery-side table checks for data correctness, and statistical baseline-derived checks for slow degradation. All check definitions live in a single config table — adding a check to a new pipeline is one INSERT and one task line in the DAG. Check results all write to a single quality log table which feeds the SLA dashboard.
> 
> The execution has three primitives matched to what each check actually needs. SQL checks that run once and pass or fail go to a DataQualityOperator inline in the DAG. Checks that need to wait and retry — upstream readiness — go to a sensor with mode=reschedule so they don't hold worker slots. Infrastructure checks that need to fire asynchronously or cover all pipelines simultaneously — task duration, schema drift, cost anomalies — live in separate monitoring DAGs or DAG-level callbacks.
> 
> Baselines are not hardcoded numbers. Row count thresholds are derived from 14-day rolling same-hour same-day-of-week distributions. Task duration thresholds are derived from monthly p95/p99 per task from the Airflow task_instance table. Null rate thresholds are per-column ratios against a 30-day baseline, not absolute values. The only baselines that are manually maintained are the promotional calendar, because 11.11 and Tet produce volume patterns that no rolling window can correctly model — they are structural outliers that need to be flagged as expected before the statistical checks see them."