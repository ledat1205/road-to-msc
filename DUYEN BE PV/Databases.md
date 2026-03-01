### 1. Data Modeling and Schema Design at Scale

Interviewers test if you can avoid future pain points like slow joins, bloat, or impossible migrations.

1. How would you design a schema for a high-traffic system like Twitter/X (tweets, users, follows, likes) handling 500M+ users and billions of interactions?
    - Why asked: Tests normalization vs. denormalization, fan-out patterns, eventual consistency.
    - Key points: Hybrid approach — normalized core (users, tweets), denormalized feeds (materialized timelines via separate tables or Redis), partitioning by user_id or time, soft deletes.
2. When do you choose denormalization over normalization in a scaled system? Give concrete examples.
    - Trade-offs: Normalization reduces anomalies but increases join cost; denormalization speeds reads but complicates writes/updates.
    - Examples: Duplicate user info in post table for fast rendering; pre-computed counts (followers_count) updated via triggers/jobs.
3. UUID vs. auto-increment IDs for primary keys in sharded/distributed systems — pros/cons?
    - UUID: Globally unique, no central coordination, better for sharding/merging.
    - Auto-increment: Sequential → better locality/index performance, but hotspot on insert in distributed setups.
    - At scale: UUIDv7 (time-ordered) or Snowflake IDs often win.
4. How do you perform zero-downtime schema migrations on a 1TB+ database with high write traffic?
    - Tools: pt-online-schema-change (Percona for MySQL), pg_squeeze / logical replication (Postgres), gh-ost.
    - Steps: Add new column → backfill → dual-write → cutover → drop old.
5. Design partitioning strategy for a 100B+ row time-series table (e.g., metrics, logs).
    - Range partitioning by date (e.g., monthly), list by tenant/region.
    - Declarative partitioning in Postgres 10+, sub-partitioning.
    - Benefits: Faster queries/pruning, easier archiving (drop old partitions).
6. How to model multi-tenancy at scale (SaaS app with 100k+ tenants)?
    - Options: Shared DB + tenant_id column (with row-level security in Postgres), separate schemas per tenant, separate DBs (costly).
    - Trade-offs: Isolation vs. cost/ease of management.
7. Soft deletes vs. hard deletes in high-write systems — when to use each?
    - Soft: Keep history, easier recovery, but increases table size → needs partitioning/VACUUM.
    - Hard: Cleaner, but audit/compliance issues.

### 2. Query Optimization and Performance Tuning at Scale

Expect EXPLAIN ANALYZE walkthroughs or "fix this slow query" scenarios.

8. Walk through optimizing a query that joins 5 tables and scans 100M+ rows slowly.
    - Steps: EXPLAIN (ANALYZE, BUFFERS), check seq scans → add indexes, push filters early, rewrite joins (hash vs. nested loop), limit rows fetched.
9. Why does Postgres/MySQL sometimes choose seq scan over index even with index present?
    - Low selectivity, bad stats (ANALYZE needed), parameter tuning (random_page_cost too high on SSD), functional conditions without expression index.
10. Explain covering indexes and index-only scans — how to achieve Heap Fetches: 0 in Postgres.
    - Include all needed columns in index (INCLUDE in Postgres), ensure visibility map bits set via VACUUM.
    - Huge win for COUNT(*) or SELECT few cols on large tables.
11. How to tune autovacuum in Postgres for a write-heavy table with frequent UPDATE/DELETE?
    - Lower scale_factor (0.01–0.05), increase workers, monitor pg_stat_user_tables for bloat.
    - Per-table settings, aggressive ANALYZE.
12. N+1 query problem in ORMs at scale — how to detect and fix?
    - Detection: Slow query log, APM tools.
    - Fix: Eager loading (JOIN FETCH), batching, materialized views.
13. Bitmap index scans vs. regular index scans — when does Postgres prefer them?
    - Multiple conditions on different indexes → bitmap AND/OR → efficient for medium selectivity.
14. How to handle full-text search on billions of rows efficiently?
    - Postgres: GIN on tsvector, trigram for fuzzy.
    - MySQL: Full-text indexes (limited), or external like Elasticsearch.

