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

Note: different between Airflow 2.0 and Airflow 3.0 is API Server


# Airflow: Local Development Environment

local setup with [Astro CLI](https://github.com/astronomer/astro-cli)

# Airflow: DAGs 101

1. A DAG must have a unique identifier
2. The start date is optional and set to None by default
3. The schedule interval is optional and defines the trigger frequency of the DAG
4. Defining a description, and tags to filter is strongly recommended.
5. To create a task, look at the https://registry.astronomer.io/ first.
6. A task must have a unique identifier within a DAG
7. You can specify default parameters to all tasks with default_args that expects a dictionary
8. Define dependencies with bitshift operators (>> and <<) as well as lists.
9. chain helps to define dependencies between task lists


# Airflow sensor
The purpose of a Sensor is to wait for an event.

That can be useful for many different use cases, such as:

- Processing files from an S3 bucket as they arrive while waiting for them.
- Running different tasks at different times but within the same DAG.
- Triggering a data pipeline when another one completes.
- Ensuring an API is available to make requests.
- Transforming data as soon as data are present in a SQL table.

Key takeaways:

- Sensors wait for an event/condition to be met to complete
- By default, a Sensor times out after 7 days. You should define a better value with the `timeout` parameter
- A sensor checks an event/condition at every `poke_interval` (60 seconds by default)
- While a sensor waits, it continuously takes a work slot
- If you have many sensors or expect them to take time before complete, use the `reschedule` mode
- With the reschedule mode, while a sensor is waiting, its status will be `up_for_reschedule`
- You can create a sensor with `@task.sensor`
