**“Designed data pipelines and feature schemas supporting real-time model evaluation; defined canonical metric definitions (NDCG, CTR) used consistently across search and recommendation model iterations.”**

This was one of the highest-leverage pieces of the entire recommendation platform re-architecture I owned. Before my work, model evaluation was a daily nightmare — different teams calculated CTR and NDCG differently, features were stale, and new model versions could only be tested the next day after the batch Airflow job finished.

### 1. The Problem I Inherited

- Hundreds of scattered BigQuery SQL transformations inside 300+ Airflow DAGs.
- CTR was defined 12+ different ways across search, recsys, and analytics teams.
- NDCG precursors were computed inconsistently (some used position discounting, some didn’t).
- All features were batch-only → models could only be evaluated offline the next morning.
- During flash sales or Lunar New Year, models were trained and evaluated on 6–8-hour-old data → immediate online metric degradation.

### 2. What I Designed: Canonical Metric Layer + Feature Schemas

I built a **single source of truth** for both features and metrics using **dbt** (the same migration I mentioned in the first bullet).

**Step-by-step what I delivered:**

**A. Canonical Metric Definitions (the part I’m most proud of)** I created a central macros/metrics.sql file with reusable Jinja macros so **every** model iteration uses the exact same definitions.

SQL

```
-- macros/metric_definitions.sql (used everywhere)
{% macro canonical_ctr_7d(device_filter=none) %}
    SUM(CASE WHEN clicked THEN 1 ELSE 0 END) 
    / NULLIF(SUM(CASE WHEN impression THEN 1 ELSE 0 END), 0)
    OVER (PARTITION BY {{ device_filter or 'all' }} 
          ORDER BY event_date 
          ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
{% endmacro %}

{% macro ndcg_precursor(position) %}
    (clicked * (1 / LOG2({{ position }} + 1))) 
    / NULLIF(SUM(impression * (1 / LOG2(position + 1))) OVER (...), 0)
{% endmacro %}
```

These macros were referenced in **every** dbt model (fct_search_features, fct_rec_features, fct_real_time_evaluation).

Result: From that day forward, NDCG@10 and CTR were **identical** across search team, recsys team, A/B testing platform, and offline evaluation notebooks. No more “which CTR definition are we using?” discussions.

**B. Feature Schemas for Real-Time Model Evaluation** I redesigned the entire feature mart layer in dbt with three layers:

1. **Staging** (stg_*): Raw Kafka + CDC data with polluted-click deduplication (Flink + dbt).
2. **Intermediate** (int_*): All rolling windows (7d/30d/90d) computed once using the canonical macros.
3. **Mart** (fct_*): Final feature tables + **real-time evaluation side outputs**.

The key innovation: I added a new mart model fct_real_time_evaluation that computes:

- Live CTR, NDCG precursors, and predicted vs actual scores **in real time** (via Flink side outputs).
- These are written to ScyllaDB and also backfilled nightly in dbt.

This allowed the ML team to run **real-time model evaluation**:

- New model version deployed → immediate shadow inference on live traffic.
- Compare predicted CTR vs actual CTR within minutes (not next day).
- A/B test results visible in <30 minutes instead of 24 hours.

### 3. How the Pipelines Support Real-Time Model Evaluation

The pipelines I designed are hybrid (Lambda style):

- **Flink streaming layer** (the speed layer I re-architected):
    - Consumes the same Kafka clickstream.
    - Computes all canonical metrics and features in stateful windows.
    - Outputs to ScyllaDB for <2-second freshness + side output to Druid for real-time monitoring.
- **dbt + Airflow batch layer** (the serving + training layer):
    - Nightly backfill of historical features using the exact same canonical macros.
    - Data-quality gates (KS drift + polluted-click checks) run before training DAG starts.
    - Training dataset is built from fct_* marts that are guaranteed to use canonical definitions.

Because everything flows through the same dbt macros + schemas, the offline training data and online serving data are now perfectly aligned. Models can be evaluated in real time against production traffic with zero definition drift.

### 4. Challenges I Faced & How I Solved Them

- **Challenge**: Legacy teams wanted to keep their hand-written SQL. **Solution**: Built dbt docs + lineage graphs showing “your old CTR is now this macro”. Ran knowledge-sharing sessions + showed revenue impact of consistent metrics.
- **Challenge**: Real-time + batch consistency. **Solution**: Shared macros + dual-write pattern (Flink writes to ScyllaDB, dbt backfills the same tables).
- **Challenge**: Performance at peak (5× traffic). **Solution**: Incremental dbt models + Flink state TTL + 10% stratified sampling for drift checks.
- **Challenge**: Making NDCG precursors work in streaming. **Solution**: Implemented position-aware discounting inside Flink using event-time windows and watermarks.