### 3. Transactions, Concurrency, and Locking at Scale

15. Explain MVCC in Postgres vs. MySQL InnoDB — differences in implementation and impact.
    - Both MVCC, but Postgres version tuples per row; MySQL gap/next-key locking in RR to prevent phantoms.
    - Postgres better for long reads; MySQL can deadlock more on writes.
16. Isolation levels deep dive — when to use Serializable, and perf cost at scale?
    - Serializable prevents write skew/phantom; expensive (predicate locking/snapshot isolation fallback).
    - Use for financial; Read Committed default for most.
17. How to detect/prevent deadlocks in production?
    - Logs, pg_locks / SHOW ENGINE INNODB STATUS.
    - Retry logic, consistent lock order, short transactions.
18. Optimistic vs. pessimistic locking — examples and when to use at scale.
    - Optimistic: version column + WHERE version=old → good for low conflict.
    - Pessimistic: SELECT FOR UPDATE → good for high conflict but blocks.
19. Long-running transactions — problems and mitigations.
    - Prevent vacuum → bloat, replication lag, xmin horizon.
    - Break into batches, use advisory locks.

### 4. Scaling Strategies: Replication, Sharding, Partitioning, HA

20. Vertical vs. horizontal scaling for SQL — limits and when to switch.
    - Vertical: Easier, up to ~64–128 cores / TB RAM.
    - Horizontal: Sharding/replication needed beyond single node limits (write bottleneck ~10k–50k TPS typical).
21. Streaming replication in Postgres vs. binlog replication in MySQL — setup, lag handling, failover.
    - Postgres: Physical (WAL) → exact copy, async/sync.
    - MySQL: Logical (binlog) + GTID for easier failover.
    - Tools: Patroni/Repmgr (Postgres), MHA/Orchestrator (MySQL).
22. How to implement read scaling with replicas? Handling replication lag?
    - Route reads to replicas (e.g., via proxy like ProxySQL, pgBouncer).
    - Lag monitoring → stale reads → eventual consistency trade-off.
23. Sharding strategies: Range, hash, directory lookup — pros/cons.
    - Hash (consistent hashing): Even distribution, re-sharding pain.
    - Range: Hotspots possible, easier queries on ranges.
    - At scale: Vitess (MySQL), Citus (Postgres extension).
24. What is application-level sharding vs. DB-native (Citus/Vitess)?
    - App: Custom routing logic → flexible but error-prone.
    - Native: Transparent queries, cross-shard joins limited.
25. Handling cross-shard transactions/queries at scale?
    - Avoid if possible (eventual consistency, sagas).
    - Or use distributed tx (XA/2PC — slow), or aggregate via ETL.
26. High availability setups — multi-AZ, multi-region, failover time?
    - Postgres: Synchronous replication + Patroni → sub-30s failover.
    - MySQL Group Replication → multi-master, auto-failover.

### 5. Monitoring, Operations, and Reliability at Scale

27. Key metrics to monitor for SQL DB health at scale?
    - QPS, replication lag, connection count, cache hit ratio, vacuum/dead tuple stats, slow queries (pg_stat_statements), disk I/O.
28. How to handle database incidents (e.g., replication stopped, out of disk)?
    - Runbooks: Promote replica, point-in-time recovery (PITR via WAL/binlog).
29. Backups and PITR strategy for petabyte-scale DB?
    - Logical (pg_dump slow), physical (pg_basebackup + WAL archiving).
    - Tools: Barman/WAL-G for Postgres.
30. When would you choose SQL over NoSQL (or vice versa) in a scaled backend?
    - SQL: Strong consistency, complex joins, transactions needed.
    - NoSQL: Extreme write scale, flexible schema, eventual consistency OK.



### 1. How would you design a schema for a high-traffic system like Twitter/X (tweets, users, follows, likes) handling 500M+ users and billions of interactions?

At this scale, pure relational design fails due to join explosion, write hotspots, and read latency. Use a **hybrid approach** combining normalization for core data integrity and heavy denormalization/materialization for reads.

