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