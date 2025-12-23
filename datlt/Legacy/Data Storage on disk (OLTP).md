# Terminology
* **Disk I/O Operations**: These are write and read operations that involve accessing the computer disk
* **Records**: a row in database table maps to a record on computer's disk. There are different types of records. Data records, index records and records to metadata
* **Pages**: a page consists of many records. Pages have fixed size

# Records 
When you add a new row to a database table using SQL, that row is converted internally to a _record_. Think of a record as an array of bytes, where each byte or group of bytes stores information about the record. The information stored in a group of bytes could be:
- The type of the record: data, index, or other metadata.
- Whether the inserted row has columns with null values or columns have fixed or variable length data types.
- Where each column's data starts and ends in the array.
- The actual data encoded in bytes, divided into separate sections depending on whether the data type used for storing the data has a fixed length or a variable length.
- ![[Pasted image 20251223170744.png]]
- The "Information about columns..." here typically includes:
    1. The number of columns that match the condition i.e. fixed length, null or variable length.
    2. The positions of those columns; where those columns start and end. This makes it easier to query the data for a particular column.

# Pages
When this record is created, it is added to a _page._ Databases tend to work with pages. A page is the smallest unit of data that a database will load into a memory cache.
Why pages are important:
* database can have billions of records or more. Store records in a page can have data easier to manage (storing and retrieving)
* with fixed size page, databases determine how much space to allocate in memory for the operations easily 
* easy to find a record in a page in stead search in a billions of records 