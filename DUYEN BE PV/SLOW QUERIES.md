### Solutions for ClickHouse

Here’s a comprehensive list of solutions to prevent or mitigate slow/timeout issues in long-running (hours-long) queries in ClickHouse. These are drawn from best practices in 2026, focusing on optimization first (preferred over just increasing timeouts), monitoring, and configuration. Prioritize identifying slow queries via `system.query_log` before applying fixes.

| Category | Solutions |
|----------|-----------|
| **Query Optimization** | - Use filters on `ORDER BY` columns first for efficient pruning.<br>- Enable proper partition pruning with date/time keys.<br>- Avoid `SELECT *`; select only necessary columns to reduce data read.<br>- Use `PREWHERE` for heavy filters (e.g., non-indexed columns) to scan less data.<br>- Optimize `GROUP BY` with low-cardinality keys or approximations (e.g., `uniqCombined`).<br>- Rewrite joins to use `JOIN` with indexed columns; prefer `LEFT JOIN` over subqueries.<br>- Use `EXPLAIN` or `EXPLAIN ESTIMATE` to analyze plans and spot full scans.<br>- Disable filesystem cache during testing (`SET enable_filesystem_cache = 0`) for accurate benchmarks. |
| **Indexing & Schema** | - Add skip indexes (e.g., minmax, bloom_filter) for common filters.<br>- Use proper data types (e.g., LowCardinality for strings) and compression codecs.<br>- Switch to `AggregatingMergeTree` for pre-aggregated data to speed up summaries.<br>- Materialize views for complex computations to pre-compute results. |
| **Configuration Tuning** | - Increase `max_threads` (start at half CPU cores) for parallel execution.<br>- Set `max_memory_usage` higher for large datasets, but monitor OOM.<br>- Use `max_bytes_before_external_group_by`/`max_bytes_before_external_sort` to spill to disk and avoid OOM.<br>- Enable query queuing (`<queue_max_wait_ms>5000</queue_max_wait_ms>`) instead of rejection.<br>- Adjust `max_concurrent_queries` to limit overload. |
| **Monitoring & Management** | - Log slow queries with `log_queries_min_query_duration_ms`.<br>- Kill long queries: `KILL QUERY WHERE query_id = 'id'` or by user/pattern.<br>- Use connection pooling to manage concurrent queries.<br>- Increase timeouts only as last resort (e.g., `SET max_execution_time = 300`). |
| **Hardware/Cluster** | - Use SSDs for faster I/O; scale shards/replicas for distribution.<br>- Batch inputs properly to avoid small inserts. |

### Solutions for Apache Druid

For Apache Druid, focus on segment optimization and query patterns first, as long-running queries often stem from small segments or unfiltered scans. Use request logging to identify issues.

| Category | Solutions |
|----------|-----------|
| **Query Optimization** | - Always filter on time (__time) and dimensions first.<br>- Minimize inequality filters; they’re resource-heavy.<br>- Use approximations (e.g., HyperLogLog for cardinality) for high-cardinality GROUP BY.<br>- Avoid large IN clauses (~14k+ elements slow planning).<br>- Break complex queries into subqueries or use materialized views. |
| **Data/Segment Tuning** | - Compact small segments into larger ones (target 5M rows/segment) via compaction tasks to reduce overhead.<br>- Optimize segment sizes for better query routing and I/O.<br>- Use rollup during ingestion to pre-aggregate. |
| **Configuration Tuning** | - Enable query caching (result/segment) for repeated queries.<br>- Increase JVM heap/GC tuning on Historicals/Brokers for large intermediates.<br>- Set higher `druid.server.http.maxIdleTime` to avoid web timeouts.<br>- Use query laning to isolate slow queries from real-time ones.<br>- Adjust query context: higher `timeout` or `perSegmentTimeout`. |
| **Monitoring & Management** | - Enable request logging for query-level monitoring to spot slow ones.<br>- Kill queries via Historical logs or API if they exceed timeouts.<br>- Use service tiering for dedicated resources on priority queries. |
| **Hardware/Cluster** | - Scale Historical nodes for parallelism; use SSDs for segment cache.<br>- Batch ingestion to avoid small segments. |

### Solutions for PostgreSQL

PostgreSQL handles long-running queries via timeouts and optimization. Start with logging slow queries and EXPLAIN ANALYZE.

| Category | Solutions |
|----------|-----------|
| **Query Optimization** | - Use EXPLAIN ANALYZE to identify seq scans; add indexes on WHERE/JOIN/ORDER BY columns.<br>- Rewrite queries: avoid complex subqueries, use CTEs, denormalize if needed.<br>- Run ANALYZE/VACUUM to update stats and reduce bloat.<br>- Use materialized views for pre-computed results. |
| **Timeouts & Killing** | - Set `statement_timeout` (e.g., '30s') per session/role/server to auto-kill long queries.<br>- Use `pg_cancel_backend(pid)` to softly kill; `pg_terminate_backend(pid)` for hard kill.<br>- Set `lock_timeout` to prevent long lock waits.<br>- Use `idle_in_transaction_session_timeout` for idle tx. |
| **Configuration Tuning** | - Increase `work_mem` for sorts/joins, but per-operation to avoid OOM.<br>- Tune `shared_buffers` (25% RAM) and `effective_cache_size`.<br>- Adjust `checkpoint_completion_target` and `checkpoint_timeout` for smoother I/O.<br>- Use SERIALIZABLE isolation for consistency, with app retries. |
| **Monitoring & Logging** | - Log slow queries: set `log_min_duration_statement = 1000`.<br>- Enable `pg_stat_statements` and `auto_explain` for slow query plans.<br>- Monitor with `pg_locks`, `pg_stat_activity` for blockers. |
| **Hardware/Cluster** | - Use connection pooling (PgBouncer) to limit connections.<br>- Scale with read replicas for offloading analytics.<br>- Use SSDs; tune autovacuum for aggressive cleanup. |
### 1. "A query is slow in production — walk me through your step-by-step debugging process."

