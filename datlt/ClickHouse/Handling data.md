# Table Engines

Tree family: Data store in CH

Integration engines: Data will stream from other source to CH when query. Ex: S3, HDFS,... (Can not leverage CH query engine speed)

![[Pasted image 20251207034604.png]]

## Inserting data to table

![[Pasted image 20251207035426.png]]

can use `async_insert` to buffer row for later insert
Note: in case care about real-time much

![[Pasted image 20251207035510.png]]

![[Pasted image 20251207035627.png]]

![[Pasted image 20251207035808.png]]
We can not control, when part is merge to bigger part.
We have to have resource for background merge task beside resource for run query

![[Pasted image 20251207040041.png]]

![[Pasted image 20251207040449.png]]

![[Pasted image 20251207040518.png]]

![[Pasted image 20251207040758.png]]