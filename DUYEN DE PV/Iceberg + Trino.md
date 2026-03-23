## Hive vs Iceberg — Architecture Deep Dive

These are very different in philosophy. The actual data files (Parquet/ORC) are the same — what differs entirely is the **metadata layer**.

Here's the structural difference:

V

visualize

V

visualize show_widget

### Key structural differences

**Hive** tracks data at the folder/partition level in a central relational database (the HMS). Every Hive query depends on the Metastore to resolve where data lives and how it is structured. As the number of tables and partitions grew, Metastore lookups slowed down, and the service itself became a single point of failure. [Starburst](https://www.starburst.io/blog/hive-vs-iceberg/)

**Iceberg** tracks data at the individual file level using a metadata tree stored alongside the data itself. Instead of listing O(N) partitions in a table during job planning, Iceberg performs an O(1) RPC to read the snapshot. The file pruning and predicate pushdown can also be distributed to jobs, so the Hive Metastore is no longer a bottleneck. [Medium](https://medium.com/expedia-group-tech/a-short-introduction-to-apache-iceberg-d34f628b6799)

Iceberg can use any compatible metastore, but for Trino it supports the Hive metastore and AWS Glue. The catalog stores a pointer to `metadata.json`, while Iceberg's own metadata tree (manifest lists → manifest files → data files) lives in the object store itself. [Starburst](https://www.starburst.io/blog/iceberg-metadata/)

---

Now let's look at how each component behaves under different write scenarios:

V

visualize

V

visualize show_widget

### How the Iceberg metadata tree enables this

Iceberg has a persistent tree structure that manages various snapshots created for every mutation of the data. This enables a concurrency model that supports serializable isolation, and features like time travel across a linear progression of snapshots. The tree contains two types of Avro files: manifest lists and manifest files. Manifest list files contain pointers to manifest files, and manifest files themselves point to data files. [Starburst](https://www.starburst.io/blog/iceberg-metadata/)

Writers work in isolation, not affecting the live table, and perform a metadata swap only when the write is complete, making the changes in one atomic commit. [Medium](https://medium.com/expedia-group-tech/a-short-introduction-to-apache-iceberg-d34f628b6799)

### Iceberg Native Write (via Spark/Flink directly, bypassing Trino)

When writing directly via the Iceberg Java API (e.g. Spark or Flink), the flow is similar but bypasses Trino entirely. The engine writes data files and calls the Iceberg `AppendFiles` / `OverwriteFiles` API which commits directly to the catalog. Writers can operate concurrently using optimistic concurrency control, and readers never see partial updates. [Olake](https://olake.io/blog/apache-iceberg-hive-comparison/) If two writers conflict (same snapshot base), one will retry with a fresh snapshot.

### Summary comparison table

|Feature|Hive|Iceberg|
|---|---|---|
|Metadata location|Central HMS (relational DB)|Distributed: catalog → metadata.json → manifests → files|
|Partition tracking|Folder paths in HMS|File-level stats in manifest files|
|ACID|No (or heavy workarounds)|Yes, snapshot-based|
|Schema evolution|Fragile (rename/drop breaks things)|Safe in-place changes|
|Query planning cost|O(N) partition + file scans|O(1) snapshot read|
|Time travel|No|Yes (snapshot ID or timestamp)|
|Hidden partitioning|No — requires explicit partition columns in queries|Yes — users don't need to filter on partition columns|
|Concurrent writes|Unsafe|Safe (optimistic concurrency)|
|Trino write limit|`hive.max-partitions-per-writers`|`iceberg.max-partitions-per-writer` (same default: 100)|