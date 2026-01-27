### 1. Overview & Architectural Comparison

These databases fall into different categories:

- **Relational OLTP** → MySQL (InnoDB), PostgreSQL (heap-based)
- **Columnar OLAP / Analytics** → Druid (segment-based), ClickHouse (MergeTree), BigQuery (Capacitor)
- **Distributed Wide-Column / Key-Value NoSQL** → ScyllaDB/Cassandra (LSM-tree), HBase (LSM on HDFS)

| Database             | Type                | Storage Paradigm               | Key Storage Engine / Format                     | Write Strategy                         | Read Optimization                     | Concurrency & Scalability              | Primary Use Case                    |
| -------------------- | ------------------- | ------------------------------ | ----------------------------------------------- | -------------------------------------- | ------------------------------------- | -------------------------------------- | ----------------------------------- |
| MySQL                | Relational OLTP     | Row-oriented, clustered B+tree | InnoDB (default)                                | WAL → buffer pool → change buffer      | Clustered PK, buffer pool caching     | MVCC row-locking, replication/sharding | Transactional apps, web services    |
| PostgreSQL           | Object-Relational   | Row-oriented heap              | Heap + heapam (default)                         | Append-only MVCC → WAL                 | Indexes → heap via TID, parallel exec | True MVCC, extensions (Citus)          | Complex queries, mixed OLTP/OLAP    |
| Druid                | Columnar OLAP       | Immutable columnar segments    | Segment-based (time-partitioned)                | Ingest → roll-up → immutable seg       | Column prune + bitmap/inverted idx    | Immutable, distributed services        | Time-series aggregations, real-time |
| ClickHouse           | Columnar OLAP       | Columnar LSM (parts)           | MergeTree family                                | Insert → small part → background merge | Sparse PK index + marks/granules      | Background merges, multi-master repl   | High-speed OLAP, analytics          |
| ScyllaDB / Cassandra | Wide-Column NoSQL   | LSM-tree wide-column           | SSTables + Memtables (shard-per-core in Scylla) | Commit log → Memtable → SSTable        | Bloom + cache + multi-SSTable merge   | Shard-per-core (Scylla), linear scale  | High-throughput distributed KV      |
| BigQuery             | Columnar Serverless | Columnar (nested/repeated)     | Capacitor (proprietary)                         | Ingest → Capacitor → Colossus          | Columnar prune + compression          | Serverless auto-scaling                | Petabyte-scale analytics            |
| HBase                | Column-Family NoSQL | Column-family LSM on HDFS      | HFiles + MemStore                               | WAL → MemStore → HFile                 | Block cache + bloom + multi-HFile     | Region splitting, Hadoop-integrated    | Sparse big data, random access      |

### 2. Deep Dive: Storage Engines

**MySQL (InnoDB)**

- Row-oriented with clustered primary key index (data stored in B+tree leaves).
- On-disk: Tablespaces (.ibd files), redo/undo logs, doublewrite buffer.
- In-memory: Buffer pool (caches data/indexes), change buffer (defers secondary index updates), adaptive hash index.
- Write path: WAL-first → buffer pool → background flush.
- Read path: B+tree traversal; secondary indexes → PK lookup.
- Concurrency: MVCC via undo + row locks.
- Maintenance: Purge old versions, adaptive flushing.

**PostgreSQL (Heap-based)**

- Unordered heap files (8KB pages), separate secondary indexes (TID pointers).
- On-disk: Heap files, visibility map, FSM.
- In-memory: Shared buffers (no dedicated change buffer).
- Write path: Append new tuple versions (MVCC) → WAL.
- Read path: Index → TID → heap fetch.
- Concurrency: True MVCC (readers never block writers).
- Maintenance: Autovacuum to reclaim space and freeze tuples.

**Druid**

- Immutable, time-partitioned columnar segments with dictionary encoding, bitmap indexes.
- On-disk: Deep storage (S3/HDFS/local).
- In-memory: Memory-mapped segments on Historicals.
- Write path: Ingestion → pre-aggregation → publish immutable segment.
- Read path: Broker → Historicals scan needed columns + indexes.
- Maintenance: Background compaction merges small segments.

**ClickHouse (MergeTree family)**

- Strictly columnar per-column files, parts sorted by primary key (ORDER BY).
- On-disk: Parts (.bin data, .mrk marks, .idx primary index), high compression.
- In-memory: Insert buffers.
- Write path: Inserts → small parts → background LSM-style merges.
- Read path: Sparse PK index → granule skipping → vectorized column reads.
- Maintenance: Continuous background merges, TTL, deduplication variants.

**ScyllaDB / Cassandra**