**Core Normalized Tables** (Postgres/MySQL style):
- `users` (user_id PK, username UNIQUE, bio, created_at, followers_count, following_count, tweet_count — counters updated async).
- `tweets` (tweet_id PK UUIDv7, user_id FK, text, created_at, reply_to_tweet_id, retweet_of_tweet_id).
- `follows` (follower_id, followee_id, created_at — composite PK (follower_id, followee_id)).
- `likes` (user_id, tweet_id, created_at — composite PK (user_id, tweet_id)).

**Denormalized / Materialized for Scale**:
- **Timelines**: Separate tables for user home timeline and mentions (fan-out on write).
  - `user_timelines` (user_id, tweet_id, inserted_at, source_type — e.g., 'follow', 'mention').
  - Use partitioning by `user_id` (hash) or time.
  - Write path: When a user tweets, fan-out to followers' timelines via async jobs (Kafka + workers) — limit fan-out to ~5k active followers; for celebrities, use pull model (fans pull recent tweets).
- **Counters**: `tweet_stats` (tweet_id, like_count, retweet_count, reply_count) updated via Redis or async counters (avoid real-time joins).
- **Soft deletes**: Add `deleted_at` on tweets/users — mark deleted, archive old ones.
- **Partitioning**: Tweets by `created_at` (monthly), follows by `follower_id` hash.

**Trade-offs & Why**:
- Normalization preserves integrity (e.g., user delete cascades correctly).
- Denormalization avoids joins at read time (critical for timeline feed — <50ms).
- Eventual consistency via async fan-out (Twitter uses this pattern).
- Sharding: Shard by user_id (consistent hashing) using Vitess/Citus.

**Example Write Path**:
1. User tweets → insert into `tweets`.
2. Async job fans out to followers' `user_timelines` (or pushes to Redis sorted sets).
3. Increment counters in Redis → eventual DB sync.

This mirrors Twitter's architecture (Fanout on Write + Pull for high-follower users).

### 2. When do you choose denormalization over normalization in a scaled system? Give concrete examples.

**Normalization** (3NF+) minimizes redundancy/anomalies but increases joins → bad for read-heavy, high-latency systems.

**Denormalization** trades write complexity for read speed — essential when reads >> writes or joins are expensive.

**Choose denormalization when**:
- Read performance is bottleneck (e.g., 100k+ QPS).
- Joins span many tables or large datasets.
- Consistency can be eventual (async updates).
- Data is append-heavy or immutable.

**Examples**:
- **Pre-computed counts**: `users.followers_count` updated via triggers/jobs instead of `COUNT(follows)` join.
- **Duplicate data**: In `tweets`, store `user_username` and `user_avatar_url` (denormalized from users) — avoids user join on every tweet display.
- **Materialized timelines**: Twitter-style `user_timelines` table with tweet copies — read one table instead of joining follows + tweets.
- **Order total**: In e-commerce, `orders.total_amount` computed/stored vs. summing order_items on every query.
- **Full-text denormalization**: Concatenate searchable fields into one column for GIN index.

**Trade-offs**:
- Writes slower/complex (dual-write, eventual consistency via CDC/jobs).
- Anomalies possible (stale data — use TTL or background sync).
- More storage.

**Rule**: Normalize first, denormalize where profiling shows joins >10–20ms or >10% of QPS.

### 3. UUID vs. auto-increment IDs for primary keys in sharded/distributed systems — pros/cons?

**Auto-increment (serial/bigserial)**:
- **Pros**: Sequential → excellent locality (clustered index good for range scans), smaller size (8 bytes), fast inserts on single node.
- **Cons**: Central sequence → hotspot in distributed/sharded systems (all inserts hit same node), merging shards hard (ID collisions), security risk (predictable IDs).

**UUID (uuid_generate_v4 or v7)**:
- **Pros**: Globally unique (no coordination), no hotspot (random distribution), easy sharding/merging, secure (hard to guess).
- **Cons**: Larger (16–36 bytes), slower indexing/inserts (random writes), larger indexes.
- **UUIDv7** (time-ordered): Combines time prefix + random → better locality than v4, monotonic-ish.

