## Architecture of a feature store

At a high level, a feature store maintains two classes of data storage along with pipelines and metadata: an offline store for bulk feature data, such as historical or batch-computed features, and an online store for real-time feature serving. It also includes data ingestion and transformation processes and a feature registry that stores metadata to keep track of feature definitions and versions. Finally, it provides query or API interfaces for model training code and serving applications to retrieve features on demand.

