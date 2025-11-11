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