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

**Downstream uses**: While not part of the pipeline, this step is about the downstream uses of the processed data.

The value of the pipeline is realized when it is consumed by various applications and appropriate stakeholders.

**Orchestrate and monitor**: Orchestration and monitoring are essential components that wrap around the entire pipeline.
- Orchestration involves scheduling, managing, and coordinating the various tasks within the pipeline, ensuring they run in the correct order and handle dependencies.
- Monitoring: This involves tracking the health, performance, and data quality of the pipeline, alerting on errors, and ensuring data integrity.

## Processing and common challenges
**Key features of batch data processing**
- Scheduled and automated: designed for periodic execution, often triggered by a schedule (e.g., daily at midnight) or a predefined event (e.g., source file availability).
- high throughput: efficiently process massive datasets
- latency: immediate doesn't matter
- resource optimization: dynamic scale up or down based on workload

**Challenges**
Volume and scale: Rapid data growth overwhelms old systems. Pipelines must auto-scale for fluctuating data volumes.

Data quality: Diverse data sources lead to format issues and errors. Clean, consistent data is crucial for accurate financial reporting.

Complexity and maintainability: Pipelines become complicated with more sources and logic, making them hard to manage and fix.

Reliability, error handling, and observability: Batch job failures delay reports. Pipelines need to be reliable, handle errors gracefully, and provide performance insights.

