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

Read and Write operations.
1. When you add a new record (row) or update an existing record in your database table, that change is reflected on a data page which exists in an in-memory cache.
2. When you want to read a record from your database table, the system first checks if the page exists in the cache before hitting the disk otherwise.

How pages are loaded from the disk into the in-memory cache.
1. **Temporal Locality**: means that the pages are loaded in-memory based on the likelihood that a recently accessed page will be accessed again soon. It means that if you make a query that happens to fetch some data pages from disk, those data pages will be stored in memory for as long as possible to prevent having to go to the disk to fetch them again. 
2. **Spatial Locality**: This works based on the prediction that if a page is loaded in memory, the pages that are stored physically close to that page on disk will likely be accessed soon. As a result, some pages are 'pre-fetched' ahead of when they are actually used. It means that if you run a query that fetches a page which has records with IDs that range from 1-30, the page which has records from 31-60 will also likely be loaded alongside it to prevent a subsequent trip to the disk. 

This process help database minimize I/O operations needed.
Page structure:
![[Pasted image 20251223172328.png]]
- The Page Header contains information about the page like: how many records it contains, how much space it has left, what table a page belongs to etc.
- The Record Offset array helps to manage the location of the records on a page. Each 'slot' in the array points to the beginning of a record, and helps to locate where the record is physically stored on disk.

# Writeback
The records will be added to page are in-memory cache first. Writeback is the process data write page in cache to disk. Because a page can have 2 copies: in cache and in disk so we have to update content in disk since it is the durable data store.

# Write-ahead Logs
