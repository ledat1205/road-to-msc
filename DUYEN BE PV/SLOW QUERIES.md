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