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

**PTransform**: transform pipeline
**ParDo**: transform applied to individual element of a PCollection
**DoFn**: custom logic transform
**GroupByKey (PTransform)**: 
![[Pasted image 20260114035046.png]]

**CoGroupByKey (PTransform)**
![[Pasted image 20260114035200.png]]

**Flatten**
Merge multiple PCollections of same type to a single PCollection

### Dataflow window

**Tumbling windows / "fixed" windows**: 
- Fixed duration 
- Non-overlapping
- Sequential
![[Pasted image 20260114215242.png]]

**Hopping windows / "sliding" windows**:
* Fixed duration
* Overlapping
* Fixed frequentcy
![[Pasted image 20260114215420.png]]

**Session-based windows**
- Dynamic
- User-centric
- Natural groupings
![[Pasted image 20260115082943.png]]

**Watermarking**
timestamps that keep track of progress in pipeline
watermarking based on event time

![[Pasted image 20260115083514.png]]

**Trigger**
- conditions that determine when the aggregated results of data should be emitted 
- ![[Pasted image 20260115083909.png]]

### Common challenges

**Increases latency**
- increases in end-to-end or per-stage latency
- indicators of bottlenecks:
	- specific stage run longer than expected	
	- data backlogs in certain stage
	- high la
