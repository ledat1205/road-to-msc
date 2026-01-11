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

### Org policies to know 

- ``constraints/compute.resouceLocatios``: constrain resources to only operate within certain geographic regions
- ``contraints/iam.allowedPolicyMemberDomains``: control with domain users must belong to in order to be added to IAM
- ``contraints/compute.vmExternalIpAccess``: constrain or block VMs from having external IPs 
- ``contraints/compute.requireOsLogin``: Must use IAM to control SSH access to VMs
![[Pasted image 20260109041547.png]]

### IAM

Principal: Entity can be granted access to GCP resources
**User account**: associate with a human, used for accessing cloud services
**Service account**: associate with services/VMs
Important of service account:
- crucial for secure communication between cloud and application components
- ensure authorized resource can access to each other
- scheduled/automated process

Types:
- Google-managed: google control and used by GCP services
- User-managed: user created, custom permission

Service account admin to create modify user-managed service account need Service Account Admin role

#### Roles and permissions
Roles: control what specific actions a user or service account can perform in GCP

**Basic**: board permissions at the project level 
**Predefined**: defined by GCP, tailored for common job functions
**Custom**: managed by user, any permission

Basic Roles:
- Owner: Full access
- Editor: Read/write
- Viewer: Read only

**Principle of least privilege**
Always allocate the minimum necessary permissions

Best practice to manage multiple people with similar or identical access is using Group
- create a google group
- add relevant member to group 
- add group to IAM with a role

IAM with gcloud:
- `gcloud iam role copy`
- `gcloud ___ get iam-policy`


### Cloud observability suite

**Cloud Logging**: search, store and analyze log data and events from GCP resources and applications
integrated with Cloud Monitoring for analytics and setup alerts 

Core log types:
- Platform logs: GCP log
- Application logs: Code log
- Audit logs: User and system activities log
Retention: default 30 days

Log sinks: Export log to new destinations
![[Pasted image 20260111230214.png]]

long-term retention of audit logs
export sink to storage bucket with the coldline storage class
archive storage cloud be an option if retrieved once a year or less
![[Pasted image 20260111231549.png]]

**Cloud Monitoring**: collect, analyze and visualize metric to monitor GCP resources (application and infra), real-time metrics and alert

Types of metric:
* Built-in metrics: 
	* infra (CPU, RAM, disk,...)
	* application (error rates, latency, response times,...)
	* system metrics (system load, running processes)
* Custom metrics: Business metrics 
* External metrics

Alert: Real-time alert can setup on any metrics
