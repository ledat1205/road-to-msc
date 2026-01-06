## When to choose batch data pipelines

**Periodic reporting**: For scheduled reports and analytics on historical data.
**Large scale transformations**: For massive datasets that require cleaning, aggregation, and transformation.
**Data warehousing**: For loading and updating data warehouses from various sources.
**Data migration**: For moving large volumes of data between systems or from on-prem to cloud.
**Scheduled backups**: For creating regular, large-scale data backups for disaster recovery or archival.

Components:
**Data sources**: Every pipeline begins with data sources. These are the origins of your raw data, which can come in various formats (e.g., CSV, JSON, database tables, log files).

**Data ingestion**: The first stage of the pipeline is data ingestion. This is the process of acquiring raw data from its sources and transferring it into a central, temporary storage area, often called a 'landing zone' or 'staging area.'  This process is frequently automated and the data is often directly transformed and landed in the final sink.

**Data transformation**: Ingested data may be cleaned, validated, enriched, mapped, and/or restructured into a consistent and standardized format, before landing in the destination sink.

This stage can involve various processing steps, such as filtering out irrelevant data, aggregating information, joining data from different sources, mapping, or applying business logic.

**Data sink**: Throughout the pipeline, data needs to be stored. This includes:
- Intermediate storage: Often part of the ingestion or transformation stages, this holds data temporarily as it moves through the pipeline (e.g., the landing zone in Cloud Storage).
- Final storage: After transformation, the clean, structured data is loaded into a destination optimized for its intended use. The final destination is frequently a data warehouse (like BigQuery), a data lake (like Cloud Storage with formats like Apache Iceberg), or other analytical data stores.

Downstream use