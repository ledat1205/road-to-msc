# Databases
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