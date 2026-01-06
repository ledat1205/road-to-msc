## When to choose batch data pipelines

**Periodic reporting**: For scheduled reports and analytics on historical data.
**Large scale transformations**: For massive datasets that require cleaning, aggregation, and transformation.
**Data warehousing**: For loading and updating data warehouses from various sources.
**Data migration**: For moving large volumes of data between systems or from on-prem to cloud.
**Scheduled backups**: For creating regular, large-scale data backups for disaster recovery or archival.

Components:
**Data sources**: Every pipeline begins with data sources. These are the origins of your raw data, which can come in various formats (e.g., CSV, JSON, database tables, log files).

**Data ingestion**: The first stage of the pipeline is data ingestion. This is the process of acquiring raw data from its sources and transferring it into a central, temporary storage area, often called a 'landing zone' or 'staging area.'  This process is frequently automated and the data is often directly transformed and landed in the final sink.

