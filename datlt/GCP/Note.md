Week 1:

**Creating a Data Warehouse Through Joins and Unions**

Instead of using `UNION or UNION ALL`, we can use wildcard filter with `_TABLE_SUFFIX` filter
example:
```
SELECT * FROM ecommerce.sales_by_sku_20170801
UNION ALL
SELECT * FROM ecommerce.sales_by_sku_20170802
```
can use:
```
SELECT * FROM `ecommerce.sales_by_sku_2017*`
(WHERE _TABLE_SUFFIX = '0802')
```

**Creating Date-Partitioned Tables in BigQuery**

The query still processes 1.74 GB even though it returns 0 results. The query engine needs to scan all records in the dataset to see if they satisfy the date matching condition in the WHERE clause.

Additionally, the LIMIT does not reduce the total amount of data processed, which is a common misconception.

use partition to reduce data scan

```
PARTITION BY field_name
OPTIONS (
	partition_expiration_days=60,
	description=''
)
```



**Working with JSON, Arrays, and Structs in BigQuery**
- finding the number of elements with `ARRAY_LENGTH(<array>)`
- deduplicating elements with `ARRAY_AGG(DISTINCT <field>)`
- ordering elements with `ARRAY_AGG(<field> ORDER BY <field>)`
- limiting `ARRAY_AGG(<field> LIMIT 5)`



## Data lakes

Think of a data lake as a vast reservoir of data. It stores enormous amounts of raw data in its native format. You can pour any kind of data into it; structured, semi-structured, and unstructured.

For Cymbal, this means storing:

- **Structured data**, like transaction tables from its sales database.
- **Semi-structured data**, like JSON logs from its web servers.
- **Unstructured data**, like customer-submitted product images, videos, and text reviews.

What are the **advantages** of data lakes?

- **Flexibility**: Stores all data types.
- **Agility:** Fast to ingest data.
- **Scalability:** Can grow to exabyte scale.
- **Cost-effectiveness:** Uses low-cost object storage.
- **Support for advanced analytics:** Ideal for AI/ML model training.

What are the **disadvantages** of data lakes?

- **Risk of becoming a "data swamp":** Without proper governance, it can become a disorganized collection of unusable data.
- **Management complexity:** Requires significant overhead to maintain.
- **Time-consuming analysis:** Data often needs to be cleaned and wrangled before it is usable.
- **Security risks:** Raw data formats can increase security and compliance risks.

## Data warehouses

A data warehouse, in contrast, is like a highly organized library. Data is cleaned, transformed, and structured before it's stored. This process, known as schema-on-write, ensures the data is optimized for analysis and business intelligence (BI).

What are the **advantages** of warehouses?

- **Speed:** Optimized for high-performance queries.
- **High-quality data:** Provides consistent and reliable information.
- **Business-focused:** Directly answers common business questions.
- **Historical intelligence:** Enables deep analysis of trends over time.

What are the **disadvantages** of warehouses?

- **Inflexibility:** Cannot easily accommodate new data types or unstructured data.
- **Cost:** Often expensive to build and maintain.
- **Limited data types:** Primarily designed for structured data only.
- **Long development time:** Can take a long time to design and implement.

# The modern approach: Data lakehouse

A data lakehouse architecture combines the low-cost storage of a data lake with the management features and query performance of a data warehouse. The goal is to create a single, unified platform that can support traditional BI, modern data science, and AI workloads without moving or duplicating data.

A lakehouse achieves this by implementing a metadata and governance layer on top of open-format files stored in low-cost object storage, like Google Cloud Storage. This gives you the best of both worlds.

What are the **key feautures** of a Google Cloud data lakehouse?

- Support for most data formats
- Flexible schema-on-read or schema-on-write
- Access for all types of data users
- Cost flexibility based on needs
- Unified data governance
- ACID transaction support

Data lakehouses solve many challenges by unifying data storage and access.  
**The benefits include:**
- Reduced data redundancy
- Unified governance
- Broken-down data silos
- Improved flexibility and scalability

Big query con perform federated query to query on external system

## BigQuery fundamentals

BigQuery is Google's enterprise-grade, fully managed, serverless data warehouse.


**Fully managed**
The infrastructure (like the hardware, the networking, the low-level software) is all handled by Google. Your team doesn't need to worry about patches, updates, or hardware failures.


**Severless**
Serverless takes things a step further. You don't have to provision or manage any servers at all. You simply load your data and start querying. BigQuery automatically allocates the necessary resources to run your queries and scales them up or down based on the complexity of your request.


The magic behind BigQuery is its architecture. BigQuery separates storage from compute.

![[Pasted image 20251124003844.png]]

Think of it like a library. The books are the data, stored reliably and inexpensively in Google's distributed file system. When you want to find specific information, the librarians are the compute resources. BigQuery can call upon thousands of librarians (or compute workers) simultaneously to scan the entire library (your data) very quickly. This distributed processing engine, called Dremel, is what makes your queries run so fast.

Because storage and compute are separate, they can scale independently.

   
Let's break down how BigQuery achieves its incredible speed by examining two core concepts:  
**slots and shuffle.**

What is a slot?

Think of a slot as a virtual worker—a small, self-contained unit of computational power that includes CPU, RAM, and network bandwidth. When you run a query, BigQuery's Dremel engine assigns potentially thousands of these slots to your job. Each slot processes a small piece of your data simultaneously. This is the "massively parallel processing" that allows BigQuery to scan terabytes of data so quickly.


What is shuffle?

When the results from all those parallel workers need to be combined, such as for a `GROUP BY` or a `JOIN`, the shuffle comes in. Shuffle is the process of redistributing the intermediate data that the slots have processed. Using Google’s petabit internal network, Jupiter, shuffle gathers and reorganizes this data, sending it to the next set of slots for further processing like aggregation or joining. This incredibly fast redistribution of data between query stages is essential for executing complex analytical queries efficiently at a massive scale.

### Partitioning and clustering in BigQuery

**Partitioning**

Partitioning is like adding dividers to a filing cabinet. Instead of one giant drawer, you have separate sections for each year, month, or day. In BigQuery, you can partition a table based on a date or an integer column.

- faster elapsed time
- faster slot time consumed (few worker to run query)
- bytes shuffled smaller (data move between slot fewer)


**Clustering**

While partitioning divides the data into large chunks, clustering sorts the data within each of those chunks. Think of it as organizing the files within each drawer of your filing cabinet alphabetically by customer name.

it can jump directly to the data for that customer instead of reading through the entire partition.

BigQuery works differently depending on whether you’re using its native tables or external Apache Iceberg tables stored in Cloud Storage.

Whether you’re working with BigQuery native tables or Apache Iceberg tables, the principle is the same: **Use metadata to skip unnecessary data, so queries run faster and cost less.**


### **BigLake and external tables**

BigLake acts as a storage engine and connector that allows you to extend the capabilities of BigQuery to your data in object storage, like Google Cloud Storage. BigLake lets you create tables in BigQuery that do not hold the data themselves but instead point to the data files living in your data lake. These are called **external tables**.

**Governance and security**

One of the most powerful features of BigLake is how it centralizes governance and security. You can apply fine-grained security controls, including row-level and column-level security, directly on the BigLake tables within BigQuery. This is enabled through access delegation.