**At scale**:
- Use **UUIDv7** or Twitter Snowflake (64-bit time + machine + sequence) for most new systems.
- Auto-increment OK for single-instance or if sharded by ID range (but painful re-sharding).
- Hybrid: Auto-increment for internal, UUID for external/exposed IDs.

**Example**:
```sql
CREATE TABLE tweets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),  -- or uuid_generate_v7()
    ...
);
```

### 4. How do you perform zero-downtime schema migrations on a 1TB+ database with high write traffic?

**Goal**: No downtime, no long locks.

**Tools & Strategies**:
- **MySQL**: gh-ost (triggerless, online) or pt-online-schema-change (Percona, uses triggers).
- **Postgres**: pg_squeeze (repack), logical replication (PG 10+), or tools like pglogical + shadow tables.

**General Steps (gh-ost/pg_squeeze style)**:
1. Create shadow table with new schema (e.g., add column `new_col`).
2. Backfill existing rows (batched, throttled).
3. Set up trigger/CDC to dual-write new/old columns.
4. Swap: Rename old → new, rename shadow → old.
5. Drop old table.
6. Cutover application to use new column.

**Example (Postgres logical replication)**:
- Create new table with desired schema.
- Use `pglogical` to replicate changes.
- Backfill old data.
- Switch app to write to new table.
- Promote new table.

**Best practices**:
- Throttle backfill (e.g., 1k rows/sec).
- Monitor replication lag.
- Test rollback plan.
- Use blue-green deployment for app if needed.

### 5. Design partitioning strategy for a 100B+ row time-series table (e.g., metrics, logs).

**Declarative Partitioning** (Postgres 10+):
- **Range** by timestamp (e.g., `created_at`):
  ```sql
  CREATE TABLE metrics (
      id BIGSERIAL,
      created_at TIMESTAMPTZ NOT NULL,
      metric_name TEXT,
      value NUMERIC
  ) PARTITION BY RANGE (created_at);

  CREATE TABLE metrics_y2026m01 PARTITION OF metrics
      FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
  ```
- Monthly partitions → sub-partition by tenant_id if needed.

**Benefits**:
- Query pruning: `WHERE created_at > '2026-01-01'` skips old partitions.
- Archiving: `DROP TABLE metrics_y2025m01;` instant.
- Maintenance: VACUUM per partition.
- Parallel queries across partitions.

**Alternatives**:
- List partitioning by tenant/region for multi-tenant.
- BRIN index on created_at (cheap, good for time-ordered data).
- TimescaleDB extension for automatic time partitioning + compression.

**Trade-offs**: More partitions = more overhead (planning time), but 100–500 partitions OK.

### 6. How to model multi-tenancy at scale (SaaS app with 100k+ tenants)?

**Options**:
1. **Shared DB, shared schema + tenant_id** (most common):
   - Add `tenant_id` to every table.
   - Row-level security (RLS) in Postgres: `CREATE POLICY tenant_isolation ON users USING (tenant_id = current_setting('app.current_tenant')::uuid);`
   - Pros: Cheap, easy joins, one backup.
   - Cons: Noisy neighbors, hard isolation.

2. **Shared DB, separate schemas per tenant**:
   - `CREATE SCHEMA tenant_abc;` — each tenant has own tables.
   - Pros: Better isolation, schema-level backups.
   - Cons: Schema explosion (planning overhead), cross-tenant queries hard.

3. **Separate DBs per tenant/group**:
   - Pros: Strong isolation, independent scaling.
   - Cons: Expensive, management hell (100k DBs impossible).

**Best at scale**: Shared DB + tenant_id + RLS + connection pooling (set tenant per session). Shard by tenant_id if one tenant dominates.

**Example**:
```sql
SET app.current_tenant = 'tenant123';
SELECT * FROM users;  -- RLS enforces tenant_id
```

### 7. Soft deletes vs. hard deletes in high-write systems — when to use each?

**Soft deletes**:
- Add `deleted_at TIMESTAMPTZ`, `is_deleted BOOL`.
- `WHERE deleted_at IS NULL` on queries.
- Pros: Easy recovery, audit trail, no cascade issues.
- Cons: Table bloat (needs partitioning), slower queries (extra filter), VACUUM pressure.
- Use when: Compliance (GDPR right-to-be-forgotten exceptions), frequent undeletes, history needed.

