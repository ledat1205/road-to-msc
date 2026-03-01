### 1. Quick Recap – What Each Does

|Command|What it returns|Cost / Side Effect|When to use it|
|---|---|---|---|
|EXPLAIN|**Estimated** query plan (how the DB thinks it will execute)|No cost — does not run the query|First step: see the plan without running the query|
|EXPLAIN ANALYZE|**Actual** execution plan + real timings, row counts, memory usage, etc.|**Runs the query** — can be expensive/slow|When you need real numbers (most debugging cases)|

**Rule of thumb**:

- Start with plain EXPLAIN (safe, fast)
- If you need truth → EXPLAIN ANALYZE (but be careful in production)

### 2. PostgreSQL – Most Powerful & Detailed

PostgreSQL has the best EXPLAIN output — very detailed and accurate.

#### Step-by-step Debugging Flow

1. **Run plain EXPLAIN first** (always safe)
    
    SQL
    
    ```
    EXPLAIN
    SELECT * FROM orders
    WHERE created_at > '2025-01-01'
      AND status = 'PENDING'
    ORDER BY created_at DESC
    LIMIT 50;
    ```
    
    Look for red flags:
    
    - **Seq Scan** (full table scan) on large table → missing index
    - **Nested Loop** with high row estimates → bad join order
    - **High cost** numbers (the numbers are arbitrary units, but higher = slower)
    - **Rows Removed by Filter** very high → index not covering filter
2. **Run EXPLAIN ANALYZE** (when you want real execution stats)
    
    SQL
    
    ```
    EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
    SELECT * FROM orders
    WHERE created_at > '2025-01-01'
      AND status = 'PENDING'
    ORDER BY created_at DESC
    LIMIT 50;
    ```
    
    Key things to look at in output:
    
    - **Actual time** = real time spent (in ms)
    - **Rows** = how many rows were actually processed vs estimated
    - **Loops** = how many times the node ran (high loops = bad)
    - **Buffers: shared hit/read** → how much data came from memory vs disk
    - **Planning Time** + **Execution Time** → total time
    - **Filter** or **Rows Removed by Filter** → if very high → bad index/filter
    
    Example bad output:
    
    text
    
    ```
    Seq Scan on orders  (cost=0.00..123456.78 rows=500000 width=200) (actual time=0.123..4500.456 rows=120000 loops=1)
      Filter: ((created_at > '2025-01-01'::date) AND (status = 'PENDING'::text))
      Rows Removed by Filter: 8800000
    ```
    
    → Full table scan + filtered out 8.8 million rows → needs index on (created_at, status)
    
3. **Add useful options** (make output more readable)
    
    - EXPLAIN (ANALYZE, BUFFERS, VERBOSE) → shows column names, buffer hits/misses
    - EXPLAIN (ANALYZE, FORMAT JSON) → easier to read in tools like pgAdmin or online explainers
    - EXPLAIN (ANALYZE, TIMING OFF) → run fast without timing (for very slow queries)
4. **Visualize the plan** (strongly recommended)
    
    - Copy output → paste into [https://explain.depesz.com/](https://explain.depesz.com/) or [https://tanzu.vmware.com/content/blog/postgresql-explain-visualized](https://tanzu.vmware.com/content/blog/postgresql-explain-visualized)
    - Look for thick arrows (high row counts) and red nodes (Seq Scan, high cost)

### 3. MySQL / MariaDB

MySQL’s EXPLAIN is simpler, but still very useful.

**Basic usage**:

SQL

```
EXPLAIN
SELECT * FROM orders
WHERE created_at > '2025-01-01'
  AND status = 'PENDING'
ORDER BY created_at DESC
LIMIT 50;
```

**Key columns to check**:

- type: ALL = full table scan (bad), range, ref, eq_ref, index = good
- rows: estimated rows scanned (high = bad)
- Extra: “Using filesort” (slow sort), “Using temporary” (temp table), “Using where” with high rows = bad
- key: which index used (NULL = no index → bad)

**With real execution** (MySQL 8.0.18+):

SQL

```
EXPLAIN ANALYZE
SELECT * FROM orders WHERE created_at > '2025-01-01' AND status = 'PENDING';
```

Shows actual time, loops, rows — similar to PostgreSQL.

**Quick fix pattern**:

- type = ALL + large table → add index on created_at, status
- Using filesort → add index on sort column (created_at DESC)

### 4. Common Patterns & What to Do

|Pattern in EXPLAIN / ANALYZE|What it means|Fix (in order of preference)|
|---|---|---|
|Seq Scan / type=ALL on large table|Full table read → very slow|Add index on WHERE clause columns|
|High Rows Removed by Filter|Index exists but not selective enough|Make index more specific (include status, etc.)|
|Using filesort / temporary table|Sort or group without index|Add index on ORDER BY / GROUP BY columns|
|Nested Loop with high row count|Bad join order or missing index on join column|Add index on join keys, rewrite query|
|High actual time vs estimated time|Bad statistics → outdated planner info|Run ANALYZE table_name or VACUUM ANALYZE|
|Buffers: shared read >> hit|Lots of disk I/O|Increase shared_buffers, add index, cache warm-up|

### 5. Practical Tips for Production Debugging

1. **Never run EXPLAIN ANALYZE on huge production queries without LIMIT** → Can take minutes/hours and load the DB → Use EXPLAIN first, or add LIMIT 1000 for testing
2. **Use pgBadger / pg_stat_statements** (PostgreSQL)
    - Find slow queries automatically
    - SELECT * FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
3. **MySQL: Use PERFORMANCE_SCHEMA or slow query log**
    - Enable slow query log: slow_query_log = 1, long_query_time = 1
    - Then analyze with mysqldumpslow or pt-query-digest
4. **Visual tools** (highly recommended):
    - **pgAdmin / DBeaver**: EXPLAIN button → graphical plan
    - **[https://explain.depesz.com/](https://explain.depesz.com/)** → paste EXPLAIN ANALYZE text → beautiful tree
    - **EverSQL** or **pgMustard** → online analyzers with suggestions