### Components: 
- **Hadoop Common**: This refers to the common utilities and libraries that provide essential support to the other Hadoop modules. These utilities are foundational and necessary for the operation of the entire Hadoop Ecosystem.
- **Hadoop Distributed File System (HDFS)**: A storage system that handles large datasets across low-spec hardware. It is designed to be scalable, allowing a single Hadoop cluster to expand to hundreds or thousands of nodes.
- **MapReduce**: The processing engine of Hadoop that divides large datasets into smaller chunks, processes them in parallel, and then aggregates the results.
- **YARN (Yet Another Resource Negotiator)**: Manages and allocates system resources (RAM, CPU) to different tasks, enabling Hadoop to perform various types of processing like batch, stream, interactive, and graph processing.

### Challenges and Drawbacks of Hadoop

- **Transaction Processing**: Hadoop struggles with transactional data processing due to its lack of random access capabilities.
	Reason:
	 - **HDFS is append-only** — files can be written once and read many times, but not updated in place. This design avoids file corruption in distributed systems but makes **random writes or updates impossible**
	- Hadoop’s **MapReduce** model processes entire datasets in batches — not individual records.
	- Lacks **transaction coordination** (no commit/rollback or locking mechanisms).


- **Sequential Dependencies**: Hadoop is inefficient when tasks cannot be parallelized or when data dependencies exist, such as when one record must be processed before another.
	- Hadoop’s **MapReduce** framework assumes all map tasks can run **independently in parallel**
	- Each task works on a separate chunk of data with **no shared state** between mappers or reducers.
	- If one record depends on the output of a previous one (e.g., sequential time steps), Hadoop must run multiple jobs in sequence — and each job reads/writes from HDFS, causing **high disk I/O** and latency.


- **Low Latency Access**: Hadoop is not suited for applications requiring real-time processing with minimal delays, such as online trading or gaming.
	- Hadoop’s **job startup time** (initializing mappers/reducers, allocating containers in YARN, scheduling, etc.) can take several seconds.
	- Data is stored on disk (HDFS) and processed in **batch mode**, not in memory.
	- The architecture involves **heavy I/O** between tasks and across nodes.
	- Designed for **throughput (large data scans)**, not for **latency (quick responses)**.



- **Small File Processing**: Hadoop is not optimized for handling large numbers of small files, although improvements like IBM’s Adaptive MapReduce are being developed.
	- In HDFS, the **NameNode** keeps metadata (location, permissions, block mapping) for every file **in memory**.
	- Each small file still requires a block and a metadata entry, causing **memory pressure** and **metadata lookup overhead**.
	- MapReduce jobs have to spawn one map task per file, so many small files create **too many small tasks**, wasting resources.
	- Hadoop’s block size (default 128 MB) is large — small files don’t fill a block, leading to **poor storage efficiency**.


- **Intensive Computations with Little Data**: Hadoop is inefficient for tasks involving heavy computations on small datasets.
	- Hadoop’s **job orchestration overhead** (starting JVMs, reading configurations, scheduling across the cluster) is significant.
	- When data is small, this setup cost outweighs actual computation time.
	- Data is always read/written from HDFS — even temporary intermediate results — causing **unnecessary disk I/O**.
	- Hadoop doesn’t exploit **in-memory caching** or CPU acceleration (SIMD, vectorization) effectively.


### MapReduce
I implemented a MapReduce framework from scratch and deployed it on k8s. Details: https://github.com/ledat1205/mapreduce_from_scratch
![[Pasted image 20251023141520.png]]

### Stages of the Hadoop Ecosystem 

![[Pasted image 20251023150630.png]]

#### Ingest Stage:
The ingest stage is the initial phase of Big Data processing, where data from various sources is collected and transferred into the Hadoop system.
**Tools:**
- **Flume:** A distributed service that collects, aggregates, and transfers large amounts of data to HDFS. Flume is particularly well-suited for streaming data and is known for its flexible and simple architecture.
- **Sqoop:** Sqoop is used for transferring bulk data between relational databases and Hadoop. It generates MapReduce code to efficiently import and export data, making it easier to integrate structured data into Hadoop.

