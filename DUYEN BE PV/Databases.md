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