**Hard deletes**:
- `DELETE FROM ...`
- Pros: Clean disk, faster long-term.
- Cons: Irreversible, audit loss, cascade complexity.
- Use when: Data truly ephemeral, no compliance needs, bloat unacceptable.

**Hybrid**: Soft delete + background job to hard delete after 30–90 days.

**Mitigation for soft**: Partition by `deleted_at` or use partial indexes `WHERE deleted_at IS NULL`.

### 8. Walk through optimizing a query that joins 5 tables and scans 100M+ rows slowly.

**Steps**:
1. Run `EXPLAIN (ANALYZE, BUFFERS)` — identify seq scans, high cost joins, bad estimates.
2. Check row estimates vs. actual (huge gap → ANALYZE or CREATE STATISTICS).
3. Push filters early: Move WHERE conditions into subqueries or JOIN ON.
4. Rewrite joins: Prefer hash joins (large tables) over nested loops (small + large).
5. Add indexes on join keys + filters (composite if needed).
6. Limit fetched columns (covering indexes).
7. Use CTEs for materialization if complex.
8. Increase work_mem if hash spills.

**Example slow query**:
```sql
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
JOIN products p ON o.product_id = p.id
JOIN addresses a ON o.address_id = a.id
JOIN payments pay ON o.id = pay.order_id
WHERE o.created_at > '2026-01-01';
```

**Optimized**:
- Add indexes: `ON orders (created_at, user_id)`, etc.
- Rewrite to:
  ```sql
  SELECT o.id, u.name, p.name, ...  -- only needed columns
  FROM orders o
  JOIN users u ON o.user_id = u.id
  ...
  WHERE o.created_at > '2026-01-01';
  ```
- Use `EXPLAIN` to confirm hash joins or index scans.

### 9. Why does Postgres/MySQL sometimes choose seq scan over index even with index present?

**Reasons**:
- Low selectivity (>10–20% rows returned) — seq scan cheaper.
- Outdated stats — `ANALYZE` table.
- Non-sargable: `WHERE func(col) = value` — no index match.
- `random_page_cost` too high (default 4 → lower to 1.1 on SSD).
- Small table — overhead > benefit.
- Bad estimates — correlated columns (use CREATE STATISTICS).

**Debug**: Compare est. vs. actual rows in EXPLAIN ANALYZE.

### 10. Explain covering indexes and index-only scans — how to achieve Heap Fetches: 0 in Postgres.

**Covering index**: Index contains all columns query needs (SELECT + WHERE + ORDER).
**Index-only scan**: Read index only, no heap access.

**To achieve Heap Fetches: 0**:
- Use `INCLUDE`:
  ```sql
  CREATE INDEX orders_cover ON orders (customer_id) INCLUDE (status, amount);
  ```
- Query: `SELECT customer_id, status, amount FROM orders WHERE customer_id = 123;`
- Ensure pages all-visible (run VACUUM regularly).

**Result**: Index-only scan, 5–50× faster for read-heavy.

### 11. How to tune autovacuum in Postgres for a write-heavy table with frequent UPDATE/DELETE?

**Default conservative** → bloat on hot tables.

**Per-table tuning**:
```sql
ALTER TABLE orders SET (autovacuum_vacuum_scale_factor = 0.05);
ALTER TABLE orders SET (autovacuum_vacuum_threshold = 1000);
ALTER TABLE orders SET (autovacuum_analyze_scale_factor = 0.02);
```

**Global**:
- Increase `autovacuum_max_workers = 8`.
- Lower `autovacuum_naptime = 15s`.
- Raise `autovacuum_vacuum_cost_limit = 5000`.

**Monitor**:
```sql
SELECT relname, n_dead_tup, last_autovacuum FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;
```

### 12. N+1 query problem in ORMs at scale — how to detect and fix?

**Problem**: Loop fetches related data (e.g., 100 users → 100 queries for posts).

**Detect**:
- Slow query logs show many similar queries.
- APM (New Relic, Datadog) shows N+1 patterns.

