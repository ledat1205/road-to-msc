Airflow cloud managed service

### Cloud Composer API vs Airflow REST API
Cloud Composer API: Manage infrastructure
Airflow API: Manage workflow 

### Architecture
Simple: Airflow + K8s (GKE) + Cloud Storage

Hosting a pipeline
- upload file to specific bucket. bucket name format `<region>-composer-<environment-name>-<project-id>-<random-characters>`
- composer will constantly checks for updates to the environment bucket

### Trigger DAG in Composer
- schedule 
- trigger via API
- manual, on-demand

can leverage trigger via API with cloud functions to perform event-based trigger

Integrate with other services on GCP: because composer is airflow so it 