- LSM-tree: SSTables (immutable), Memtables.
- ScyllaDB: Shard-per-core (shared-nothing per CPU core), O_DIRECT I/O.
- On-disk: SSTables + commit log.
- In-memory: Memtables per shard, row cache, bloom filters.
- Write path: Commit log → Memtable → flush to SSTable.
- Read path: Memtable + bloom → index → merge multiple SSTables.
- Maintenance: Compaction (Leveled/TimeWindow best for most).

**BigQuery (Capacitor)**

- Columnar format supporting nested/repeated fields.
- On-disk: Colossus FS, high compression (RLE, dictionary).
- Write path: Batch/streaming ingest → Capacitor files.
- Read path: Dremel tree → scan compressed columns only.
- Maintenance: Fully managed (automatic).

**HBase**

- Column-family LSM: HFiles on HDFS per store (column family).
- On-disk: HFiles, WAL.
- In-memory: MemStore per region/column family, block cache.
- Write path: WAL → MemStore → flush to HFile.
- Read path: Merge MemStore + HFiles (bloom/block cache).
- Maintenance: Minor/major compactions, region splitting.

### 3. Performance Tuning: Industry Best Practices (2025–2026)

**MySQL (InnoDB)**

- innodb_buffer_pool_size: 60–80% RAM (dedicated server).
- innodb_log_file_size: 1–4GB for high writes.
- innodb_flush_log_at_trx_commit=2 (performance) vs. 1 (durability).
- innodb_io_capacity: Match SSD IOPS.
- innodb_file_per_table=ON, disable query cache (MySQL 8+).
- Monitor: SHOW ENGINE INNODB STATUS, Percona Toolkit.
- Recent insights (2025–2026): MySQL 9.5/Percona excels in stability/scalability; use innodb_dedicated_server=ON for auto-tuning.
- Resources: Percona "MySQL 101 Parameters" (2025 updates), Releem guide.

**PostgreSQL**

- shared_buffers: 25–40% RAM.
- work_mem / maintenance_work_mem: 4–64MB (per-op caution).
- effective_cache_size: 50–75% RAM.
- WAL: max_wal_size larger, checkpoint_completion_target=0.9.
- Autovacuum: Aggressive for high-update tables.
- Parallelism: Enable max_parallel_workers.
- Monitor: pg_stat_statements, EXPLAIN (ANALYZE, BUFFERS).
- Recent insights (2025): Use pgBadger, timescaledb-tune for time-series; focus on parallel workers and history-based optimizations.
- Resources: Mydbops "PostgreSQL Parameter Tuning 2025", Percona tuning guide.

**ClickHouse**

- Primary/ORDER BY: Low-cardinality first for skipping.
- index_granularity: Lower for point queries.
- Merges: Tune concurrency ratio, use projections/materialized views.
- Inserts: Batch large, async.
- Monitor: system.parts, query logs, EXPLAIN.
- Recent insights (2026): Definitive guide emphasizes "read less data" via ORDER BY design (up to 100× gains); projections for 20×+ speedups.
- Resources: ClickHouse "Definitive Guide to Query Optimization (2026)", official query optimization docs.

**Druid**

- Segment size: 500MB–1GB (tune segmentGranularity).
- Compaction: Always-on to merge small segments.
- Historicals: Heap ~0.5GiB × cores; druid.processing.numThreads = cores–1.
- Queries: Approximations, caching; increase replicas/numThreads for concurrency.
- Resources: Official "Basic Cluster Tuning", Imply tuning guides.

**ScyllaDB / Cassandra**

- Compaction: Leveled (read-heavy), TimeWindow (time-series).
- Schema: Even partition keys, avoid large partitions.
- Drivers: Shard-aware (Scylla).
- Scylla-specific: Monitor shard balance; auto-tunes heavily.
- Recent insights (2025): Production Readiness Guidelines; compaction strategies table for workloads.
- Resources: ScyllaDB "Tips and Tricks for Maximizing Performance", Production Readiness docs.

**BigQuery**

- Minimize scanned bytes: Project columns, filter early, avoid SELECT *.
- Partition/clustering: Date + high-filter columns.
- Materialized views, BI Engine caching.
- Use APPROX functions, history-based optimizations (2025+).
- Monitor: Query explanation, bytes processed.
- Resources: Google Cloud "Optimizing Query Performance", "Optimize Query Computation".

**HBase**

- Heap: 16–36GB (G1GC preferred).
- hfile.block.cache.size: 40–60%.
- Regions/RS: 50–200, pre-split tables.
- Compaction: FIFO/Exploring strategies.
- Avoid hotspots: Salting keys, load balancing.
- Resources: Apache/Cloudera tuning guides, recent Medium series on hotspots/compaction.

Notes
### Storage & Data Organization

