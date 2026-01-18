### Overview
Blob storage

### gsutil
- command line tool specifically for Cloud Storage
- utility functions such as uploading, downloading, deleting, copying
- can enable parallel composite uploading to maximize the use of available bandwidth

### Storage class
![[Pasted image 20260118151506.png]]

Archive storage special case
![[Pasted image 20260118151758.png]]

### Location options
![[Pasted image 20260118151917.png]]

### Lifecycle rules
used to manage the transition of objects to different storage classes or their deletion based on specific condition
![[Pasted image 20260118152204.png]]

### Object versioning
![[Pasted image 20260118152326.png]]

gsutil cmd to enable:  `gsutil versioning set on gs://[BUCKET_NAME]`

### Retention policies
![[Pasted image 20260118152547.png]]

### Autoclass
- Automatically transitions objects based on access pattern
- must be enable 
- save cost (always start at Standard class)

### Recovery Point Objective (RPO)
![[Pasted image 20260118153218.png]]

- maximum acceptable amount of data loss, measured in time, when a disaster/outage occurs
- implement a failover to at least a second region and replicate data more often than your company's RPO. Ex: RPO is 30 minutes, replicate data every 15 minutes

**Approach**: Multi region setup
![[Pasted image 20260118153647.png]]

If RPO need to < 1 hour:
![[Pasted image 20260118153843.png]]

### GCS Access
![[Pasted image 20260118154059.png]]

Common roles:
![[Pasted image 20260118154218.png]]

**Signed URLs**
![[Pasted image 20260118154330.png]]

**Hosting Static website**:
![[Pasted image 20260118154555.png]]

### Transfer appliance
![[Pasted image 20260118154824.png]]

### Storage transfer service
migrate from on-prem or to/from other cloud service
- automate ongoing or scheduled transfer
- high bandwidth and stable connection (at least 100 mbps, 1gbps is preferred)
- up to hundreds of TB 


