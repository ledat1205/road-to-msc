![[Pasted image 20251223004213.png]]

![[Pasted image 20251223004805.png]]

![[Pasted image 20251223004843.png]]

![[Pasted image 20251223005114.png]]![[Pasted image 20251223005157.png]]
![[Pasted image 20251223005541.png]]
![[Pasted image 20251223005738.png]]

![[Pasted image 20251223010358.png]]

Dataproc: require cluster
Dataflow: serverless

![[Pasted image 20251223025845.png]]

# Dataflow 
- java or python
- open-source api can also be executed on Flink, Spark, etc
- parallel task are autoscaled
- real-time and batch 

operations: 
- ParDo: Allows for parallel processing
- Map: 1:1 relationship between input and output in Python
- FlatMap: For non 1:1 relationships, usually with generator in Python
- .apply(ParDo): Java for Map and FlatMap
- GroupBy: Shuffle
- GroupByKey: Explicitly shuffle
- Combine: Aggregate values

template workflow supports non-developer users

# BigQuery

- Near real-time analysis of massive datasets
- NoOps 
- Pay for use
- inexpensive; charged on amount of data processed
- durable
- immutable audit logs
- mashing up different datasets
- interactive analysis of pertabyte-scale databases 
- familiar sql
- many ways to ingest, transform, load and export data
- nested and repeated fields, user-defined functions in JS
- structured data

Warehouse solution
Governance in many levels: table, column, row

BigQuery Solutions 
![[Pasted image 20251223110444.png]]

Dataproc solutions
![[Pasted image 20251223110515.png]]

Dataflow solutions
![[Pasted image 20251223110600.png]]

**Design Data Processing Infrastructure**
- Data ingest solutions: CLI, web UI, or API![[Pasted image 20251223111129.png]]
	- 3 ways of loading data into BigQuery:
		- files in disk, Cloud Storage or Filestore
		- stream data
		- federated data source

Noted:
- transferring data from on-premises location use gsutill
- from another cloud provider iuse Storage Transfer Service
- otherwise, evaluate both tools with respect to your specific scenario

# Pub/Sub
- NoOps, serverless global message queue
- Asynchronous; publisher never waits, a subscriber can get the message now or any time within 7

