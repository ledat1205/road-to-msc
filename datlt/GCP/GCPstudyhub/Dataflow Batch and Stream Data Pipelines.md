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
	- high latency value in monitor dashboard
should do:
- inspect log
- identify bottleneck
- check resource limitation, data skew or dependencies

**Missing message**:
notice:
- gaps in data
- incomplete aggregation
- sharp drop throughput
should do:
- convert streaming temporary to batch to check output 
- ensure no data is dropped and isolates if the issue due to pipeline configuration or data loss

debug step:
- store incoming data in cloud storage or bigquery
- create a batch job to run that data
- compare results with streaming pipeline
- review streaming pipeline

**out-of-order data**
![[Pasted image 20260115174111.png]]

**fusion**: combine several pipeline steps into one execution stage. Optimize some process but can lead to parallelization issues
![[Pasted image 20260115174448.png]]

![[Pasted image 20260115174502.png]]

**Preventing fusion**: 
![[Pasted image 20260115174636.png]]

### Networking in dataflow

Essential port:
![[Pasted image 20260116145248.png]]

Firewall rules:
in VPC, ensure firewall rules allow ingress and egress on necessasary port 
- TCP 443 for job management and API request
- TCP 12345 and 12346 for communication between workers and services 
if ports are blocked, dataflow wont be able to communication with each other or with control plane 

