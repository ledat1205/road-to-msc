based on very simple  mechanism. write to append-log first, flush log to disk then apply change to real object (table,...)

WAL have offset in each event to track status (apply to obj or not)

# Write-Ahead log in PostgreSQL

Workflow:
1. **Log First**: PostgreSQL writes the intended changes as **WAL records** to an append-only log file before modifying any database files (heap or indexes). This includes granular details like the operation type (`INSERT`, `UPDATE`, or `DELETE`), the modified page, and before-and-after states if necessary. Each WAL entry is sequentially appended to the log file.

2. **Flush to Disk**: Before PostgreSQL marks the transaction as **committed**, it flushes the WAL to disk using `fsync`. This guarantees durability: even if the system crashes before applying the changes to the main database files, the WAL ensures the transaction can be recovered.

3. **Apply Changes Later**: PostgreSQL applies the changes to the data files asynchronously during a process called **checkpointing**. At each checkpoint, the dirty pages in memory (modified data) are written back to the storage to reduce reliance on WAL during recovery. If a crash happens between the WAL flush and the data file update, PostgreSQL can use the WAL to "redo" the operations that occurred after the last checkpoint.

The WAL is divided into **16 MB segment files**, which are stored in the `pg_wal` directory. PostgreSQL rotates and archives old WAL files to manage storage efficiently. Key components of the WAL include:

- **Redo Records**: Detailed operations describing changes to be replayed.
- **Transaction IDs**: Unique identifiers to group WAL entries belonging to the same transaction.
- **Commit Markers**: Flags indicating the end of a transaction.

PostgreSQL also supports **archiving WAL segments** for disaster recovery and **point-in-time recovery (PITR)**, where the database can be restored to a specific state by replaying archived WAL files.

## Replication

Leverages its WAL for 2 primary types of replication: Physical and Logical

### Physical replication
WAL segments are streamed directly from primary node to the replica nodes. This method is fast, efficient and idea for HA setups. However it requires the primary and replicas to have identical database structures and postgresql versions

Synchronous replication require replicas to acknowledge WAL entries before commited

### Logical replication

Logical replication decodes WAL records into logical change events like SQL-level operations 