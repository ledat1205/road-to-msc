#### MySQL (with InnoDB Engine)

MySQL is a relational database primarily used for OLTP (Online Transaction Processing) workloads like web applications. Its default InnoDB engine emphasizes reliability and concurrency.

- **Storage Organization**: InnoDB uses clustered indexes based on primary keys, where data is stored in B-tree structures. This minimizes I/O for primary key lookups. Data is organized in tablespaces (system, undo for rollbacks, temporary), supporting file-per-table for isolation. It includes compression, foreign keys for integrity, and row-based storage, making it efficient for row-oriented access but less so for analytics.
- **Query Executor**: Employs row-level locking and MVCC (Multi-Version Concurrency Control) for consistent reads without blocking. Queries are optimized for primary key access, with support for full-text search and geospatial indexes. It uses an adaptive hash index for faster reads.
- **Throughput**: High for read-heavy OLTP via in-memory buffer pools (caching data/indexes), change buffers (batching inserts), and adaptive hashing. However, it's optimized for simpler queries; complex analytics can bottleneck due to row-oriented scanning.
- **Scalability**: Supports replication (master-slave) for read scaling and up to 64TB storage. High concurrency via MVCC and row-locking, but vertical scaling is limited; horizontal via sharding or clustering tools like Vitess.

#### PostgreSQL

PostgreSQL is an object-relational database, excelling in complex queries, data integrity, and mixed OLTP/OLAP workloads. It's more feature-rich than MySQL.

- **Storage Organization**: Heap-based storage (non-clustered), where tables are stored as unordered heaps with separate indexes (B-tree by default). Uses tablespaces for distributing objects across filesystems, supporting partitioning for large tables. It handles diverse data types (e.g., JSON, geospatial) and is row-oriented, but with extensions for columnar via add-ons like cstore_fdw.
- **Query Executor**: Process-per-connection model for better isolation. Advanced query optimizer with parallel execution, indexing techniques (e.g., GiST, GIN for complex types), and MVCC without read-write locks, allowing high concurrency for writes. Supports window functions, CTEs, and extensions for custom logic.
- **Throughput**: Better for write-heavy and complex queries due to MVCC and partitioning. Handles large datasets with parallel queries, but may be slower for simple reads compared to MySQL. Extensions like TimescaleDB boost time-series performance.
- **Scalability**: Horizontal via partitioning, sharding (e.g., Citus extension), and replication. Log-shipping for reliability, table partitioning for performance. Better for growing datasets than MySQL, with seamless handling of concurrency.

Key differences from MySQL: PostgreSQL is ACID-compliant across all features, better for writes and complex queries, while MySQL (InnoDB) is faster for reads and simpler setups but requires specific engines for ACID.

#### Apache Druid

Druid is a columnar database optimized for real-time analytics and time-series data, like logs or events.

- **Storage Organization**: Columnar format with data segmented by time and dimensions, stored in immutable segments. Uses deep storage (e.g., S3, HDFS) for persistence, with compression and indexing (inverted indexes) for fast scans. Data is pre-aggregated during ingestion.
- **Query Executor**: Distributed query layer with Brokers routing to Historical nodes (holding segments). Supports SQL-like queries with aggregations, joins (limited), and approximate algorithms. Pre-fetching and multi-level indexing enable sub-second responses.
- **Throughput**: High for aggregations on large datasets (billions of rows/sec), but struggles with complex joins or updates. Handles high concurrency via shared-nothing architecture.
- **Scalability**: Horizontal, cloud-friendly with independent services (e.g., add Historical nodes for data, Brokers for queries). Uses ZooKeeper for coordination. Linear scaling, but complex topology (multi-server clusters).

Differences from MySQL/PostgreSQL: Columnar vs. row-oriented; optimized for OLAP analytics with pre-aggregation, not transactions. Less flexible for ad-hoc queries, but faster for scans.

#### ClickHouse

ClickHouse is a columnar OLAP database for fast analytics on large datasets.

- **Storage Organization**: Column-wise storage, reading only needed columns to reduce I/O. High compression, with parts merged in background. Supports materialized views for pre-computation.
- **Query Executor**: Leverages columnar storage for vectorized execution of filters, aggregations, and joins (adaptive: hash for small, merge for large). SQL-compatible with sampling and approximate functions for speed.
- **Throughput**: Extremely high (e.g., >1B rows/sec) for analytical queries due to minimal data transfer and optimizations.
- **Scalability**: Distributed with asynchronous multi-master replication for redundancy. Scales horizontally across nodes.

Differences from Druid: More general OLAP focus with dynamic processing; Druid emphasizes time-series and pre-aggregation.

#### ScyllaDB / Cassandra

Both are wide-column NoSQL databases for distributed, high-availability workloads. ScyllaDB is a performance-optimized rewrite of Cassandra.