#### Storage stage:
In this stage, the ingested data is stored in the Hadoop system for future processing.
**Tools:**
- [**HDFS**](https://hadoop.apache.org/docs/r1.2.1/hdfs_design.html)**:** The primary storage system in Hadoop, where data is distributed across a cluster of nodes.
- [**HBase**](https://hbase.apache.org/)**:** A column-oriented, non-relational database system that runs on top of HDFS. HBase provides real-time read/write access to large datasets and uses hash tables to store data for faster lookups.
- [**Cassandra**](https://medium.com/@thisis-Shitanshu/mastering-apache-cassandra-part-1-71257d54d70e)**:** A scalable NoSQL database designed to handle large amounts of data across many commodity servers with no single point of failure.

#### Process and Analyze Stage:
In this stage, the stored data is processed and analyzed to extract meaningful insights.
**Tools:**
- [**Pig**](https://pig.apache.org/)**:** A high-level platform that provides a procedural data flow language for analyzing large datasets. Pig is particularly useful for executing complex data transformations.
- [**Hive**](https://hive.apache.org/)**:** A data warehousing tool that allows for SQL-like querying of data stored in HDFS. Hive is designed for creating reports and conducting data analysis on large datasets.

#### Access Stage:
This final stage involves accessing and interacting with the processed and analyzed data.
**Tools:**
- [**Impala**](https://impala.apache.org/overview.html)**:** A scalable SQL query engine that allows users to perform low-latency, real-time queries on data stored in HDFS without needing advanced programming skills.
- [**Hue**](https://gethue.com/)**:** An acronym for Hadoop User Experience, Hue is a web-based interface that allows users to interact with the Hadoop ecosystem. It provides tools for browsing, querying, and managing data, including SQL editors for Hive and other query languages.

### HDFS
Core Concepts:
- HDFS splits large files into **smaller blocks**, **replicates** them, and **distributes** them across multiple machines.
- Enables **streaming data access** — continuous, high-throughput data transfer instead of bursty reads/writes.

Key Features
- Cost Efficiency: Runs on commodity (low-cost) hardware, reducing overall storage expenses.
* Scalability:
	- Large Data Handling: Can manage petabytes to exabytes of data.
	- Cluster Scalability: Supports hundreds to thousands of nodes, allowing seamless expansion.
- Fault Tolerance:
	- Data Replication: Multiple replicas ensure data availability even during node failures.
	- Rack Awareness: Replicas stored on different racks to survive rack-level failures.
- Portability:
	- Easily portable across platforms, integrates with different environments without major reconfiguration.
- Write Once, Read Many (WORM):
	- Files are immutable after being written, but can be read or appended multiple times.
	- Ensures data integrity and simplified concurrency control.

![[Pasted image 20251023153114.png]]

Nodes: a **node** is a physical or virtual machine that participates in storing or managing data.
2 main types:
- NameNode (master / primary node): manages metadata and file system namespace.
	- Acts as the **brain** of HDFS.
	- Keeps the **metadata** — information about files, directories, block locations, permissions, etc.
	- Manages **file operations** (open, close, rename) and coordinates how blocks are distributed to DataNodes.
	- There’s typically **one active NameNode** per cluster (sometimes with a standby for high availability).
	- It **does not store actual data**, only the metadata.
- DataNode (worker / secondary node): stores actual file blocks and executes I/O.
	- Stores the **actual data blocks**.
	- Responds to **read/write requests** from clients based on NameNode instructions.
	- Sends regular **heartbeat and block reports** to the NameNode to confirm health and data status.
	- A cluster can have **hundreds or thousands** of DataNodes.

Blocks:
- In HDFS, data is **split into fixed-size blocks** before storage.
- The **default block size** is **64 MB or 128 MB**, but it can be configured.
- Large files are divided into blocks for **parallel storage and processing** across multiple nodes.
    - Example: A 500 MB file with 128 MB blocks → 4 blocks (3×128MB + 1×116MB).
- The NameNode keeps track of **which blocks belong to which file** and **where each block is stored**.

Rack Awareness:
- A **rack** is a group of DataNodes connected via the same **network switch** (typically 40–50 nodes).
- **Rack Awareness** means HDFS knows which nodes belong to which rack.
- It uses this information to:
    - **Reduce network traffic:** by placing replicas on the same or nearby racks when possible.
    - **Improve fault tolerance:** by storing copies of data on **different racks**, so if one rack fails, data remains accessible elsewhere.

Replication: 
- HDFS ensures **data reliability** by storing multiple copies (replicas) of each block.
- The **Replication Factor** (usually 3 by default) defines how many copies are made.
- Replicas are stored on **different DataNodes** and **across racks** to prevent data loss.
- Even if one node or rack fails, other copies keep the data available
- **Rack-Aware Replication Policy:**
	- One replica on the same rack for performance.
	- Other replicas on different racks for fault tolerance.

Read Operation in HDFS
1. The client sends a request to the Name Node to get the location of the Data Nodes that contain the required data blocks.
2. The Name Node verifies the client’s permissions and provides the locations.
3. The client then reads the data from the closest Data Nodes, ensuring efficient data retrieval.

 Write Operation in HDFS
1. The client sends a request to the Name Node to check if the file already exists. If it does, the client receives an IO exception.
2. If the file does not exist, the Name Node grants write permission and provides the locations of the Data Nodes where the data will be stored.
3. The Data Nodes then create replicas of the data and confirm the successful write operation back to the client.
. 
 Note
- **Metadata Operations**: When a client needs to access data, it first interacts with the Name Node to get metadata information, such as where the data blocks are located.
- **Block Operations**: The Name Node manages the distribution of data blocks across the Data Nodes. These operations involve the placement and movement of data blocks.

### Hive

**Apache Hive** is a **data warehouse system** built on top of **Hadoop**.  
It allows users to **query and analyze large datasets** stored in **HDFS** (Hadoop Distributed File System) using a **SQL-like language** called **HiveQL**.

Hive translates these SQL queries into **MapReduce**, **Tez**, or **Spark** jobs for distributed execution — making it ideal for **batch processing** of big data.


**Architecture Overview**
![[Pasted image 20251024112958.png]]
**1. Hive Client**:
Provides drivers like JDBC and ODBC for communication with different types of applications. These clients interact with Hive services to execute queries.
- **JDBC and ODBC Clients**: These allow different applications (Java-based or ODBC-compliant) to connect to Hive.

**2. Hive Services**:
Includes the command-line interface and the server that handles query execution. The driver within Hive services processes query statements and manages the session lifecycle. The meta store in this section stores metadata related to tables, such as schema and data location.

- **Hive Server**: Manages query execution and supports multiple clients submitting requests simultaneously.
- **Driver**: Receives queries, initiates sessions, and sends queries to the compiler.
- **Optimizer**: Performs transformations on the execution plan to improve efficiency and speed.
- **Executor**: Carries out the tasks after optimization.
- **Meta Store**: Central storage for metadata, maintaining information about tables, schemas, and data locations.

**3. Hive Storage and Computing**:
This component handles the actual storage of data in the Hadoop cluster or HDFS and the metadata in a dedicated database. The processing of tasks is done through MapReduce or similar frameworks.

**Workflow (Query Execution Flow)**
1. **User submits** a HiveQL query (e.g., `SELECT * FROM sales WHERE region='APAC';`)
2. **Driver** receives the query → sends it to the **Compiler**.
3. **Compiler** parses and validates syntax → interacts with **Metastore** to get schema.
4. **Query plan** is generated and optimized → converted to **MapReduce/Tez/Spark jobs**.
5. **Execution Engine** runs the jobs across the Hadoop cluster.
6. **Results** are collected and returned to the client.

Advantages of Apache Hive

| Advantage                     | Description                                                            |
| ----------------------------- | ---------------------------------------------------------------------- |
| **SQL-like Interface**        | Familiar HiveQL syntax for users from SQL background.                  |
| **Handles Huge Data**         | Efficiently queries terabytes or petabytes in HDFS.                    |
| **Scalable & Fault-tolerant** | Built on top of Hadoop’s distributed framework.                        |
| **Extensible**                | Supports UDFs (User-Defined Functions) and integration with Tez/Spark. |
| **Data Warehouse Capability** | Supports partitioning, bucketing, and schema evolution.                |

Limitations of Apache Hive

| Limitation                                          | Description                                                     |
| --------------------------------------------------- | --------------------------------------------------------------- |
| **High Latency**                                    | Designed for batch processing — **not real-time** queries.      |
| **No Transactional Consistency (earlier versions)** | Limited ACID support; suitable for analytical workloads only.   |
| **Schema on Read**                                  | Flexible but can lead to performance issues if poorly designed. |
| **Limited Indexing**                                | Not as efficient as traditional RDBMS indexing.                 |
|                                                     |                                                                 |
Note: about schema on read and schema on write.

![[Pasted image 20251024113037.png]]