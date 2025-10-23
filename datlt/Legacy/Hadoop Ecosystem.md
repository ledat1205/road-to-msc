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