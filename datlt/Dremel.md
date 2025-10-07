Foundation for Apache Parquet:

How to run:

## 1. Defining the Schema in Protocol Buffers

The schema describes a `Document` that contains a list of `Emails`, and each `Email` may or may not have an `Address`.

`// The overall document is the top-level message (REQUIRED by default)`
`message Document {`
  `// A required field must be present (Max D for DocId is 0, since no optional/repeated parents)`
  `required int64 DocId = 1;` 

  `// The 'Emails' field is REPEATED. This means it is a LIST of Email messages.`
  `// This is the first field that allows repetition, so its level is 1.`
  `repeated Email Emails = 2;` 
`}`

`// A nested message type for an email entry`
`message Email {`
  `// The 'Address' field is OPTIONAL. It can be present or absent (NULL).`
  `// The path to this field is Document.Emails.Address.`
  `// The optional field in the path is 'Address' itself (1) and the repeated field 'Emails' (1).`
  `// Max D for Address = 2 (1 for Emails + 1 for Address).`
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