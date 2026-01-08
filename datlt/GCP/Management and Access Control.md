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

**Organizations**:
- top-level container 
- represent for entire company
- project can exist without Org
- centralize management of project
- roles can be granted at this level

![[Pasted image 20260109040126.png]]

**Folder**:
- Optional intermediate container
- Organize projects in local group
- Like department
- Inherit polices and permissions from the parent org
![[Pasted image 20260109040306.png]]

### Project name vs ID vs number
![[Pasted image 20260109040508.png]]

### Org level roles
- Admin: Full control
- Policy admin: manages org policies, constraints, conditions
- Viewer: View-Only
- Browser: Read-Only
