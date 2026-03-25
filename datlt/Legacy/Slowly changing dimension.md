# Type 0: Not accept any change
# Type 1: Overwriting

Always overwrite when dimension records change. The user always observe the latest state of the records 

![[Pasted image 20260323031255.png]]

Useful when we don't care about keeping track of history

# Type 2: Adding new rows

Note: most used one

Tracking records history by adding two 2 colunms: start_day and end_day. A new record version will be added to table with start_day is end_day of old record and end_day cam be null or 9999-12-31

![[Pasted image 20260324225611.png]]

# Type 3: Adding new columns

Track change state of a column.

![[Pasted image 20260325112637.png]]

# Type 4: Separating

## Approach A

Split frequently changing columns into a separate dimension table.

![[Pasted image 20260325112953.png]]

## Approach B

kinda similar to SCD2 but we will maintain latest state and historical changes in 2 separate tables. A flexibility approach allow you can use each table you want based on your demand

![[Pasted image 20260325114544.png]]

# Type 5: Mini-Dimension and Type Outringer

Base dimension has a FK reference to mini-dimension (frequently update). Overwriting this FK to point to latest mini-dimension record

![[Pasted image 20260325142611.png]]

