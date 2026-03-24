# Type 0: Not accept any change
# Type 1: Overwriting

Always overwrite when dimension records change. The user always observe the latest state of the records 

![[Pasted image 20260323031255.png]]

Useful when we don't care about keeping track of history

# Type 2: Adding new rows

Note: most used one

Tracking records history by adding two 2 colunms: start_day and end_day. A new record version will be added to table with start_day is end_day of old record and end_day cam be null or 9999-12-31

![[Pasted image 20260324225611.png]]

# Type 3: 
