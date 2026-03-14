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