Overview:
- Handle both batch and streaming
	- google managed version of Apache Beam
- auto scaling, serverless
- integrates with Cloud Storage, Pub/Sub, BigQuery
- connectors are available for BigTable and Kafka

### Core concepts 

**PCollection & Elements**
PColletion: distributed dataset
Elements: an entry (a row)

Distributed dataset: Elements spread across multiple nodes (like idea of hdfs). Each machine handles a different subset of the PCollection
![[Pasted image 20260113173835.png]]

