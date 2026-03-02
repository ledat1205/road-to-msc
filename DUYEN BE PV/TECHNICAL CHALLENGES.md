- Slow code due to wrong usage of framework vertx (not offloading heavy task) + not using redis for idempotency checking but relying on db (inbox pattern) 
- not releasing lock (for idempotency) while hanlding error requests

- Scale down node druid and clickhouse + fix lag clickhouse (kafka engine tables) + MERGE DRUID SEGMENT
- Database Postgres: replication https://stackoverflow.com/questions/14592436/postgresql-error-canceling-statement-due-to-conflict-with-recovery, connection pool handling, partitioning
- Consistency in distributed system with redis and so on
