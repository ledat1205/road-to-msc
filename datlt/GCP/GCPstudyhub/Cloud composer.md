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

Integrate with other services on GCP: because composer is airflow so its not only integrate with GCP, but also interact with multi-cloud platform

**BigQuery Operator**:
- execute jobs
- manage tables
- manage datasets
- validate data
`BigQueryInsertJobOperator`

**IAM**:
- Composer Admin: full control
- Composer Developer: deploy, modify DAGs
- Composer Viewer: read configurations
- Composer Users: run and schedule DAGs
- Composer environment and storage accessor: access bucket 

**Cloud Functions**:
- most abstracted and lightweight of GPC's compute services
- serverless execution environment
- event-driven code 
- auto scaling 
- GCP version of AWS Lambda

Types of trigger:
sources:
- changes in cloud storage
- new message in pub/sub
- http request

**Cloud Workflow**:
- fully-managed service that allow you to orchestrate and automate the execution of GCP services and HTTP-based APIs in a reliable, scalable and serverless maner