**Fix**:
- Eager loading: `User.with_posts()` or JOIN FETCH.
- Batch: `WHERE id IN (list)`.
- Materialized views for aggregates.
- Cache (Redis) for hot data.

**Example (SQLAlchemy)**:
```python
users = db.query(User).options(joinedload(User.posts)).all()
```

### 13. Bitmap index scans vs. regular index scans — when does Postgres prefer them?

**Regular index scan**: Single index, direct TID lookup.

**Bitmap scan**: Builds bitmap from multiple indexes → AND/OR → then heap fetch.
- Prefers when: Multiple conditions (e.g., `status = 'active' AND country = 'US'` with separate indexes).
- Medium selectivity (too many rows for index-only, too few for seq).
- Efficient for OR conditions or combining indexes.

**EXPLAIN shows**: Bitmap Heap Scan + Bitmap Index Scan.

### 14. How to handle full-text search on billions of rows efficiently?

**Postgres**:
- `tsvector` + GIN index:
  ```sql
  ALTER TABLE posts ADD COLUMN tsv tsvector GENERATED ALWAYS AS (to_tsvector('english', title || ' ' || body)) STORED;
  CREATE INDEX posts_fts ON posts USING GIN(tsv);
  ```
- Query: `WHERE tsv @@ to_tsquery('cat & dog')`
- Trigram for fuzzy: `pg_trgm` extension.

**MySQL**: Built-in FULLTEXT (limited) or migrate to Elasticsearch.

**Scale**: External search (Elasticsearch/OpenSearch) for complex ranking.

### 15. Explain MVCC in Postgres vs. MySQL InnoDB — differences in implementation and impact.

**Both MVCC** (multi-version concurrency control):
- **Postgres**: Each UPDATE creates new row version; old versions live until vacuumed. No gap locking in Repeatable Read.
  - Pros: No read blocking, great for analytics.
  - Cons: Bloat → needs vacuum.
- **MySQL InnoDB**: Versions in undo logs; Repeatable Read uses next-key/gap locking to prevent phantoms.
  - Pros: Less bloat.
  - Cons: More deadlocks, reads can block writes.

**Impact**: Postgres better for long-running reads; MySQL for write-heavy OLTP with short tx.

### 16. Isolation levels deep dive — when to use Serializable, and perf cost at scale?

**Levels**:
- Read Uncommitted: Dirty reads.
- Read Committed: Default (no dirty).
- Repeatable Read: No non-repeatable reads.
- Serializable: No phantoms/write skew.

**Serializable**: Uses SSI (serializable snapshot isolation) in Postgres → detects conflicts, aborts.
- Cost: High CPU (conflict checks), aborts under contention.
- Use for: Financial (prevent double-spend), rare at scale.

**Default**: Read Committed or Repeatable Read.

### 17. How to detect/prevent deadlocks in production?

**Detect**:
- Postgres: `pg_locks`, deadlock logs.
- MySQL: `SHOW ENGINE INNODB STATUS`.

**Prevent**:
- Consistent lock order (always lock users before orders).
- Short transactions.
- Retry logic (exponential backoff).
- Optimistic locking where possible.

### 18. Optimistic vs. pessimistic locking — examples and when to use at scale.

**Optimistic**:
- `version` column: `UPDATE ... WHERE version = old_version`.
- Pros: No blocking, good for low-conflict.
- Example: Cart update.

**Pessimistic**:
- `SELECT ... FOR UPDATE`.
- Pros: Prevents conflicts.
- Cons: Blocks others.
- Example: Seat reservation.

**At scale**: Prefer optimistic → less contention.

### 19. Long-running transactions — problems and mitigations.

**Problems**:
- Block vacuum → bloat.
- Hold xmin horizon → replication lag.
- Lock contention.

**Mitigations**:
- Batch in small tx.
- Use advisory locks.
- Avoid idle-in-transaction.
- Monitor `pg_stat_activity`.

### 20. Vertical vs. horizontal scaling for SQL — limits and when to switch.

**Vertical**:
- Add CPU/RAM/disk.
- Limit: ~128 cores, 10–20 TB RAM, ~50k TPS writes.

