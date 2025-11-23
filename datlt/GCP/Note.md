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