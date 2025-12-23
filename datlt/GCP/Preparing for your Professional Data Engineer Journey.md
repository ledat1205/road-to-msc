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
- immu