### 5. Impact & Why This Was Transformative

- Model evaluation cycle: daily batch → real-time (minutes).
- Metric consistency: 100% across all teams and iterations.
- During 2025 seasonal peaks: zero definition-related bugs.
- ML team feedback: “For the first time we can trust that the features and metrics we train on are exactly the same as what runs online.”
- This directly enabled faster iteration (new model versions tested same-day) and protected revenue during high-traffic periods.

### 1. High-Level Flow (Real-Time Eval → Amplitude)

text

```
Flink (streaming feature computation)
    ↓ (side outputs + shadow inference)
Real-time Evaluation Events (predicted vs actual metrics)
    ↓ (Kafka topic + direct SDK)
Amplitude (A/B experiment tracking + metric monitoring)
```

### 2. What Data We Stream (the actual online eval payload)

Every time a recommendation is served (or every 5 minutes in aggregate), the system emits a rich event to Amplitude with:

- **Experiment metadata**experiment_id, variant (e.g., “model_v2_high_decay”, “control”), user_id, session_id
- **Canonical metrics (real-time)**
    - predicted_ctr (from shadow model)
    - actual_ctr (observed click/impression)
    - ndcg_precursor (position-discounted)
    - final_score vs base_score (intent boost applied)
- **Feature values that drove the prediction**ctr_mobile_7d, is_sale_period, category_intent_score, etc. (sampled 1% for cost reasons)
- **Business context**is_peak_season, device_type, master_id, position

These events are emitted **in real time** so Amplitude dashboards show live lift in NDCG@10 and CTR within 2–5 minutes of traffic.

### 3. Technical Implementation (how I made it stream reliably)

**A. Flink Side-Output for Shadow Evaluation** In the main Flink job (the same one that computes user intent and features), I added a **side output** stream:

Java

```
// Inside the main Flink ProcessFunction
if (shouldRunShadowInference()) {
    double predictedCTR = shadowModel.predict(features);   // lightweight XGBoost inference
    double actualCTR = observedClick ? 1.0 : 0.0;
    
    EvaluationEvent event = new EvaluationEvent(
        userId, experimentId, variant,
        predictedCTR, actualCTR, ndcgPrecursor,
        features  // map of key metrics
    );
    
    ctx.output(shadowEvalSideOutput, event);
}
```

This side output is **non-blocking** — the main recommendation path continues at <2 ms latency.

**B. Routing to Amplitude** Two parallel paths (for reliability):

1. **Real-time path** (low latency): Flink side output → Kafka topic real_time_eval_events → lightweight Go consumer (or Flink sink) that calls Amplitude’s HTTP SDK in batches of 100 events. Latency: <30 seconds from impression to Amplitude.
2. **Batch fallback** (for high-volume safety): Every 5 minutes, a small Airflow task aggregates the Kafka topic and sends summarized events to Amplitude (prevents rate limits during peaks).

**C. A/B Bucketing Integration**

- Amplitude experiment variants are assigned in the Go serving layer (using the same bucketing logic as Amplitude’s own SDK).
- The variant is attached to every recommendation request and flows through Flink → shadow model → eval event.
- This lets PMs and ML engineers see live charts in Amplitude like:
    - “Variant A vs Control: CTR lift +3.2%”
    - “NDCG@10 delta over last 30 min”

### 4. Why This Works So Well for Real-Time Model Evaluation

- **Canonical metrics guarantee**: All events use the exact same dbt macros for CTR and NDCG precursors (no definition drift).
- **Real-time vs offline alignment**: The shadow model uses the exact same feature schemas as production training.
- **Seasonal & drift safety**: If covariate drift is detected, the eval events are automatically tagged with drift_flagged: true and filtered out of Amplitude experiment dashboards until the gate is cleared.
- **Scale**: Handles 1,000+ QPS with <1% sampling on feature-heavy events.

### 5. Real Impact I Saw

- Before: Model evaluation took 24 hours + manual Excel work.
- After: New model versions could be A/B tested **live** in Amplitude. We went from 1–2 model iterations per week to 5–8 per week during peak seasons.
- During Lunar New Year 2025, we caught a concept drift in <15 minutes via Amplitude’s live CTR divergence chart and rolled back the bad variant instantly.