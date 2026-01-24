### intro
![[Pasted image 20260125050054.png]]

### Command line tools
interact with BigTable data through `cbt`(Cloud BigTable) or `HBase shell`
interact with infra through `gcloud`

### instance config

Development:
- 1 Node total
- Low cost
- No replication
- No throughput guarantee

Prod:
- 1+ cluster
- 3+ nodes per cluster
- replication 
- throughput guarantee

Cluster location  
![[Pasted image 20260125051032.png]]

