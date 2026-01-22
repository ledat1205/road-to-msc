serverless sql database
storage and analytics

Access:
- Cloud console
- bq command line 
- Client libs

Resource Hierachy
![[Pasted image 20260121032332.png]]

### importing data

**Web UI**:
- Local update, GCS, Drive, BigTable, Azure Blob, AWS s3 
- file types:
	- JSONL
	- CSV
	- Parquet
	- Avro
	- ORC
- Adjust schema and table setting during the import

**bq command line**:
faster and large data operations

**BigQuery Data Transfer Service**
![[Pasted image 20260121032809.png]]
Schedule transfer

BigQuery Streaming API and Storage write API
![[Pasted image 20260121033019.png]]

![[Pasted image 20260121033033.png]]

![[Pasted image 20260121033308.png]]

### Spark-Bigquery Connector
![[Pasted image 20260121033410.png]]

### External Tables and Federated Queries
external table is table that data is stored on GCS and user Bigquery to query on that data (without move data to bigquery)
![[Pasted image 20260121033649.png]]

![[Pasted image 20260121033807.png]]

![[Pasted image 20260121033916.png]]

### Dataset config
Region and Multi Region
![[Pasted image 20260121034151.png]]

### Data encryption
![[Pasted image 20260121034302.png]]

### Backing up
![[Pasted image 20260121034624.png]]

### referential integrity
the principle foreign key values must always correspond to valid entries in the parent table, ensure data consistency

### ### Normalization and Denormalization
Normalization:
- when data integrity and update efficiency are more important than query speed
- system where maintaining consistent, accurate data is a priority

Denormalization:
- query performance is primary concern 
- read-heavy systems with large datasets where JOINs are a bottleneck

### Partitioning
![[Pasted image 20260122110842.png]]

Can set partition expiration: allow to automatically delete partitions once they reach the expiration age

### Advanced Query Management
**Standard Views**: like a view in other database, store query and run when query the view
**Materialize Views**: pre-compute data
**Authorized Views**: security mechanism that allow user query on specific data

**User-Defined Function**: write custom function using JS or SQL and call in your query

### Storage Types
![[Pasted image 20260122140531.png]]

![[Pasted image 20260122140638.png]]

### Compute cost

Query cost:
- charge for bytes read during query execution
- also charge for data storage (active, long-term) but separate from running query
Estimate query size 
- dry run using bq command line tool
- UI preview

BigQuery slots: resource to execute query, Google fully management, dynamic scale
![[Pasted image 20260122172853.png]]

Pricing Models
![[Pasted image 20260122173152.png]]

Capacity-based pricing
![[Pasted image 20260122173435.png]]

Metric to monitor
![[Pasted image 20260122173537.png]]

Control Query Quota to control cost
![[Pasted image 20260122173722.png]]

![[Pasted image 20260122173854.png]]

