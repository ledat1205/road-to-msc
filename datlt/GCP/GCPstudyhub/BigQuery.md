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
external table is table that data is stored on GCS and user Bigquery to query on that data (wi)