**Why asked**: This is the #1 slow-query question. It reveals if you panic-fix (e.g., blindly add indexes) or systematically diagnose.

**Strong structured answer** (use this framework in interviews):

1. **Reproduce & baseline** — Run the query in a safe environment (staging or with LIMIT if safe), time it, note params/load.
2. **Check slow query log / monitoring** — Enable log_min_duration_statement = 250 (ms) in prod; use pgBadger or Datadog/New Relic for aggregation.
3. **Capture the plan** — EXPLAIN (ANALYZE, BUFFERS, TIMING, VERBOSE, SETTINGS) the exact query. Compare est. vs. actual rows.
4. **Look for red flags in EXPLAIN**:
    - Seq Scan / high Rows Removed by Filter → bad selectivity / missing index.
    - High Heap Fetches / random I/O → non-covering index.
    - Nested Loop with high rows → bad join order / missing stats.
    - Hash Join spilling to disk → low work_mem.
    - Parallelism disabled or ineffective → tune max_parallel_workers.
5. **Check system-level** — pg_stat_activity for blocking/long tx, pg_stat_statements for top consumers, disk I/O wait (iostat), CPU/memory pressure.
6. **Stats & vacuum** — pg_stat_user_tables (n_dead_tup, last_analyze), run ANALYZE if outdated.
7. **Application context** — Is it N+1 from ORM? Parameter sniffing? Frequent small queries vs. one big one?
8. **Test fixes iteratively** — Add index → re-EXPLAIN, tune GUCs session-level first, rewrite query.

**Follow-up**: "What if it's intermittent/sometimes slow?" → Parameter-dependent plans (use pg_stat_statements + query params), lock contention, cache misses, autovacuum storms.

### 2. "How do you use pg_stat_statements to find and debug slow queries?"

**Key points**:

- Enable it (usually default in modern PG): shared_preload_libraries = 'pg_stat_statements', pg_stat_statements.track = all.
- Query top slow ones:
    
    SQL
    
    ```
    SELECT query, calls, total_exec_time, mean_exec_time, rows, shared_blks_hit + shared_blks_read AS total_blocks
    FROM pg_stat_statements
    ORDER BY total_exec_time DESC LIMIT 20;
    ```
    
- Look for: high total_exec_time (cumulative), high mean but low calls (bad outliers), high rows but low time (cached?).
- Reset: SELECT pg_stat_statements_reset(); after testing.
- Interview tip: Mention extensions like pg_stat_statements + check_postgres.pl or tools like pgHero/PgAnalyze.

### 3. "You see a query with perfect indexes but still slow — why? How to debug?"

**Common culprits** (beyond seq scan):

- **Locking/blocking** — Long-running tx holding locks → check pg_locks + pg_blocking_pids(pid).
- **Disk I/O saturation** → BUFFERS shows many shared_blks_read → add RAM or better storage.
- **Work_mem too low** → Sort/hash spills to disk → increase (but watch OOM), test with SET work_mem = '64MB';.
- **Bad cardinality / correlated columns** → CREATE STATISTICS on (col1, col2).
- **Visibility map not set** → Index-only scan falls back → run VACUUM on hot tables.
- **Parallel query overhead** → Disable with SET max_parallel_workers_per_gather = 0; to test.
- **Planner bugs/edge cases** → Use pg_hint_plan extension temporarily to force better plan.

### 4. "Explain a time you fixed a slow query — what was the root cause and fix?"

**Behavioral variant** — Prepare a real (or realistic) story:

- Example: "Query joining orders + users on customer_id, 10s → EXPLAIN showed nested loop with 1M+ inner scans. Root: No index on orders.customer_id. Fix: Added index → down to 15ms. But then high heap fetches → made covering index with INCLUDE (status, amount) → index-only scan, 3ms."
- Another: "Intermittent spikes → autovacuum killed I/O. Tuned scale_factor=0.05 on table → stable."

### 5. "How do you handle slow complex queries with many joins / subqueries / window functions?"

**Approach**:

- Push filters early (WHERE before JOIN).
- Rewrite correlated subqueries → LEFT JOIN or EXISTS.
- Use CTEs judiciously (materialized in PG 12+ with MATERIALIZED keyword).
- Break into temp tables for very complex analytics.
- Consider materialized views for repeated expensive computations.

### 6. "Difference in slow query debugging between PostgreSQL and MySQL?"

**Quick comparison** (bonus points):

- Postgres: EXPLAIN ANALYZE + pg_stat_statements + BUFFERS → very detailed.
- MySQL: EXPLAIN (no ANALYZE until 8.0+), slow query log with long_query_time, SHOW PROCESSLIST, PERFORMANCE_SCHEMA.
- MySQL: Optimizer hints more common (FORCE INDEX), InnoDB buffer pool tuning critical.
- Postgres: Better at complex analytics, MySQL faster for simple OLTP.

### 7. "What tools/extensions do you use for slow query investigation in production?"

**Popular answers**:

- pgBadger (log analyzer).
- check_postgres.pl or pgmetrics.
- auto_explain extension (logs slow plans automatically).
- pg_stat_statements + visualization (Metabase, Grafana).
- External: pgAnalyze, EverSQL, Datadog APM, New Relic.

**Interview tip**: Emphasize "measure first, change second" — always EXPLAIN before/after any fix, and prefer config/index/query changes over disabling features (e.g., no global enable_seqscan=off).