- **Storage Organization** (Cassandra): Data distributed via hash partitioning in a ring topology. Wide-column model with partition keys; stored in SSTables (immutable files) with MemTables for writes. Replication across nodes/data centers.
- **Query Executor** (Cassandra): Any node as coordinator, using gossip for discovery. CQL queries (SQL-like) with consistency levels (e.g., QUORUM). Scans, gets, but limited joins.
- **Throughput** (Cassandra): High for reads/writes via load balancing; tunable consistency affects it.
- **Scalability** (Cassandra): Linear horizontal via adding nodes; vNodes for even distribution.
- ScyllaDB: Similar data model/CQL, but shard-per-core architecture (shared-nothing per CPU core) eliminates Java GC pauses, maximizes utilization. Storage in SSTables, but optimized I/O schedulers.
- Differences/Optimizations in ScyllaDB: 2-5x higher throughput, lower latencies, faster scaling (e.g., 7x faster bootstrap). Shard-aware drivers direct queries efficiently. Better for massive OPS without tuning.

#### Google BigQuery

BigQuery is a serverless, columnar analytics database for big data.

- **Storage Organization**: Columnar (Capacitor format) with separation from compute; data replicated across zones. Supports ACID, ingestion via batch/streaming.
- **Query Executor**: Massively parallel distributed engine; ANSI SQL with ML integrations. Dynamic resource allocation.
- **Throughput**: Seconds for TBs, minutes for PBs; high concurrency without provisioning.
- **Scalability**: Serverless auto-scaling; no manual management.

Differences from ClickHouse: Cloud-native, no resource contention; ClickHouse is on-premise with integrated storage/compute.

#### Apache HBase

HBase is a column-family NoSQL database on Hadoop for massive, sparse data.

- **Storage Organization**: Column-oriented with regions (horizontal partitions) split by row keys. Each region has stores per column family: MemStore (in-memory) + HFiles (immutable on HDFS). WAL for durability.
- **Query Executor**: Clients query via gets/puts/scans; RegionServers handle with scanners merging MemStore/HFiles. Filters and coprocessors for server-side logic.
- **Throughput**: Optimized with caches (block/off-heap), compactions, bulk loads. High for random access.
- **Scalability**: Horizontal via region splitting/balancing; integrates with HDFS/YARN for massive scale.

### Comparison Table

| Database   | Type                | Storage Organization                                         | Query Executor                                                | Throughput Considerations                             | Scalability                                   | Key Use Cases & Differences                         |
| ---------- | ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------- | ----------------------------------------------------- | --------------------------------------------- | --------------------------------------------------- |
| MySQL      | Relational (OLTP)   | Row-oriented, clustered B-tree indexes, tablespaces          | Row-level locking, MVCC, hash indexes                         | High for reads; buffer pools for caching              | Replication, up to 64TB; high concurrency     | Web apps; simpler than PostgreSQL for reads         |
| PostgreSQL | Object-Relational   | Row-oriented, heap-based with separate indexes, partitioning | Process-per-connection, parallel optimizer, MVCC              | Better for writes/complex queries; parallel execution | Horizontal sharding/partitioning, replication | Analytics/mixed workloads; more features than MySQL |
| Druid      | Columnar OLAP       | Columnar segments, time-partitioned, deep storage            | Distributed brokers/historicals, SQL-like with approximations | Sub-second for aggregations; high concurrency         | Horizontal services, linear                   | Time-series; pre-agg vs. relational ad-hoc          |
| ClickHouse | Columnar OLAP       | Column-wise, high compression, merges                        | Vectorized, adaptive joins, sampling                          | >1B rows/sec for analytics                            | Multi-master replication                      | General OLAP; dynamic vs. Druid's time-focus        |
| Cassandra  | Wide-Column NoSQL   | Partitioned SSTables, MemTables, ring topology               | Coordinator nodes, CQL, consistency levels                    | Tunable, high for distributed reads/writes            | Linear horizontal, vNodes                     | Distributed apps; base for ScyllaDB                 |
| ScyllaDB   | Wide-Column NoSQL   | Similar to Cassandra, but shard-per-core SSTables            | Similar to Cassandra, shard-aware drivers                     | 2-5x Cassandra; low latency                           | Faster scaling (7x+), linear                  | High-perf Cassandra alternative; no GC              |
| BigQuery   | Columnar Serverless | Columnar, separated from compute, replicated                 | Parallel distributed SQL engine                               | Fast on massive data; auto-optimized                  | Serverless auto-scaling                       | Big data analytics; cloud vs. on-prem               |
| HBase      | Column-Family NoSQL | Regions with MemStore/HFiles on HDFS                         | Scans/gets with filters/coprocessors                          | High random access; caches/compactions                | Horizontal region splitting                   |                                                     |