**Horizontal**:
- Sharding/replication.
- Switch when: Single node bottleneck (I/O, CPU, connections).

**When**: >10k TPS writes or 1–10 TB data.

### 21. Streaming replication in Postgres vs. binlog replication in MySQL — setup, lag handling, failover.

**Postgres**:
- Physical (WAL) → byte-for-byte copy.
- Async/sync.
- Tools: Patroni for HA.

**MySQL**:
- Logical (binlog) → statement/row-based.
- GTID for failover.

**Lag handling**: Monitor `pg_stat_replication` or `SHOW SLAVE STATUS`.
**Failover**: Patroni auto, MHA/Orchestrator manual/auto.

### 22. How to implement read scaling with replicas? Handling replication lag?

**Setup**:
- Read replicas.
- Proxy: ProxySQL (MySQL), pgBouncer + pgbouncer-rr-patch (Postgres).
- App-level routing.

**Lag**:
- Monitor seconds_behind_master.
- Stale reads OK for non-critical → route with lag threshold.
- Strong consistency → read from master.

### 23. Sharding strategies: Range, hash, directory lookup — pros/cons.

**Range**:
- Pros: Range queries easy.
- Cons: Hotspots (e.g., recent users).

**Hash**:
- Pros: Even load.
- Cons: Re-sharding hard, no range queries.

**Directory**: Lookup table for shard.
- Pros: Flexible.
- Cons: Extra lookup.

**Best**: Hash for even, range for time-based.

### 24. What is application-level sharding vs. DB-native (Citus/Vitess)?

**App-level**:
- App routes queries (e.g., user_id % 10).
- Pros: Full control.
- Cons: Error-prone, no cross-shard joins.

**Native**:
- Transparent (Citus for Postgres, Vitess for MySQL).
- Pros: SQL works.
- Cons: Limited cross-shard.

### 25. Handling cross-shard transactions/queries at scale?

**Avoid**:
- Design to avoid (e.g., denormalize).
- Use eventual consistency (Kafka, sagas).

**If needed**:
- 2PC/XA (slow).
- App-level coordination.
- ETL for aggregates.

### 26. High availability setups — multi-AZ, multi-region, failover time?

**Multi-AZ**:
- Sync replication + witness → sub-30s failover (Patroni).

**Multi-region**:
- Async replication + DNS switch.
- Failover: Minutes (manual) or auto with tools.

**MySQL Group Replication**: Multi-master, auto-failover <10s.

### 27. Key metrics to monitor for SQL DB health at scale?

- QPS/TPS
- Replication lag
- Connections (idle/active)
- Cache hit ratio (>99%)
- Vacuum stats (dead tuples)
- Slow queries (pg_stat_statements)
- Disk I/O, CPU, memory
- Lock waits, deadlocks

Tools: Prometheus + Grafana, pgBadger.

### 28. How to handle database incidents (e.g., replication stopped, out of disk)?

**Replication stopped**:
- Check logs.
- Restart replica.
- Re-sync if needed.

**Out of disk**:
- Promote replica.
- Add storage.
- Clean WAL/archived files.

**Runbooks**:
- Automated failover scripts.
- Alerts on lag >30s, disk <10%.

### 29. Backups and PITR strategy for petabyte-scale DB?

**Physical backups**:
- pg_basebackup + WAL archiving (Postgres).
- xtrabackup (MySQL).

**PITR**:
- WAL-G/Barman for Postgres → restore to any point.
- binlog + xtrabackup for MySQL.

**Strategy**:
- Daily full + continuous WAL.
- Offsite storage.
- Test restores quarterly.

### 30. When would you choose SQL over NoSQL (or vice versa) in a scaled backend?

**Choose SQL**:
- Strong ACID, complex joins, transactions.
- Schema stability, reporting.
- Examples: Finance, e-commerce orders.

**Choose NoSQL**:
- Extreme writes (millions/sec), flexible schema.
- Eventual consistency OK.
- Examples: Logs, feeds, user events.

**Hybrid**: SQL for core (transactions), NoSQL for feeds/search.

This covers the questions in depth with practical examples and trade-offs. Let me know if you'd like to expand on any specific one!