- **Clustered Index** — Primary key index where table data rows are physically stored in index order (InnoDB default; enables fast PK lookups but can cause page splits on inserts).
- **Heap Storage** — Unordered row storage where tuples are appended without sort order; secondary indexes point via TID (Tuple ID) — core of PostgreSQL's default access method.
- **Columnar Storage** — Data stored by column rather than row; enables column pruning, vectorized execution, and high compression (ClickHouse, BigQuery Capacitor, Druid segments).
- **Log-Structured Merge-Tree (LSM-Tree)** — Write-optimized structure using leveled or sized-tiered immutable files (SSTables/HFiles); background compaction/merges — foundation of Cassandra/ScyllaDB, HBase, ClickHouse MergeTree.
- **SSTable** — Immutable sorted string table file in LSM-trees; contains sorted key-value pairs + indexes + bloom filters (Cassandra, ScyllaDB, HBase).
- **MemTable** — In-memory write buffer that flushes to SSTables when full (LSM systems); per-shard in ScyllaDB for contention-free writes.
- **Parts** — Immutable on-disk directories in ClickHouse MergeTree containing columnar .bin files, sparse primary indexes (.idx), and marks (.mrk) for granule skipping.
- **Segment** — Time-partitioned, immutable columnar chunk in Druid; published to deep storage with bitmap/inverted indexes.
- **Capacitor Format** — BigQuery's proprietary columnar storage supporting nested/repeated fields with advanced compression (dictionary, RLE, frame-of-reference).
- **HFile** — Immutable sorted file per column family in HBase; stored on HDFS with block-level indexes.

### Write Paths & Durability

- **Write-Ahead Logging (WAL)** — Append-only log of changes written before data pages; ensures crash recovery (InnoDB redo logs, PostgreSQL WAL, HBase WAL, Cassandra commit log).
- **Change Buffer** — InnoDB optimization that defers secondary index updates to batch them later, reducing random I/O on writes.
- **Doublewrite Buffer** — InnoDB mechanism to prevent torn pages during page flushes by writing pages twice (once to buffer, once to data).
- **Background Merges / Compaction** — LSM process that combines small immutable files into larger ones (ClickHouse merges, Cassandra/ScyllaDB compaction strategies: SizeTiered, Leveled, TimeWindow).
- **MemStore Flush** — HBase/ScyllaDB/Cassandra action of spilling in-memory writes to on-disk files.

### Query Execution & Optimization

- **Vectorized Execution** — Processing data in batches (SIMD-friendly vectors) rather than row-by-row; dramatically faster for columnar scans (ClickHouse, modern BigQuery/Spanner columnar engine).
- **Sparse Primary Index** — ClickHouse's granule-based index (e.g., one entry per 8192 rows) that allows skipping large irrelevant ranges during scans.
- **Bitmap Index** — Compact per-value bitsets for fast filtering on low-cardinality columns (Druid segments, some ClickHouse use cases).
- **Inverted Index** — Term → document pointers; accelerates text/dimension filters (Druid).
- **Granule / Mark** — ClickHouse concept: smallest skip unit in primary index; marks point to byte offsets in column files.
- **Projections** — ClickHouse materialized sub-tables with different sort keys for query-specific optimization.
- **Dremel Tree** — BigQuery's distributed query execution architecture (tree-shaped dispatch for massive parallelism).

### Concurrency & Isolation

- **Multi-Version Concurrency Control (MVCC)** — Maintains multiple versions of rows to allow non-blocking reads during writes (PostgreSQL heap versions, InnoDB undo logs).
- **Visibility Map** — PostgreSQL bit array tracking which heap pages contain only visible tuples (reduces index-only scan I/O).
- **Shard-per-Core** — ScyllaDB's shared-nothing architecture assigning data shards to CPU cores; eliminates locks and GC pauses for extreme concurrency.
- **Tunable Consistency** — Cassandra/ScyllaDB levels (ONE, QUORUM, ALL) trading latency vs. consistency per operation.
- **TidStore** — PostgreSQL 17+ internal structure reducing vacuum memory usage and contention.

### Maintenance & Optimization

- **Autovacuum** — PostgreSQL background process reclaiming dead tuples and updating statistics (tunable aggressiveness critical for update-heavy tables).
- **Purge** — InnoDB cleanup of old MVCC versions from undo logs.
- **Bloom Filter** — Probabilistic structure to avoid unnecessary SSTable/HFile reads during point lookups (Cassandra, ScyllaDB, HBase).
- **Block Cache / Row Cache** — In-memory caching of decompressed blocks or rows (HBase block cache, ScyllaDB row cache).

### Other Advanced / Emerging Terms

- **Hybrid Transactional/Analytical Processing (HTAP)** — Systems supporting both OLTP and OLAP on same data (e.g., recent OceanBase columnar + LSM, Spanner columnar engine).
- **Frame-of-Reference Encoding** — Lightweight compression preserving order for vectorized ops (used in DuckDB, influences modern columnar engines).
- **SIMD-Accelerated Operations** — Use of CPU vector instructions for bulk processing (PostgreSQL 16+, ClickHouse vectorized engine).