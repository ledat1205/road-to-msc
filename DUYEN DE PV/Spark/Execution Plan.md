## Intro

In this article, we'll explore how Apache Spark plans and optimizes its internal execution processes. The article is structured as follows: first, we'll dive into SparkSQL; next, we'll examine the Adaptive Query Execution framework introduced in Spark 3; and finally, we'll review a real case study from Airbnb, demonstrating how they leverage this framework to enhance their data warehouse.

---

## Spark SQL

In a [2015 paper](https://people.csail.mit.edu/matei/papers/2015/sigmod_spark_sql.pdf), Apache Spark presented the new model Spark SQL (it was officially released in May 2014). This new module lets Spark programmers leverage the benefits of relational processing (e.g., declarative queries and optimized storage). Spark SQL introduces two significant enhancements. First, it provides a seamless integration between relational and procedural processing through a declarative DataFrame API that works smoothly with procedural Spark code. Second, it incorporates a highly extensible optimizer, Catalyst, which leverages Scala's features to facilitate the addition of composable rules and manage code generation.

Originally, Spark was designed as a general-purpose cluster computing engine offering a functional programming API. Users can manipulate distributed collections called Resilient Distributed Datasets (RDDs). Each RDD is a collection of Java or Python objects partitioned across a cluster.

The goals of SparkSQL are:

- Support relational processing of Spark’s native RDDs and external data sources using a convenient API.
    
- Offering high performance using DBMS techniques.
    
- Efficiently supporting new data sources,
    
- Enabling extension with advanced analytics algorithms such as graph processing and machine learning.
    

Spark SQL runs as a library on top of Spark and exposes SQL interfaces, which can be accessed via JDBC/ODBC or the DataFrame API.

[

![](https://substackcdn.com/image/fetch/$s_!Peyk!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fcdd1cb18-3251-4ad0-8dfa-3677dd28bd10_684x681.png)



](https://substackcdn.com/image/fetch/$s_!Peyk!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fcdd1cb18-3251-4ad0-8dfa-3677dd28bd10_684x681.png)

Image created by the author. [Reference](https://people.csail.mit.edu/matei/papers/2015/sigmod_spark_sql.pdf)

The primary abstraction in Spark SQL’s API is a DataFrame, a distributed collection of rows with the same schema. A DataFrame is equivalent to a table in a relational database and can be manipulated similarly to RDDs. It keeps track of their schema and supports various relational operations that lead to more optimized execution.

[

![](https://substackcdn.com/image/fetch/$s_!gSlU!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F825c1467-4a15-4c42-ad69-cfdb4f515c08_762x461.png)



](https://substackcdn.com/image/fetch/$s_!gSlU!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F825c1467-4a15-4c42-ad69-cfdb4f515c08_762x461.png)

Image created by the author.

DataFrames can be created from tables from external data sources or existing RDDs of native Java/Python objects. Once constructed, they can be manipulated with relational operators, such as where and groupBy. On the surface, DataFrames provide the same operations as relational query languages like SQL. However, it lets users express the processing logic using programming languages like Java, Spark, or Python.

[

![](https://substackcdn.com/image/fetch/$s_!ZoDc!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8f0ff168-32f2-4527-99fc-4736a4714940_598x290.png)



](https://substackcdn.com/image/fetch/$s_!ZoDc!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8f0ff168-32f2-4527-99fc-4736a4714940_598x290.png)

Image created by the author.

Another important note: unlike the traditional dataframe APIs (e.g., Pandas Dataframe), Spark Dataframe is lazy. Each DataFrame object represents a logical plan to compute a dataset, but no execution occurs until the user calls a special output operation. (This is similar to the transformation and action in RDD.)

Next, we will examine Spark SQL’s Catalyst Optimizer in more detail. Spark's creator designed Catalyst based on functional programming constructs in Scala. Catalyst supports both rule-based and cost-based optimization.

> _**Rule-Based Optimization (RBO):** Rule-based optimization in databases relies on a set of predefined rules and heuristics to choose the execution plan for a query. These rules are usually based on the structure of the query, the operations involved, and the database schema. The optimizer applies these rules in a fixed order, without considering the actual data distribution or workload._
> 
> _**Cost-Based Optimization (CBO):** Cost-based optimization, on the other hand, uses statistical information about the data—such as table size, index selectivity, and data distribution—to estimate the cost of various execution plans. The optimizer evaluates multiple potential plans and chooses the one with the lowest estimated cost. However, it requires accurate statistics available._

Essentially, Catalyst includes a general library for representing trees and applying rules to manipulate them. Developers can extend Catalyst to support desired external data sources and user-defined types.

When we define the DataFrame transformation logic, before the actual execution, it must go through an optimized process that contains four phases: analyzing a logical plan, optimizing the logical plan, physical planning, and finally, code generation.

[

![](https://substackcdn.com/image/fetch/$s_!Qr8B!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2ce2f578-8c0e-4d7c-8b2c-b540fe14961d_989x492.png)



](https://substackcdn.com/image/fetch/$s_!Qr8B!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2ce2f578-8c0e-4d7c-8b2c-b540fe14961d_989x492.png)

Image created by the author. [Reference](https://people.csail.mit.edu/matei/papers/2015/sigmod_spark_sql.pdf)

We will visit each phase below:

- **Analysis:** The first phase begins with a relation to be processed. It can come from the abstract syntax tree retrieved from the SQL parser or the DataFrame object defined using the DataFrame API. Both cases have unresolved attribute references or relations; is the column/table name valid? What is the columns’ type? Spark SQL uses Catalyst rules and a Catalog object to resolve these attributes. It starts by building an “unresolved logical plan” tree with unbound attributes and data types, then applies predefined rules to resolve attributes.
    
    > _The Spark SQL Catalog object enables interaction with metadata for databases, tables, and functions. It allows users to list, retrieve, and manage these entities, as well as refresh table metadata to keep Spark's view in sync with underlying data sources. It’s a central interface for managing and querying Spark SQL metadata._
    
- **Logical Optimization:** After resolving the logical plan, Spark applies standard rule-based optimizations, such as predicate pushdown, projection pruning, null propagation, and more.
    
- **Physical Planning:** Spark SQL takes a logical plan and generates one or more physical plans. It will select the final plan using a cost model.
    
- **Code Generation:** The final query optimization phase involves generating Java bytecode for execution. Spark SQL uses code generation to speed up processing, especially for CPU-bound in-memory datasets. Catalyst simplifies this by leveraging Scala’s [quasiquotes](https://docs.scala-lang.org/overviews/quasiquotes/intro.html), which allow programmatic construction of abstract syntax trees (ASTs) compiled into bytecode at runtime. Catalyst transforms SQL expressions into Scala ASTs, which are compiled and executed.
    

After understanding the SparkSQL original optimizer, we will move to the next section, which describes Spark's new optimization framework.

> To celebrate Lunar New Year (the true New Year holiday in Vietnam), I’m offering _**50% off the annual subscription**_. The offer ends soon; grab it now to get full access to nearly 200 high-quality data engineering articles.
> 
> [50% off the annual subscription](https://vutr.substack.com/subscribe)

## Spark 3 - Adaptive Query Execution

As mentioned in the above section, the Catalyst optimizer uses the cost model framework to choose the optimal plan at the end of physical planning. The framework collects and uses different data statistics (e.g., row count, number of distinct values, NULL values, max/min values, etc.) to help Spark choose the optimal plan.

However, what happens when the statistics are outdated or unavailable? This can lead to suboptimal query plans. In Apache Spark 3, released in 2020, Adaptive Query Execution (AQE) was introduced to tackle such problems with the ability to adjust query plans based on runtime statistics collected during the execution.

Spark operators are typically pipelined and executed in parallel processes. A shuffle or broadcast exchange breaks the pipeline into query stages, where each stage materializes intermediate results. The next stage can only begin once the previous stage is complete. This pause creates an opportunity for re-optimization, as data statistics from all partitions are available before the following operations start.

[

![](https://substackcdn.com/image/fetch/$s_!7EgR!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F4ba39be5-afe9-4411-bd22-df70d7d00087_670x880.png)



](https://substackcdn.com/image/fetch/$s_!7EgR!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F4ba39be5-afe9-4411-bd22-df70d7d00087_670x880.png)

Image created by the author.

Let's overview the flow of the AQE framework:

- The Adaptive Query Execution (AQE) framework starts by executing leaf stages, which do not depend on other stages. (reading data input)
    
- Once one or more of these stages complete materialization, the framework marks them as complete in the physical query plan. It updates the logical query plan with runtime statistics from the completed stages.
    
- The framework uses these new statistics to run the optimizer, applying a selected list of logical optimization rules, the physical planner, and physical optimization rules, including adaptive-execution-specific rules like coalescing partitions and skew join handling.
    
- The AQE framework identifies and executes new query stages with the newly optimized plan.
    
- The execute-reoptimize-execute process repeats until the entire query is completed.
    

Next, we will explore the three features of the AQE framework: dynamically coalescing shuffle partitions, dynamically switching join strategies, and dynamically optimizing skew joins.

### Dynamically coalescing shuffle partitions.

Shuffle operations can significantly impact performance when running queries on large datasets in Spark. Shuffle is expensive because it requires moving data around to redistribute it for downstream operators. Thus, the number of partitions directly affects the shuffle performance. However, determining the optimal number in the first place is challenging:

- Too few partitions can result in large partitions that may cause tasks to spill data to disk.
    
- Too many partitions lead to small partitions, causing inefficient I/O with numerous small network data fetches.
    

To address this, the user can start with a relatively large number of shuffle partitions and then combine smaller adjacent partitions at runtime by analyzing shuffle file statistics with the help of AQE. This approach helps balance partition sizes and improve query performance. Let's visit the example below for a better understanding:

[

![](https://substackcdn.com/image/fetch/$s_!JmBq!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F51f32f8d-fa02-4ebf-b202-2815f3f53425_892x687.png)



](https://substackcdn.com/image/fetch/$s_!JmBq!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F51f32f8d-fa02-4ebf-b202-2815f3f53425_892x687.png)

Image created by the author. [Reference](https://www.databricks.com/blog/2020/05/29/adaptive-query-execution-speeding-up-spark-sql-at-runtime.html)

### Dynamically switching join strategies.

Spark offers several join strategies, with broadcast hash join typically the most efficient when one side of the join comfortably fits in memory. Spark initially plans a broadcast hash join if it estimates that the join relation's size is below a specified threshold at the planning phase. However, this estimate can be off due to factors like highly selective filters or complex joint operations.

To address this, Adaptive Query Execution (AQE) now replans the join strategy at runtime based on the actual size of the join relation collected during the execution.

[

![](https://substackcdn.com/image/fetch/$s_!CHry!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fadfa3560-2c9b-4e23-a3a2-23420d409074_793x589.png)



](https://substackcdn.com/image/fetch/$s_!CHry!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fadfa3560-2c9b-4e23-a3a2-23420d409074_793x589.png)

Image created by the author. [Reference](https://www.databricks.com/blog/2020/05/29/adaptive-query-execution-speeding-up-spark-sql-at-runtime.html)

### Dynamically optimizing skew joins.

Data skew happens when data is not distributed evenly across partitions in a cluster, which can impact query performance, especially during joins. The AQE skew joins optimization addresses this by automatically detecting skew from shuffle file statistics. It then splits the skewed partitions into smaller partitions.

If we don’t employ AQE, our application will have a straggler executor (the one that lags significantly behind the others), which increases the system's overall latency. With the optimization in place, the data from the straggler is split into smaller partitions and is in charge of more executors, which helps improve the overall performance.

[

![](https://substackcdn.com/image/fetch/$s_!_K4E!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2547bf23-b620-435b-95a7-71a33bddafc1_1095x883.png)



](https://substackcdn.com/image/fetch/$s_!_K4E!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2547bf23-b620-435b-95a7-71a33bddafc1_1095x883.png)

Image created by the author. [Reference](https://www.databricks.com/blog/2020/05/29/adaptive-query-execution-speeding-up-spark-sql-at-runtime.html)

We’ll examine how Airbnb uses Spark AQE to overcome and solve a specific problem.

## Case study at Airbnb

### Context

In a blog post in 2017, Airbnb shared that they migrated Data Warehouse (DW) storage from legacy [HDFS clusters to S3](https://medium.com/airbnb-engineering/data-infrastructure-at-airbnb-8adfb34f169c) to provide better stability and scalability. At that time, Aribnb stated they had already leveraged Spark for data processing.

Despite ongoing improvements, the scalability and flexibility of our S3 workloads remain constrained by their inherent characteristics and underlying infrastructure. A 2022 article highlighted how they enhanced their data infrastructure using Apache Spark 3 and Apache Iceberg. In this section, I’ll focus specifically on Spark, examining how Airbnb utilized Spark’s Adaptive Query Execution to optimize data analytics and processing, particularly within their ingestion pipelines.

### How does Spark 3 help?

Airbnb's Hive-based data ingestion framework processes over 35 billion Kafka messages and 1,000+ tables daily, handling datasets from kilobytes to terabytes in hourly and daily partitions.

The first step in migrating to the new tech stack was moving Hive queries to Spark. Unlike Hive, Spark uses predefined shuffle partition values; Airbnb found it challenging to choose the correct number of partitions for the event data ingestion framework due to varying data volumes.

A fixed number of shuffle partitions won't work for all events in the ingestion framework; it may be too large for some jobs and too small for others. Spark 3 AQE is the solution, especially the dynamically coalescing shuffle partitions feature.

It helps Airbnb dynamically adjust shuffle partitions during execution. The feature adjusts partition sizes at runtime to balance data across tasks, resulting in a significant performance boost.

It combines small partitions into larger ones at runtime, adapting to the changing size of shuffle data throughout the job. As data grows or shrinks between stages, AQE continuously coalesces partitions to maintain efficient processing.

---

## **Outro**

This article explores how Spark evolved into a robust analytics engine with the introduction of SparkSQL, which allows users to define processing logic declaratively while the Catalyst optimizer finds the most efficient execution path. Spark then introduced Adaptive Query Execution (AQE), a framework that dynamically adjusts the execution plan based on workload during runtime.

Through Airbnb’s use of AQE, we’ll also delve into how this feature enhances their data ingestion pipelines.

I hope you find this article both enjoyable and informative.

See you in the next article!

---

## **References**

_[1] James Mayfield, Krishna Puttaswamy, Swaroop Jagadish, and Kevin Long, [Data Infrastructure at Airbnb](https://medium.com/airbnb-engineering/data-infrastructure-at-airbnb-8adfb34f169c) (2016)_

_[2] Ronnie Zhu, Edgar Rodriguez, Jason Xu, Gustavo Torres, Kerim Oktay, Xu Zhang, [Upgrading Data Warehouse Infrastructure at Airbnb](https://medium.com/airbnb-engineering/upgrading-data-warehouse-infrastructure-at-airbnb-a4e18f09b6d5) (2022)_

[3] _Michael Armbrust, Reynold S. Xin, Cheng Lian, Yin Huai, Davies Liu, Joseph K. Bradley, Xiangrui Meng, Tomer Kaftan, Michael J. Franklin, Ali Ghodsi, Matei Zaharia [Spark SQL: Relational Data Processing in Spark](https://people.csail.mit.edu/matei/papers/2015/sigmod_spark_sql.pdf) (2015)_

_[4] Wenchen Fan, Herman van Hövell, MaryAnn Xue, [Adaptive Query Execution: Speeding Up Spark SQL at Runtime](https://www.databricks.com/blog/2020/05/29/adaptive-query-execution-speeding-up-spark-sql-at-runtime.html) (2020)_