### GCP cloud SDK:
- gcloud: command line interface
- Client libraries
- bg: command line interface for BigQuery
- gsutil: command line interface for Cloud Storage

### gcloud

Interact with services on GCP
`gcloud config configurations create [NAME]`: to create config 
`gcloud config configurations activate [NAME]`: to activate config 

require:
1. Authenicate: `gcloud auth login`
2. Set default project: `gcloud config set project [PROJECT_ID]`

### Resource hierarchy

**Project**: 
- Fundamental, lowest unit 
- Contains all GCP resources/services
- Distinct entity for billing, access management and APIs
- Must be in project at all times

Org