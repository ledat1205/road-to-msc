# Orchestration

goal: coordinate and automation of data flow across various tools and system to delivery data quality data products and analytics 

**The journey to modern data orchestration:

* Pre-Unix Era: Manual batch-processing and scheduling 
* Early Computing: Basic time-based scheduling tools, dedicated ETL tooling 
* Data & Open-Source Renaissance: Increase in data complexity and size, complexity of scheduling and ETL workloads
* Modern orchestration: Raise of pipeline as code, integrate with external systems, time and event based scheduling, observability

# Introduction to Airflow

Why Airflow become very popular?
* Pipeline as code using Python
* Community driven
* Observability 
* Data aware scheduling 
* Highly extensible 

Use cases: 
* Data powered Applications: Application rely on data 
* Critical Operational Processes: Essential workflows that are crucial for a business to function
* Analytics and Reporting: 
* MLOps and AI

How Airflow works:
* DAG refer to a data pipeline 
* Task refer to a single unit of work in DAG (node in the graph)
* Operator refer to the work a task does 
	* Action Operators. Ex. PostgresOperator 
	* Transfer Operators. Ex. S3toSnowflakeOperator 
	* Sensor Operators. Ex. FileSensor

# Airflow: Basics

Airflow core components 
* **API Server**: FastAPI server serving the UI and handling task execution requests
* **Scheduler**: Schedule tasks when dependencies are [[Vocabulary#^fulfill|fulfilled]]
* **DAG File Processor**: [[Vocabulary#^dedicate|dedicated]] process for parsing DAGs
* **Metadata Database**: A database where all metadata are stored 
* **Executor**: Defines how tasks are executed
* **Queue**: Defines the execution task order 
* **Worker**: The process execute the tasks
* **Trigger**: Process running asyncio to support [[Vocabulary#^deferrable|deferrable]] operation

How Airflow run the DAG

![[Pasted image 20251005222336.png]]

Note: different between Airflow 2.0 and Ariflow 