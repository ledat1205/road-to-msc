based on very simple  mechanism. write to append-log first, flush log to disk then apply change to real object (table,...)

WAL have offset in each event to track status (apply to obj or not)

# Write-Ahead log in PostgreSQL

Workflow:
1. Log first 