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


- **Intensive Computations with Little Data**: Hadoop is inefficient for tasks involving heavy computations on small datasets.