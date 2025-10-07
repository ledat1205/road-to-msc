Foundation for Apache Parquet:

How to run:

## 1. Defining the Schema in Protocol Buffers

The schema describes a `Document` that contains a list of `Emails`, and each `Email` may or may not have an `Address`.

`message Document {`
	`required int64 DocId = 1;` 
	`repeated Email Emails = 2;` 
	`}`
`message Email {` 
	`optional string Address = 1;` 
	`}`


## 2. Calculating the Levels from the Schema

The process of calculating the Max Repetition and Max Definition for any leaf field follows these simple rules:

### A. Max Repetition Level (Max R)

The **Max Repetition Level** for a column is the count of all **REPEATED** fields in its path, starting from the root message (but excluding the root message itself).

| Field Path     | Multiplicity | Is REPEATED? | Max R Calculation                        | Max R |
|----------------|--------------|--------------|------------------------------------------|-------|
| DocId          | Required     | No           | 0                                        | 0     |
| Emails.Address | Optional     | Yes (Emails) | Count of REPEATED fields in path: Emails | 1     |
### B. Max Definition Level (Max D)

The **Max Definition Level** for a column is the count of all **OPTIONAL** and **REPEATED** fields in its path, starting from the root message (but excluding the root message itself).

| Field Path     | Multiplicity | Is OPTIONAL/REPEATED?        | Max D Calculation                                          | Max D |
|----------------|--------------|------------------------------|------------------------------------------------------------|-------|
| DocId          | Required     | No                           | 0                                                          |       |
| Emails.Address | Optional     | Yes (Emails) + Yes (Address) | Count of OPTIONAL/REPEATED fields in path: Emails, Address | 2     |