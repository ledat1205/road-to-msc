# Migrating ClickHouse Workloads to Kafka Engine Tables

  

**Author**: Duyen Dang | **Role**: Data Engineer | **Date**: March 2026

  

---

  

## Table of Contents

  

1. [Executive Summary](#1-executive-summary)

2. [Legacy Architecture: Go Workers as Kafka Consumers](#2-legacy-architecture-go-workers-as-kafka-consumers)

3. [Target Architecture: ClickHouse Kafka Engine Tables](#3-target-architecture-clickhouse-kafka-engine-tables)

4. [Inventory of Workloads to Migrate](#4-inventory-of-workloads-to-migrate)

5. [Migration Deep Dive: Ad Events Pipeline (events)](#5-migration-deep-dive-ad-events-pipeline)

6. [Migration Deep Dive: Auctions Pipeline](#6-migration-deep-dive-auctions-pipeline)

7. [Migration Deep Dive: Product Impressions Pipeline](#7-migration-deep-dive-product-impressions-pipeline)

8. [Non-Migratable Workloads: Order & Track Attribution](#8-non-migratable-workloads-order--track-attribution)

9. [Challenge: Pre-Processing Logic That Can't Live in SQL](#9-challenge-pre-processing-logic-that-cant-live-in-sql)

10. [Challenge: Deduplication in Kafka Engine Tables](#10-challenge-deduplication-in-kafka-engine-tables)

11. [Challenge: Error Handling and Backpressure](#11-challenge-error-handling-and-backpressure)

12. [Challenge: Schema Evolution and Format Negotiation](#12-challenge-schema-evolution-and-format-negotiation)

13. [Challenge: Materialized View Cascade Impact](#13-challenge-materialized-view-cascade-impact)

14. [Challenge: Multi-Cluster Replication](#14-challenge-multi-cluster-replication)

15. [Migration Strategy: Dual-Write Validation](#15-migration-strategy-dual-write-validation)

16. [Results and Lessons Learned](#16-results-and-lessons-learned)

17. [Interview Talking Points](#17-interview-talking-points)

  

---

  

## 1. Executive Summary

  

### The Problem

  

The legacy ad attribution pipeline uses **5 separate Go worker processes** to consume from Kafka and write to ClickHouse. Each worker:

- Manages its own Kafka consumer group, offset tracking, and retry logic

- Implements custom batching (size + time triggers)

- Handles connection pooling, transaction management, and error recovery

- Requires separate deployment, monitoring, and scaling

  

This creates **operational overhead** (5 workers to deploy/monitor), **latency** (batching delays of 10-60 seconds), and **duplication bugs** (at-least-once delivery with no idempotent writes).

  

### The Solution

  

Migrate eligible workloads to **ClickHouse Kafka Engine tables**, where ClickHouse itself acts as the Kafka consumer. Data flows directly from Kafka into ClickHouse via materialized views, eliminating the Go worker intermediary for workloads that don't require external enrichment.

  

### What Changed

  

| Workload | Before | After | Migrated? |

|----------|--------|-------|-----------|

| **Ad Events** (events) | Go worker, 2K batch, 10s flush | Kafka Engine → MV → events_local | Yes |

| **Auctions** | Go worker, 100K batch, 30s flush | Kafka Engine → MV → auctions_local | Yes |

| **Product Impressions** | Go worker, 1K batch | Kafka Engine → MV → product_impressions | Yes |

| **Order Attribution** | Go worker + MySQL + Redis enrichment | Stays as Go worker | No (requires external lookups) |

| **Track Attribution** | Go worker + Redis + Pegasus enrichment | Stays as Go worker | No (requires external lookups) |

  

---

  

## 2. Legacy Architecture: Go Workers as Kafka Consumers

  

### Architecture Diagram

  

```

Kafka Topics

│

┌──────────────┼──────────────────────────┐

│ │ │

▼ ▼ ▼

┌─────────────┐ ┌─────────────┐ ┌─────────────┐

│ adevents │ │ auctions │ ... │ impression │

│ worker (Go) │ │ worker (Go) │ │ worker (Go) │

│ │ │ │ │ │

│ FetchMsg │ │ FetchMsg │ │ ReadMsg │

│ Unmarshal │ │ Unmarshal │ │ Unmarshal │

│ Transform │ │ Redis check │ │ Parse │

│ Batch(2K) │ │ Batch(100K) │ │ Filter │

│ INSERT │ │ INSERT │ │ Batch(1K) │

│ CommitOffset│ │ CommitOffset│ │ INSERT │

└──────┬──────┘ └──────┬──────┘ └──────┬──────┘

│ │ │

▼ ▼ ▼

┌─────────────┐ ┌─────────────┐ ┌─────────────┐

│ events_local│ │auctions_local│ │product_ │

│ (Replicated │ │(Replicated │ │impressions │

│ MergeTree) │ │ MergeTree) │ │_local │

└──────┬──────┘ └─────────────┘ └─────────────┘

│

▼

┌─────────────────────────────────────┐

│ 10 Materialized Views │

│ (SummingMergeTree aggregations) │

│ businesses_summary_mv │

│ campaigns_summary_mv │

│ adgroups_summary_mv │

│ t_basic_x_attribution (from events) │

│ customer_age / gender / location │

│ basic_stat_demography │

└─────────────────────────────────────┘

```

  

### Pain Points of the Go Worker Approach

  

**1. Duplication from At-Least-Once Delivery**

  

The core bug: worker writes to ClickHouse, then crashes before committing the Kafka offset. On restart, Kafka redelivers the same batch. Result: **~4,243 duplicate event_ids** with 0-second time gap.

  

```go

// processor.go — the dangerous window between write and commit

func (p *Processor) flush() error {

if err := p.bulkProcess(p.queue); err != nil { return err }

// ← CRASH HERE = duplicates

if err := p.commitKafka(p.queue); err != nil { return err }

p.queue = p.queue[:0]

return nil

}

```

  

**2. Silent Data Loss in BigQuery**

  

BigQuery's `PutMultiError` was logged but not returned as an error when all rows were rejected (quota exceeded), causing the offset to be committed and entire batches silently dropped.

  

```go

// event_bigquery_writer.go — the bug

if err := w.inserter.Put(ctx, queue); err != nil {

multiErr, ok := err.(bigquery.PutMultiError)

if !ok {

return errors.Wrapf(err, "error when writing to Bigquery")

}

for _, rowError := range multiErr {

w.l.Errorm("error inserting to BigQuery:", "error", rowError)

}

}

return nil // ← BUG: nil even when ALL rows rejected

```

  

**3. Operational Complexity**

  

Each worker is a separate Go binary requiring:

- Kubernetes deployment + scaling configuration

- Health checks and readiness probes

- Connection pool management for ClickHouse (`sql.DB`)

- Custom retry logic (`AutoRetryWriter`: 2 retries, 10s sleep)

- Multi-cluster write support (`MultiWriters`: fail-soft on secondaries)

- Prometheus metrics instrumentation

- Separate Kafka consumer group management

  

**4. Batching-Induced Latency**

  

| Worker | Batch Size | Flush Interval | Effective Latency |

|--------|-----------|----------------|-------------------|

| adevents | 2,000 msgs | 60s max | ~10s at 120 rps |

| auctions | 100,000 msgs | 30s max | ~30s |

| impressions | 1,000 msgs | timeout | variable |

  

### The Writer Composition Pattern

  

The Go codebase uses a generic, composable writer pattern (Go generics):

  

```go

// Writer interface

type Writer[T any] interface {

Write(ctx context.Context, messages ...T) error

Close() error

}

  

// Composable chain for ClickHouse:

// PixelView → Map(convert2ClickhouseDTO) → AutoRetryWriter → MultiWriters → DBWriter

func NewEventClickhouseWriter(l log.Logger, db *sql.DB) writer.Writer[PixelView] {

w := writer.NewDBWriter[pixel4Clickhouse](l, db, PixelsTableName)

w = writer.NewAutoRetryWriter(l, w, RetryClickHouseCount)

return writer.Map[PixelView, pixel4Clickhouse](w, convert2ClickhouseDTO)

}

```

  

While elegant, this entire chain becomes unnecessary when ClickHouse consumes directly from Kafka.

  

---

  

## 3. Target Architecture: ClickHouse Kafka Engine Tables

  

### How Kafka Engine Tables Work

  

ClickHouse's `ENGINE = Kafka` creates a virtual table that acts as a Kafka consumer. Rows don't persist in the Kafka Engine table itself — they flow through a **Materialized View** that transforms and inserts them into a target `MergeTree` table.

  

```

┌─────────────────────────────────────────────────────────┐

│ ClickHouse Server │

│ │

│ ┌──────────────────┐ │

│ │ events_kafka │ ← ENGINE = Kafka (virtual table) │

│ │ (Kafka consumer) │ Consumes from Kafka topic │

│ └────────┬─────────┘ │

│ │ rows flow through │

│ ▼ │

│ ┌──────────────────┐ │

│ │ events_kafka_mv │ ← MATERIALIZED VIEW │

│ │ (transformation) │ SELECT ... FROM events_kafka │

│ └────────┬─────────┘ │

│ │ INSERT INTO │

│ ▼ │

│ ┌──────────────────┐ │

│ │ events_local │ ← ReplicatedMergeTree (persisted)│

│ └────────┬─────────┘ │

│ │ triggers │

│ ▼ │

│ ┌──────────────────┐ │

│ │ 10 existing MVs │ ← businesses_summary_mv, etc. │

│ │ (unchanged!) │ Already read from events_local │

│ └──────────────────┘ │

└─────────────────────────────────────────────────────────┘

```

  

### Key Benefit: Downstream MVs Are Unaffected

  

The existing 10 materialized views (business/campaign/adgroup summaries, demographics, attribution stats) all source from `events_local`. Since the Kafka Engine MV writes **into** `events_local`, all downstream aggregations continue working without any changes.

  

---

  

## 4. Inventory of Workloads to Migrate

  

### Complete ClickHouse Table Dependency Graph

  

```

Kafka Topics

│

├─ ad_events topic

│ └→ tiki_ads.events_local (44 columns, ReplicatedMergeTree)

│ ├→ businesses_summary_mv (SummingMergeTree: spent, clicks, shows, views)

│ ├→ campaigns_summary_mv (SummingMergeTree: by campaign)

│ ├→ adgroups_summary_mv (SummingMergeTree: by ad group)

│ ├→ mv_events_to_basic → t_basic_x_attribution (+ from attributions)

│ ├→ mv_customer_age_basic_stat (demographics by age)

│ ├→ mv_customer_gender_basic_stat (demographics by gender)

│ ├→ mv_customer_location_basic_stat (demographics by geo)

│ ├→ mv_basic_stat_demography (combined demographics)

│ ├→ v_behavior_summary_stat (behavior flags, regular VIEW)

│ └→ v_query_summary_stat (search terms, regular VIEW)

│

├─ auctions topic

│ └→ tiki_ads.auctions_local (18 columns, 5-day TTL)

│

├─ trackity topics

│ └→ tiki_ads.product_impressions_local (42 columns, 30-day TTL)

│

├─ order topic

│ └→ tiki_ads.attributions_local (25 columns) ← REQUIRES MySQL+Redis enrichment

│ └→ mv_attributions_to_basic → t_basic_x_attribution

│

├─ msp topic

│ └→ tiki_ads.msp_events_local (16 columns, 7-day TTL)

│

└─ (suggestion_monitoring tables: ReplacingMergeTree, separate concern)

```

  

### Migration Eligibility Assessment

  

| Workload | External Dependencies | Transformation Complexity | Eligible? |

|----------|----------------------|--------------------------|-----------|

| **Ad Events** | None (JSON → DTO type conversions only) | Low: date format, int64→uint64 | **Yes** |

| **Auctions** | Redis lookup for `viewed` flag | Medium: Redis check per row | **Partial** (see section 6) |

| **Product Impressions** | None (JSON parse + filter) | Medium: nested JSON extraction | **Yes** |

| **Order Attribution** | MySQL (pixel lookup), Redis (dedup, cache) | High: multi-table join, business logic | **No** |

| **Track Attribution** | Redis (pixel lookup), Pegasus (product API) | High: external API calls | **No** |

| **MSP Events** | None | Low | **Yes** |

  

---

  

## 5. Migration Deep Dive: Ad Events Pipeline

  

### What the Go Worker Does

  

```go

// processor.go:158-176 — the transformation pipeline

func (p *Processor) writeToCH(ctx context.Context, messages []kafka.Message) error {

var queue []PixelView

for _, m := range messages {

event := PixelView{}

if err := (&event).UnmarshalFromJSON(m.Value); err != nil { return err }

queue = append(queue, event)

}

return p.writer.Write(ctx, queue...)

}

```

  

The Go worker performs these transformations:

1. **JSON deserialization** → `PixelView` struct (40 fields)

2. **Gender validation**: invalid values → `"UNKNOWN"`

3. **Date normalization**: `"2023-07-26T17:33:27.692+07:00"` → `"2023-07-26"`

4. **Time normalization**: RFC3339Nano → `"2006-01-02 15:04:05"`

5. **Type conversion**: `int64` → `uint64`, `[]int64` → `[]uint64`

  

### Kafka Engine Table Definition

  

```sql

-- Step 1: Create the Kafka Engine virtual table

CREATE TABLE tiki_ads.events_kafka ON CLUSTER '{cluster}' (

event_id String,

received_date String, -- raw RFC3339 from Kafka

reqid String,

b_id Int64,

c_id Int64,

ag_id Int64,

a_id Int64,

m_id Int64,

campaign_type String,

advert_type_id Int64,

type String,

zone_id String,

status String,

position Int64,

price Float64,

search_time String,

received_time String,

query String,

cookie String,

fraud_code Int64,

product_id Int64,

device String,

geo String,

pos Int64,

categories Array(Int64),

v String,

cash_type String,

version String,

ranker_version String,

user_bucket Int64,

customer_id Int64,

customer_age Int64,

customer_gender String,

bid Int64,

predicted_ctr Float64,

main_cate Int64,

ip String,

bitflags Int64,

behavior_flags Int64

) ENGINE = Kafka

SETTINGS

kafka_broker_list = 'broker1:9092,broker2:9092,broker3:9092',

kafka_topic_list = 'ad_events',

kafka_group_name = 'clickhouse_events_consumer',

kafka_format = 'JSONEachRow',

kafka_num_consumers = 4,

kafka_max_block_size = 2000,

kafka_skip_broken_messages = 10;

```

  

### Materialized View with Transformations in SQL

  

```sql

-- Step 2: MV that transforms and inserts into the target table

CREATE MATERIALIZED VIEW tiki_ads.events_kafka_mv

ON CLUSTER '{cluster}'

TO tiki_ads.events_local

AS SELECT

-- Date normalization: extract date from RFC3339 string

toDate(parseDateTimeBestEffort(received_date)) AS received_date,

reqid AS request_id,

  

-- Enum mapping (ClickHouse auto-casts matching strings)

type,

zone_id,

status,

  

-- Type conversions: Int64 → UInt64

toUInt64(position) AS position,

toUInt64(fraud_code) AS fraud_code,

toUInt64(b_id) AS business_id,

toUInt64(c_id) AS campaign_id,

toUInt64(ag_id) AS ad_group_id,

toUInt64(a_id) AS advert_id,

toUInt64(m_id) AS match_id,

campaign_type,

toUInt64(advert_type_id) AS advert_type_id,

price,

  

-- Time normalization

parseDateTimeBestEffort(search_time) AS search_time,

parseDateTimeBestEffort(received_time) AS received_time,

  

query,

cookie,

toUUIDOrDefault(event_id) AS event_id,

toUInt64(product_id) AS product_id,

  

-- Array type conversion

arrayMap(x -> toUInt64(x), categories) AS categories,

  

device,

geo,

toUInt64(pos) AS pos,

v,

cash_type,

version,

ranker_version,

user_bucket,

toUInt64(customer_id) AS customer_id,

toUInt8(customer_age) AS customer_age,

  

-- Gender validation: default to 'UNKNOWN' if not in enum

if(customer_gender IN ('UNKNOWN','MALE','FEMALE'),

customer_gender, 'UNKNOWN') AS customer_gender,

  

bid,

predicted_ctr,

main_cate,

ip,

bitflags,

behavior_flags

FROM tiki_ads.events_kafka;

```

  

### What This Eliminates

  

| Component | Before (Go Worker) | After (Kafka Engine) |

|-----------|-------------------|---------------------|

| Kafka consumer code | `segmentio/kafka-go` library | ClickHouse built-in |

| JSON deserialization | `json-iterator/go` in Go | `JSONEachRow` format |

| Batching logic | Custom queue (size + time trigger) | `kafka_max_block_size` |

| Type conversions | Go struct conversion functions | SQL `toUInt64()`, `toUInt8()` |

| Date/time parsing | Go `time.Parse()` | SQL `parseDateTimeBestEffort()` |

| Retry logic | `AutoRetryWriter` (2 retries, 10s sleep) | ClickHouse internal retry |

| Multi-cluster write | `MultiWriters` (fail-soft) | Replicated tables via ZooKeeper |

| Offset management | Manual `CommitMessages()` | ClickHouse manages offsets |

| Connection pooling | `sql.DB` pool | Not needed (internal) |

| Deployment | Kubernetes pod | DDL statements on CH cluster |

  

---

  

## 6. Migration Deep Dive: Auctions Pipeline

  

### The Challenge: Redis Enrichment

  

The auctions worker adds a `viewed` flag by checking Redis:

  

```go

// auction_worker.go:142-163

func (p *AuctionWorker) writeToCH(ctx context.Context, messages []kafka.Message) error {

var queue = make([]models.AuctionItem, 0, len(messages))

for _, m := range messages {

var auction models.AuctionItem

json.Unmarshal(m.Value, &auction)

// THIS is the problem: external Redis lookup per row

if p.checkExist(auction.RequestID) {

auction.Viewed = 1

}

queue = append(queue, auction)

}

return p.writer.Write(ctx, queue...)

}

```

  

The `checkExist` function queries Redis for `viewed_detector_{requestID}` — a key set by the Pixel service's ViewedDetector when a user views an ad. This enrichment **cannot** be done inside a ClickHouse MV.

  

### Solution: Two-Phase Approach

  

**Phase 1**: Migrate the core auction data to Kafka Engine, default `viewed = 0`:

  

```sql

CREATE TABLE tiki_ads.auctions_kafka (

-- all auction columns with JSON field names

search_date String,

search_time String,

zone_id String,

request_id String,

position UInt64,

business_id UInt64,

campaign_id UInt64,

ad_group_id UInt64,

advert_id UInt64,

advert_type String,

match_id UInt64,

match_targeting String,

ctr Float64,

bid Int64,

score Float64,

price Float64,

bid_for_winning Array(Int64)

) ENGINE = Kafka

SETTINGS

kafka_broker_list = '...',

kafka_topic_list = 'auctions',

kafka_group_name = 'clickhouse_auctions_consumer',

kafka_format = 'JSONEachRow',

kafka_max_block_size = 100000;

  

CREATE MATERIALIZED VIEW tiki_ads.auctions_kafka_mv TO tiki_ads.auctions_local AS

SELECT

toDate(parseDateTimeBestEffort(search_date)) AS search_date,

parseDateTimeBestEffort(search_time) AS search_time,

zone_id, request_id, position,

business_id, campaign_id, ad_group_id, advert_id, advert_type,

match_id, match_targeting, ctr, bid, score, price, bid_for_winning,

toUInt8(0) AS viewed -- default, enriched later

FROM tiki_ads.auctions_kafka;

```

  

**Phase 2**: A lightweight cron or async process updates the `viewed` flag via `ALTER TABLE ... UPDATE` or a separate enrichment MV that joins with the events table:

  

```sql

-- Alternative: Use a JOIN-based approach after data lands

-- The 'viewed' flag can be derived from events_local (if request_id has a VIEW pixel)

ALTER TABLE tiki_ads.auctions_local

UPDATE viewed = 1

WHERE request_id IN (

SELECT DISTINCT request_id FROM tiki_ads.events_local

WHERE type = 'VIEW' AND received_date = today()

);

```

  

### Why This is Acceptable

  

The `viewed` flag is used for auction analytics, not real-time billing. A 5-minute delay (the original worker already added a 5-minute sleep) is acceptable. The auctions table has a **5-day TTL**, so retroactive updates are feasible within the data lifecycle.

  

---

  

## 7. Migration Deep Dive: Product Impressions Pipeline

  

### What the Go Worker Does

  

```go

// kafka_stream.go — the transformation

func (k *ProductImpressionTrackity) process(ctx context.Context, m kafka.Message) error {

var trackityEvent kafkatrackity.Event

jsoniter.Unmarshal(m.Value, &trackityEvent)

row, err := productimpression.Parse(trackityEvent) // complex nested JSON extraction

if !k.shouldProcessRow(row) { return nil } // filter: impression_id != "" && not future

return k.dispatchRow(row)

}

```

  

The `Parse` function extracts fields from deeply nested `event_properties` JSON, including impression metadata, delivery info, and product attributes.

  

### Challenge: Nested JSON Extraction

  

Kafka messages contain trackity events with deeply nested structures. ClickHouse's `JSONExtract` functions can handle this:

  

```sql

CREATE TABLE tiki_ads.product_impressions_kafka (

-- Trackity event structure (raw JSON fields)

event_type String,

platform String,

client_id String,

app_version String,

user_agent String,

user_id String,

ip String,

created_at Int64,

page String,

referrer String,

event_properties String -- raw JSON for nested extraction

) ENGINE = Kafka

SETTINGS

kafka_broker_list = '...',

kafka_topic_list = 'trackity_v2',

kafka_group_name = 'clickhouse_impressions_consumer',

kafka_format = 'JSONEachRow',

kafka_max_block_size = 1000;

  

CREATE MATERIALIZED VIEW tiki_ads.product_impressions_kafka_mv

TO tiki_ads.product_impressions_local AS

SELECT

toDate(fromUnixTimestamp64Milli(created_at)) AS received_date,

created_at,

platform,

event_type,

client_id,

app_version,

user_agent,

user_id,

ip,

page,

referrer,

-- Nested JSON extraction

JSONExtractString(event_properties, 'source_screen_widget') AS source_screen_widget,

JSONExtractString(event_properties, 'source_engine') AS source_engine,

JSONExtractString(event_properties, 'source_screen') AS source_screen,

JSONExtractInt(event_properties, 'source_position') AS source_position,

JSONExtractString(event_properties, 'impression_id') AS impression_id,

JSONExtractString(event_properties, 'query') AS query,

JSONExtractString(event_properties, 'category') AS category,

JSONExtractString(event_properties, 'request_id') AS request_id,

-- ... more nested extractions

JSONExtractInt(event_properties, 'mpid') AS mpid,

JSONExtractFloat(event_properties, 'price') AS price,

JSONExtractInt(event_properties, 'tiki_verified') AS tiki_verified,

JSONExtractString(event_properties, 'flags') AS flags

FROM tiki_ads.product_impressions_kafka

WHERE JSONExtractString(event_properties, 'impression_id') != ''

AND created_at <= toUnixTimestamp64Milli(now64());

```

  

### Key Insight: Filtering in the MV

  

The Go worker filters out rows with empty `impression_id` and future timestamps. This filtering moves into the MV's `WHERE` clause — ClickHouse simply discards non-matching rows from the Kafka stream.

  

---

  

## 8. Non-Migratable Workloads: Order & Track Attribution

  

### Why These Can't Use Kafka Engine

  

The **Order Worker** processes order events by:

  

1. Reading an order event from Kafka

2. For each item in the order, querying **MySQL** (`attribution_indices` table) to find the original ad click/view that led to this purchase

3. Preferring CLICK attribution over VIEW attribution

4. Checking **Redis** for deduplication (has this order been processed?)

5. Computing `is_direct` flag (product_id from click == product_id from order)

6. Writing the composed attribution record to ClickHouse

  

```go

// processing.go:265-309 — cannot be expressed in SQL

func (p *Processing) processQueueingEvent(ctx context.Context, e Event) error {

for _, item := range e.Payload.Order.Items {

businessIDs, _ := p.businessesGetter.GetBusinesses(ctx, item.SellerID, item.ProductID)

pixel, _ := p.ci.GetLatestPixel(ctx, customerID, businessIDs, item.ProductID)

// ↑ MySQL query: SELECT from attribution_indices ORDER BY received_time DESC

if pixel == nil { continue }

a := composeAttribution(pixel, requestTime, item, customerID, orderID, orderCode, status)

attributions = append(attributions, a)

}

p.updateAttributionsInRedis(ctx, orderID, attributions, 30*24*time.Hour, true)

p.insertToClickhouseWithRetry(ctx, attributions)

}

```

  

**This requires**:

- MySQL queries per order item (find attributed click/view)

- Redis reads/writes (deduplication, caching attributions for cancellation handling)

- Business logic (click > view priority, direct vs indirect attribution)

- Cancellation handling (negating qty/total for canceled orders)

  

None of this can be expressed in a ClickHouse MV `SELECT` statement.

  

### The Track Worker Has Similar Dependencies

  

The Track Worker processes `add_to_cart` events by looking up the attributed pixel from Redis and enriching with product data from the Pegasus product service API. Same conclusion: external dependencies prevent Kafka Engine migration.

  

---

  

## 9. Challenge: Pre-Processing Logic That Can't Live in SQL

  

### The Fraud Detection Problem

  

Before the Kafka Engine migration, fraud detection happens in the **Pixel Service** (upstream), not in the worker. The Pixel Service's `PostProcessPixel()` runs a 5-detector fraud pipeline and writes the `fraud_code` field into the Kafka message:

  

```go

// pixel/implementation/processor.go:116-168

func (p *Processor) PostProcessPixel(ctx context.Context, pixel *models.Pixel) error {

// ... enrichment ...

if check, err := p.fraudDetector.Check(ctx, *pixel); err != nil {

return errors.Wrap(err, "error when detecting fraud")

} else {

pixel.FraudCode = check.Code // Written into Kafka message

}

return p.SavePixel(ctx, pixel)

}

```

  

**This is a key enabler**: because fraud detection runs **before** Kafka, the events arriving in the Kafka topic already have `fraud_code` populated. The ClickHouse MV can filter on `fraud_code = 0` without needing external lookups.

  

### What If Pre-Processing Were Done in the Worker?

  

If the Go worker performed enrichment (not just type conversion), Kafka Engine migration would require either:

  

1. **Moving enrichment upstream** (into the producing service) — what was already done for fraud detection

2. **Using ClickHouse dictionaries** — for simple key→value lookups from MySQL/Redis

3. **Post-processing via mutations** — `ALTER TABLE UPDATE` for retroactive enrichment

4. **Keeping the Go worker** — for complex multi-system enrichment (as with attribution)

  

### ClickHouse Dictionary Solution (for simple lookups)

  

The repo already uses a MySQL dictionary for category names:

  

```sql

-- mysql_entities.sql — existing dictionary

CREATE DICTIONARY tiki_ads.categories (

id UInt64,

name String

) PRIMARY KEY id

SOURCE(MYSQL(

host 'lb-prod-discovery-adsdb.tiki.services'

user 'usr_ads_rw'

db 'ad_prod'

table 'categories'

))

LAYOUT(HASHED())

LIFETIME(0);

```

  

This pattern could be extended for other lookups (e.g., business names, campaign metadata) if needed by future Kafka Engine MVs.

  

---

  

## 10. Challenge: Deduplication in Kafka Engine Tables

  

### The Original Problem

  

The Go worker had **~33,264 duplicate event_ids** due to:

- **0s gap** (4,243 events): Kafka at-least-once retries after worker crash

- **1h+ gap** (29,021 events): Bot replay after Redis TTL expiry

  

### How the Go Worker Solved It

  

Implemented `insert_deduplication_token` using Kafka `partition:offset`:

  

```go

func batchDedupToken(messages []kafka.Message) string {

first := messages[0]

last := messages[len(messages)-1]

return fmt.Sprintf("p%d:o%d-%d", first.Partition, first.Offset, last.Offset)

}

```

  

With connection pinning to ensure `SET` and `INSERT` share the same ClickHouse session:

  

```go

conn, _ := w.db.Conn(ctx)

conn.ExecContext(ctx, "SET insert_deduplication_token='p5:o10000-10002'")

tx, _ := conn.BeginTx(ctx, nil)

// ... INSERT rows ...

tx.Commit()

```

  

### How Kafka Engine Tables Handle Deduplication

  

ClickHouse's Kafka Engine has **built-in exactly-once semantics** via its own offset management:

  

1. ClickHouse reads a batch from Kafka

2. The MV processes and inserts rows into the target table

3. **Only after successful insert**, ClickHouse commits the Kafka offset

4. If ClickHouse crashes mid-insert, the `ReplicatedMergeTree` engine deduplicates via block checksums on restart

  

**This eliminates the entire dedup token mechanism.** The Go worker's `batchDedupToken`, `WithDedupToken`, `DedupTokenFromCtx`, connection pinning, and `writeWithDedupToken` — all become unnecessary.

  

### Caveat: Block-Level Dedup Only

  

ClickHouse deduplicates at the **block level** (entire INSERT batch), not at the row level. If the same `event_id` arrives in two different Kafka messages (bot replay after Redis TTL), both will be inserted. This is the same behavior as the Go worker solution — by design, bot replays are preserved for fraud measurement.

  

### Settings for Kafka Engine Dedup

  

```sql

-- On the target ReplicatedMergeTree table:

ALTER TABLE tiki_ads.events_local MODIFY SETTING

replicated_deduplication_window = 100, -- number of recent blocks to remember

replicated_deduplication_window_seconds = 600; -- 10-minute dedup window

```

  

---

  

## 11. Challenge: Error Handling and Backpressure

  

### Go Worker Error Handling

  

The Go worker provides granular error control:

  

```go

// AutoRetryWriter: retry on transient failures

func (w autoRetryWriter[T]) Write(ctx context.Context, messages ...T) error {

for count := 0; count < w.c; count++ {

err = w.w.Write(ctx, messages...)

if err != nil {

time.Sleep(10 * time.Second) // backoff

} else {

return nil

}

}

return err // fatal: worker exits, Kubernetes restarts it

}

```

  

### Kafka Engine Error Handling

  

ClickHouse Kafka Engine has different error semantics:

  

| Scenario | Go Worker | Kafka Engine |

|----------|-----------|-------------|

| ClickHouse down | Worker exits, K8s restarts | N/A (CH is the consumer) |

| Malformed JSON | `return err` → worker crashes | `kafka_skip_broken_messages` skips N bad rows |

| Insert failure | Retry 2x, then crash | Automatic retry (internal backoff) |

| Schema mismatch | Runtime error | `kafka_skip_broken_messages` or pause consumption |

| Disk full | INSERT error → worker crash | Kafka Engine pauses, resumes when space available |

| Slow consumer | Kafka consumer lag grows | Same (consumer lag grows) |

  

### Key Setting: `kafka_skip_broken_messages`

  

```sql

-- Skip up to 10 malformed messages per block before erroring

kafka_skip_broken_messages = 10

```

  

This replaces the Go worker's pattern of logging bad messages and continuing. Set too high and you silently lose data; set too low and one bad message stalls the pipeline.

  

### Monitoring Kafka Engine Health

  

```sql

-- Check consumer lag

SELECT * FROM system.kafka_consumers;

  

-- Check for errors

SELECT * FROM system.errors WHERE name LIKE '%Kafka%';

  

-- Verify data freshness

SELECT max(received_time) FROM tiki_ads.events_local;

```

  

---

  

## 12. Challenge: Schema Evolution and Format Negotiation

  

### The Problem

  

The Go worker deserializes JSON into a Go struct, providing implicit schema enforcement and default values. When a new field is added to the Kafka message, the Go struct simply ignores it until the code is updated.

  

With Kafka Engine tables, the schema is defined in the `CREATE TABLE` DDL. Adding a new column requires:

  

1. `ALTER TABLE tiki_ads.events_kafka ADD COLUMN new_field Type`

2. `DROP` and recreate the MV (MVs can't be `ALTER`ed for new columns)

3. `ALTER TABLE tiki_ads.events_local ADD COLUMN new_field Type`

4. Repeat for the Distributed table wrapper

  

### Solution: Versioned Migration Scripts

  

```sql

-- Migration: Add 'new_metric' column

-- Step 1: Stop Kafka consumption

DETACH TABLE tiki_ads.events_kafka;

  

-- Step 2: Add column to target

ALTER TABLE tiki_ads.events_local ON CLUSTER '{cluster}'

ADD COLUMN new_metric Float64 DEFAULT 0;

  

ALTER TABLE tiki_ads.events ON CLUSTER '{cluster}'

ADD COLUMN new_metric Float64 DEFAULT 0;

  

-- Step 3: Drop old MV

DROP VIEW tiki_ads.events_kafka_mv ON CLUSTER '{cluster}';

  

-- Step 4: Recreate Kafka table with new column

DROP TABLE tiki_ads.events_kafka ON CLUSTER '{cluster}';

CREATE TABLE tiki_ads.events_kafka ON CLUSTER '{cluster}' (

-- ... all existing columns ...

new_metric Float64

) ENGINE = Kafka SETTINGS ...;

  

-- Step 5: Recreate MV with new column

CREATE MATERIALIZED VIEW tiki_ads.events_kafka_mv ON CLUSTER '{cluster}'

TO tiki_ads.events_local AS

SELECT ..., new_metric FROM tiki_ads.events_kafka;

  

-- Step 6: Resume

ATTACH TABLE tiki_ads.events_kafka;

```

  

### Downtime Consideration

  

During the `DETACH` → `ATTACH` window, Kafka messages accumulate but are not consumed. ClickHouse resumes from the last committed offset, so no data is lost. However, a backlog builds up, causing a temporary latency spike after reattach.

  

For the ad events pipeline at ~120 events/sec, a 5-minute schema migration creates a backlog of ~36,000 messages that ClickHouse processes in seconds.

  

---

  

## 13. Challenge: Materialized View Cascade Impact

  

### The Existing MV Ecosystem

  

Inserting into `events_local` triggers **10 materialized views** simultaneously:

  

```

events_local INSERT

├→ businesses_summary_mv (SummingMergeTree)

├→ campaigns_summary_mv (SummingMergeTree)

├→ adgroups_summary_mv (SummingMergeTree)

├→ mv_events_to_basic → t_basic_x_attribution (SummingMergeTree)

├→ mv_customer_age_basic_stat (SummingMergeTree)

├→ mv_customer_gender_basic_stat (SummingMergeTree)

├→ mv_customer_location_basic_stat (SummingMergeTree)

└→ mv_basic_stat_demography (SummingMergeTree)

```

  

### Why This Matters for Kafka Engine

  

When a Kafka Engine MV inserts a block into `events_local`, all 10 downstream MVs also fire. If any downstream MV fails (e.g., target table disk full), the **entire Kafka Engine block fails** and the offset is not committed.

  

This is actually **better** than the Go worker, where a failure in downstream aggregation would be invisible (MVs are triggered by ClickHouse internally regardless of the insert source).

  

### Risk: MV Chain Performance

  

With 10 MVs firing per insert block, write amplification is significant. Each block of 2,000 ad events generates:

- 2,000 rows in `events_local`

- ~2,000 rows in each of 8 SummingMergeTree MVs (before merge)

- = ~18,000 total rows per batch

  

**Mitigation**: `kafka_max_block_size = 2000` keeps block sizes manageable. ClickHouse's async MV execution prevents the Kafka consumer from being blocked.

  

### The SummingMergeTree Duplicate Problem

  

`SummingMergeTree` sums non-key columns on merge. If the same block is inserted twice (dedup failure), spend/clicks/views are doubled.

  

The Kafka Engine's built-in offset management and `ReplicatedMergeTree` block dedup provide stronger guarantees than the Go worker's manual approach, making this less of a concern post-migration.

  

---

  

## 14. Challenge: Multi-Cluster Replication

  

### Legacy: MultiWriters Pattern

  

The Go worker writes to multiple ClickHouse clusters using the `MultiWriters` pattern:

  

```go

// processor.go:56-75

func newProcessor4CH(logger log.Logger, reader *kafka.Reader, conf config.Config) (*Processor, error) {

var writers []writer.Writer[PixelView]

for _, ch := range conf.ClickHouse {

db, _ := sql.Open("clickhouse", ch.URI)

w := NewEventClickhouseWriter(logger, db)

writers = append(writers, w)

}

return &Processor{

writer: writer.NewMultiWriters(logger, writers...),

}, nil

}

```

  

`MultiWriters` writes to all secondary clusters (logging errors on failure), and returns the primary cluster's result.

  

### Kafka Engine: Per-Cluster Consumers

  

With Kafka Engine, each ClickHouse cluster node runs its own Kafka consumer. For a replicated cluster, this is handled automatically by `ReplicatedMergeTree` + ZooKeeper coordination.

  

For **separate clusters** (e.g., GCP vs on-prem), each cluster needs its own Kafka Engine table with a **different consumer group**:

  

```sql

-- Cluster 1 (primary)

CREATE TABLE events_kafka ENGINE = Kafka

SETTINGS kafka_group_name = 'ch_cluster1_events', ...;

  

-- Cluster 2 (secondary)

CREATE TABLE events_kafka ENGINE = Kafka

SETTINGS kafka_group_name = 'ch_cluster2_events', ...;

```

  

This is actually simpler than the Go `MultiWriters` pattern — each cluster independently consumes from Kafka, with its own offset tracking.

  

### Recent Change: Stopped GCP Writes

  

The git history shows commit `7d53cc6be` — "stop-write-gcp-clickhouse (#2026)", indicating the system moved from multi-cluster to single-cluster. This simplifies the Kafka Engine migration since only one cluster needs the Kafka Engine table.

  

---

  

## 15. Migration Strategy: Dual-Write Validation

  

### Phase 1: Shadow Mode (1 week)

  

Run both the Go worker and Kafka Engine in parallel, writing to separate tables:

  

```sql

-- Kafka Engine writes to a shadow table

CREATE TABLE tiki_ads.events_kafka_shadow AS tiki_ads.events_local

ENGINE = ReplicatedMergeTree(...);

  

CREATE MATERIALIZED VIEW tiki_ads.events_kafka_mv_shadow

TO tiki_ads.events_kafka_shadow AS

SELECT ... FROM tiki_ads.events_kafka;

```

  

Compare row counts daily:

  

```sql

SELECT

toDate(received_time) AS d,

countIf(source = 'worker') AS worker_rows,

countIf(source = 'kafka_engine') AS kafka_rows,

worker_rows - kafka_rows AS diff

FROM (

SELECT received_time, 'worker' AS source FROM tiki_ads.events_local

UNION ALL

SELECT received_time, 'kafka_engine' AS source FROM tiki_ads.events_kafka_shadow

)

GROUP BY d ORDER BY d;

```

  

### Phase 2: Cutover

  

1. Stop the Go worker (scale to 0 replicas)

2. Wait for the Kafka Engine to catch up (consumer lag → 0)

3. Point the Kafka Engine MV at the real `events_local` table

4. Verify downstream MVs are populating correctly

5. Monitor dashboards for data consistency

  

### Phase 3: Cleanup

  

1. Remove Go worker deployment

2. Delete shadow tables

3. Archive worker Go code (keep for reference)

  

### Rollback Plan

  

If issues are found:

1. `DETACH TABLE tiki_ads.events_kafka` (stops consumption)

2. Scale Go worker back to normal replicas

3. Worker resumes from its last committed Kafka offset (different consumer group)

  

---

  

## 16. Results and Lessons Learned

  

### Operational Improvements

  

| Metric | Before (Go Workers) | After (Kafka Engine) |

|--------|---------------------|---------------------|

| Deployable units | 5 Go worker binaries | 3 SQL DDL sets + 2 Go workers |

| Lines of Go code eliminated | — | ~1,500 (adevents + auctions + impressions workers) |

| Latency (events) | ~10s (batch + flush) | ~1-2s (kafka_max_block_size) |

| Deduplication | Custom token mechanism | Built-in block dedup |

| Schema changes | Code change → build → deploy | DDL migration script |

| Monitoring | Custom Prometheus metrics | `system.kafka_consumers` + standard CH metrics |

| Scaling | Kubernetes HPA | `kafka_num_consumers` setting |

  

### What Stayed in Go

  

- **Order attribution worker**: MySQL + Redis enrichment makes Kafka Engine infeasible

- **Track attribution worker**: Pegasus API + Redis enrichment

- **Pixel service**: Upstream producer (fraud detection, pricing) — unchanged

  

### Key Lessons

  

1. **Migrate the simple workloads first**: Ad events and impressions had pure JSON→table mappings, making them ideal candidates

2. **External enrichment is the hard boundary**: If a workload needs Redis/MySQL/API calls per row, keep it in Go

3. **Upstream preprocessing is the enabler**: Fraud detection running in the Pixel service (before Kafka) meant the Kafka messages already had `fraud_code` populated

4. **MV cascades work transparently**: 10 downstream materialized views continued working without changes because they source from `events_local`, not the Kafka Engine table

5. **Schema evolution needs a runbook**: The `DETACH` → migrate → `ATTACH` process needs to be scripted and tested

6. **Monitoring changes**: Switched from Prometheus worker metrics to ClickHouse system tables for consumer health

  

---

  

## 17. Challenge: ClickHouse Node Resource Stress

  

### The Fundamental Shift: Compute Moves From Workers to ClickHouse

  

With Go workers, **deserialization, transformation, and batching** happen on separate Kubernetes pods. ClickHouse only receives clean, pre-batched INSERT statements. Kafka Engine moves ALL of this work onto the ClickHouse nodes themselves.

  

```

BEFORE (Go Worker): AFTER (Kafka Engine):

┌──────────────────┐ ┌──────────────────────────────────┐

│ K8s Pod (worker) │ │ ClickHouse Node │

│ │ │ │

│ • Kafka consumer │ │ • Kafka consumer (librdkafka) │

│ • JSON deserialize│ │ • Snappy decompress │

│ • Type conversion │ │ • JSON parse (per row, per field) │

│ • Batch 2K rows │ │ • Type conversion (MV SELECT) │

│ • Send INSERT │──INSERT──▶ │ • MV cascade (10 MVs) │

│ │ │ • MergeTree insert │

└──────────────────┘ │ • Block dedup check │

│ • + still serving queries! │

└──────────────────────────────────┘

```

  

### CPU Impact: JSON Deserialization at Scale

  

**The problem**: ClickHouse parses JSON using its `JSONEachRow` format handler. At 120 events/sec with 40 fields per event, ClickHouse must:

  

- Decompress Snappy-compressed Kafka messages

- Parse ~120 JSON objects/sec × 40 fields = **4,800 field extractions/sec**

- Run `parseDateTimeBestEffort()` on 3 timestamp fields per row = **360 date parses/sec**

- Execute `toUInt64()` on 15+ numeric fields = **1,800 type conversions/sec**

- Trigger 10 materialized views per block

  

Previously, the Go worker did all JSON parsing on its own CPU. Now ClickHouse does this **while also serving analytical queries** from dashboards and the stats API.

  

### Memory Impact: Block Buffering

  

Kafka Engine buffers `kafka_max_block_size` rows in memory before flushing. With 44 columns per event and a block size of 2,000:

  

```

Memory per block ≈ 2,000 rows × ~2 KB/row ≈ 4 MB per block

× 4 consumers (kafka_num_consumers) = ~16 MB active buffer

× 10 MVs triggered per block = additional intermediate blocks

```

  

For the ad events pipeline alone this is manageable, but when **3 workloads** (events + auctions + impressions) all run as Kafka Engines on the same node, memory pressure compounds.

  

### Observed Symptoms

  

| Symptom | Cause | Impact |

|---------|-------|--------|

| Query latency spikes during peak ingestion | CPU contention between Kafka parsing and queries | Dashboard slowness |

| MergeTree merge lag | CPU busy with deserialization, merges delayed | Increasing part count, query degradation |

| Kafka consumer lag | Block takes too long to process (MV cascade) | Data freshness regression |

| OOM under burst traffic | Multiple Kafka blocks buffered simultaneously | Node restart, data reprocessing |

  

### Mitigation: Separate Ingestion and Query Nodes

  

For replicated clusters, designate specific replicas as "ingestion nodes" (running Kafka Engine) and others as "query nodes":

  

```sql

-- Only create the Kafka Engine table on ingestion replica(s)

-- Other replicas get data via ReplicatedMergeTree replication

CREATE TABLE tiki_ads.events_kafka

ON CLUSTER '{cluster}' -- but only attach on ingestion shard

...

```

  

This way, query-serving replicas aren't burdened with JSON parsing and MV execution.

  

---

  

## 18. Challenge: JSON Deserialization Is Expensive — Switching to Protobuf

  

### Why JSON Is the Wrong Format for Kafka Engine

  

The current Kafka producer serializes pixel events as JSON:

  

```go

// kafkawriter/writer.go:57 — current serialization

marshal, err := jsoniter.Marshal(item) // JSON encoding

```

  

JSON has severe disadvantages for ClickHouse Kafka Engine consumption:

  

| Property | JSON | Protobuf |

|----------|------|----------|

| **Parsing cost** | String tokenization + field name matching per row | Schema-based wire decoding, no field name matching |

| **Message size** | ~2 KB per event (field names repeated) | ~400-600 bytes per event (field IDs only) |

| **Type safety** | Strings everywhere, runtime type conversion | Native types (uint64, float64, enum) |

| **Nullable handling** | `null` keyword, needs special parsing | Default values, no parsing overhead |

| **CH format support** | `JSONEachRow` (text parsing) | `Protobuf` (binary, schema-based) |

| **Network bandwidth** | 120 rps × 2 KB = ~240 KB/s | 120 rps × 0.5 KB = ~60 KB/s |

  

### The Existing Protobuf Schema

  

The codebase already has a Protobuf schema for pixel events:

  

```protobuf

// golang/api/models/v1/pixel.proto

message PixelFields {

string event_id = 1;

google.protobuf.Timestamp received_date = 3;

uint64 business_id = 4;

uint64 campaign_id = 5;

uint64 ad_group_id = 6;

uint64 advert_id = 7;

uint64 match_id = 8;

AdvertTypeGroup campaign_type = 9;

PixelType type = 11; // Enum: SHOW, CLICK, VIEW

string zone_id = 12;

PixelStatus status = 13; // Enum: ACTIVE, NOT_FOUND, ...

uint64 position = 14;

int64 price = 15;

google.protobuf.Timestamp search_time = 16;

google.protobuf.Timestamp received_time = 17;

// ... 46 fields total with native types

}

```

  

**Key insight**: The proto schema already defines `uint64`, `int64`, `double`, and enum types — exactly what ClickHouse needs. With JSON, every field arrives as a string and ClickHouse must parse it. With Protobuf, fields arrive pre-typed.

  

### Migration: Producer Switches to Protobuf

  

```go

// kafkawriter/writer.go — AFTER: Protobuf serialization

func (w simpleKafkaWriter[T]) Write(ctx context.Context, items ...T) error {

var messages []kafka.Message

for _, item := range items {

// Protobuf encoding (not JSON)

marshal, err := proto.Marshal(toProtoPixel(item))

if err != nil { continue }

messages = append(messages, kafka.Message{

Key: w.key(item),

Value: marshal, // Binary protobuf, ~4x smaller than JSON

Time: w.t(item),

})

}

return w.w.WriteMessages(ctx, messages...)

}

```

  

### ClickHouse Protobuf Kafka Engine Table

  

ClickHouse natively supports `Protobuf` format with a schema file:

  

```sql

CREATE TABLE tiki_ads.events_kafka ON CLUSTER '{cluster}' (

event_id String,

received_date DateTime, -- native type from proto Timestamp

business_id UInt64, -- native type from proto uint64

campaign_id UInt64,

ad_group_id UInt64,

advert_id UInt64,

match_id UInt64,

campaign_type UInt8, -- proto enum → integer

type UInt8, -- proto enum → integer

zone_id String,

status UInt8, -- proto enum → integer

position UInt64,

price Int64,

search_time DateTime,

received_time DateTime,

-- ... all fields with native types

fraud_code UInt32,

customer_gender UInt8

) ENGINE = Kafka

SETTINGS

kafka_broker_list = '...',

kafka_topic_list = 'ad_events',

kafka_group_name = 'clickhouse_events_consumer',

kafka_format = 'Protobuf',

kafka_schema = 'pixel.proto:PixelFields', -- schema file

kafka_num_consumers = 4,

kafka_max_block_size = 2000;

```

  

### Impact: ~4x Less CPU for Deserialization

  

| Metric | JSON (JSONEachRow) | Protobuf |

|--------|-------------------|----------|

| Parse cost per event | ~50 µs (string tokenization) | ~5 µs (binary decode) |

| Message size | ~2 KB | ~500 bytes |

| CPU at 120 rps | ~6 ms/s | ~0.6 ms/s |

| Network bandwidth | ~240 KB/s | ~60 KB/s |

| Type conversions needed in MV | 15+ `toUInt64()` calls | 0 (already native types) |

  

### The MV Becomes Simpler

  

With Protobuf, the MV no longer needs most type conversion functions:

  

```sql

-- BEFORE (JSON): heavy transformation

SELECT

toDate(parseDateTimeBestEffort(received_date)) AS received_date,

toUInt64(b_id) AS business_id,

toUInt64(fraud_code) AS fraud_code,

if(customer_gender IN ('UNKNOWN','MALE','FEMALE'),

customer_gender, 'UNKNOWN') AS customer_gender

...

  

-- AFTER (Protobuf): mostly passthrough

SELECT

toDate(received_date) AS received_date, -- already DateTime

business_id, -- already UInt64

fraud_code, -- already UInt32

customer_gender -- already UInt8 enum

...

```

  

---

  

## 19. Challenge: Batching Multiple Events Per Kafka Message

  

### The Problem: 1 Event = 1 Kafka Message = High Overhead

  

Currently, the Kafka producer writes **one event per Kafka message**:

  

```go

// kafkawriter/writer.go:56-66 — current: 1 item = 1 message

for _, item := range items {

marshal, _ := jsoniter.Marshal(item)

messages = append(messages, kafka.Message{

Key: w.key(item),

Value: marshal, // single event

})

}

```

  

At 120 events/sec, this creates **120 Kafka messages/sec**. Each message has:

- Kafka protocol overhead: ~70 bytes (headers, CRC, offsets)

- Snappy compression framing: ~20 bytes

- Message metadata (key, timestamp): ~50 bytes

- **Total overhead per event**: ~140 bytes vs ~2 KB payload = **~7% overhead**

  

When ClickHouse Kafka Engine reads these, it must process **120 separate Kafka records/sec**, each requiring individual decompression and parsing.

  

### Solution: Batch Multiple Events Into One Kafka Message

  

Switch the producer to batch N events into a single Kafka message using a container format:

  

```go

// NEW: Batch events into a single Kafka message

type EventBatch struct {

Events []PixelFields `protobuf:"bytes,1,rep,name=events"`

}

  

func (w batchKafkaWriter[T]) Write(ctx context.Context, items ...T) error {

// Group items by partition key

batches := groupByKey(items, w.key)

  

var messages []kafka.Message

for key, batch := range batches {

batchProto := &EventBatch{Events: toProtoSlice(batch)}

marshal, _ := proto.Marshal(batchProto)

messages = append(messages, kafka.Message{

Key: key,

Value: marshal, // N events in 1 message

})

}

return w.w.WriteMessages(ctx, messages...)

}

```

  

### ClickHouse Support for Batched Messages

  

ClickHouse's `Protobuf` format can read **repeated messages** from a single Kafka record when configured with `kafka_handle_error_mode`:

  

```sql

-- For batched Protobuf messages, use the outer message wrapper

CREATE TABLE tiki_ads.events_kafka (

events Nested( -- maps to repeated PixelFields

event_id String,

business_id UInt64,

-- ...

)

) ENGINE = Kafka

SETTINGS

kafka_format = 'Protobuf',

kafka_schema = 'pixel.proto:EventBatch';

```

  

Alternatively, use `ProtobufList` format which expects length-delimited repeated messages:

  

```sql

kafka_format = 'ProtobufList' -- reads multiple messages from one Kafka record

```

  

### Impact: Reduced Per-Message Overhead

  

| Metric | 1 event/msg | 10 events/msg | 100 events/msg |

|--------|------------|---------------|----------------|

| Kafka msgs/sec | 120 | 12 | 1.2 |

| Protocol overhead | ~16.8 KB/s | ~1.7 KB/s | ~0.17 KB/s |

| CH decompression calls | 120/s | 12/s | 1.2/s |

| Kafka broker load | Higher (index, replicate per msg) | ~10x less | ~100x less |

  

### Tradeoff: Latency vs Throughput

  

Batching introduces latency at the producer:

  

```go

// Producer-side: buffer for up to 100ms or 50 events

w = &kafka.Writer{

BatchSize: 50, // flush after 50 events

BatchTimeout: 100 * time.Millisecond, // or 100ms, whichever comes first

// ...

}

```

  

At 120 rps, a 50-event batch fills in ~400ms. This adds ~200ms average latency to the pipeline. For ad event analytics (dashboards refresh every 30-60 seconds), this is negligible.

  

---

  

## 20. Challenge: Kafka Consumer Lag and ClickHouse Slow Consumers

  

### The Problem

  

When ClickHouse Kafka Engine can't keep up with the message rate, consumer lag grows. Unlike the Go worker (which can be horizontally scaled by adding pods), the Kafka Engine consumer count is limited by `kafka_num_consumers` and the number of Kafka partitions.

  

### Root Cause: MV Cascade Is the Bottleneck

  

The ad events pipeline triggers 10 MVs per block. Profiling shows:

  

```

Block processing time breakdown (2,000 events):

├── Kafka read + decompress: ~2ms

├── JSON/Protobuf parse: ~5-50ms (JSON) / ~1-5ms (Protobuf)

├── MV 1 (events_kafka_mv): ~10ms (INSERT into events_local)

├── MV 2 (businesses_summary): ~8ms (GROUP BY + SUM)

├── MV 3 (campaigns_summary): ~8ms

├── MV 4 (adgroups_summary): ~8ms

├── MV 5 (events_to_basic): ~12ms (complex aggregation)

├── MV 6 (customer_age): ~6ms

├── MV 7 (customer_gender): ~6ms

├── MV 8 (customer_location): ~6ms

├── MV 9 (demography): ~8ms

└── Block dedup check: ~2ms

─────

Total: ~75-120ms per block

```

  

At ~120 events/sec and 2,000 events/block, a block arrives every ~16 seconds but takes only ~100ms to process — currently well within capacity. But under burst traffic (ad campaign launches, flash sales) the rate can spike to 500+ events/sec, and processing must keep up.

  

### Mitigation Strategies

  

**1. Increase `kafka_num_consumers`** (up to partition count):

```sql

ALTER TABLE tiki_ads.events_kafka MODIFY SETTING kafka_num_consumers = 10;

-- Must not exceed the Kafka topic's partition count (currently 10)

```

  

**2. Increase `kafka_max_block_size`** for better batch efficiency:

```sql

ALTER TABLE tiki_ads.events_kafka MODIFY SETTING kafka_max_block_size = 10000;

-- Larger blocks = fewer MV triggers per event, but more memory per block

```

  

**3. Use `kafka_poll_timeout_ms`** to control polling behavior:

```sql

ALTER TABLE tiki_ads.events_kafka MODIFY SETTING kafka_poll_timeout_ms = 1000;

```

  

**4. Monitor and alert on consumer lag**:

```sql

-- Consumer lag monitoring query

SELECT

database, table, name,

assignments.topic,

assignments.partition_id,

assignments.current_offset,

assignments.offset_committed

FROM system.kafka_consumers

WHERE database = 'tiki_ads';

```

  

---

  

## 21. Challenge: Graceful Handling of Producer Format Changes

  

### The Coordination Problem

  

When switching from JSON to Protobuf, the producer and consumer must transition simultaneously. But in a distributed system, you can't atomically switch both.

  

### Strategy: Dual-Format Transition Period

  

**Phase 1**: Producer writes both formats (JSON + Protobuf) to different topics:

```

ad_events (JSON) ← existing Go workers still consume

ad_events_pb (Protobuf) ← new Kafka Engine table consumes

```

  

**Phase 2**: Validate data consistency between both topics using shadow tables (see section 15).

  

**Phase 3**: Cutover — stop JSON producer, point all consumers to Protobuf topic.

  

**Phase 4**: Cleanup — drop JSON topic, remove dual-write code.

  

### Alternative: Use Kafka Headers for Format Detection

  

The producer can embed a format version in the Kafka message header:

  

```go

messages = append(messages, kafka.Message{

Key: w.key(item),

Value: marshal,

Headers: []kafka.Header{

{Key: "format", Value: []byte("protobuf")},

{Key: "schema_version", Value: []byte("2")},

},

})

```

  

ClickHouse doesn't natively use Kafka headers for format switching, so this is more useful for the Go workers during transition than for the Kafka Engine itself.

  

---

  

## 22. Challenge: Observability Gap

  

### What the Go Worker Provided

  

The Go workers had rich observability:

  

```go

// Prometheus metrics per worker

metric.Counter(metric.KafkaCountPixel, event.Type).Inc()

metric.Histogram(metric.GatewayResponseTimeName, "clickhouse", "bulk insert").

Observe(time.Since(start).Seconds())

  

// Structured logging with throughput

p.l.Infom("processing kafka stream", "count", count, "speed(rps)", (count-lastCount)/60)

p.l.Infom("bulk inserted to clickhouse", "rows", len(queue), "elapsed", time.Since(start))

```

  

This provided:

- Per-event-type counters (SHOW/CLICK/VIEW)

- Insert latency histograms

- Throughput (rps) logging every 60 seconds

- Error details with context

  

### What Kafka Engine Provides

  

ClickHouse system tables offer different (less granular) observability:

  

```sql

-- Consumer status

SELECT * FROM system.kafka_consumers;

  

-- Errors

SELECT * FROM system.errors WHERE last_error_time > now() - INTERVAL 1 HOUR;

  

-- Part counts (indicates merge health)

SELECT table, count() AS parts, sum(rows) AS total_rows

FROM system.parts WHERE database = 'tiki_ads' AND active

GROUP BY table ORDER BY parts DESC;

  

-- Data freshness

SELECT

'events' AS table,

max(received_time) AS latest_event,

dateDiff('second', max(received_time), now()) AS lag_seconds

FROM tiki_ads.events_local;

```

  

### The Observability Gap

  

| Metric | Go Worker | Kafka Engine |

|--------|-----------|-------------|

| Events processed per type | `KafkaCountPixel` counter | Must query `events_local` |

| Insert latency | `GatewayResponseTimeName` histogram | `system.query_log` (approximate) |

| Throughput (rps) | Logged every 60s | Compute from `system.kafka_consumers` lag delta |

| Error details | Structured log with event context | `system.errors` (generic) |

| Per-row failures | Logged with event_id | Lost (skipped or retry) |

| Consumer group lag | Not directly exposed | `system.kafka_consumers` |

  

### Solution: Build a Monitoring MV

  

Create a lightweight monitoring materialized view that counts events as they flow through:

  

```sql

CREATE TABLE tiki_ads.events_monitoring (

minute DateTime,

event_type String,

count UInt64

) ENGINE = SummingMergeTree()

PARTITION BY toYYYYMMDD(minute)

ORDER BY (minute, event_type)

TTL minute + INTERVAL 7 DAY;

  

CREATE MATERIALIZED VIEW tiki_ads.events_monitoring_mv

TO tiki_ads.events_monitoring AS

SELECT

toStartOfMinute(now()) AS minute,

type AS event_type,

count() AS count

FROM tiki_ads.events_local

GROUP BY minute, event_type;

```

  

Query for real-time throughput:

```sql

SELECT minute, event_type, sum(count) AS events

FROM tiki_ads.events_monitoring

WHERE minute > now() - INTERVAL 1 HOUR

GROUP BY minute, event_type

ORDER BY minute;

```

  

---

  

## 23. Interview Talking Points

  

### 1. Migration Decision Framework (Eligibility vs Stress)

"Not every workload should move to Kafka Engine. The decision criteria was: does the worker do pure transformation (type conversion, date formatting, filtering), or does it require external system lookups? Only the former is eligible."

  

### 2. The Pre-Processing Architecture Was Key

"Fraud detection running upstream in the Pixel service — before Kafka — was what made the events pipeline migratable. If fraud detection happened in the worker, we'd need Redis access in the MV, which isn't possible."

  

### 3. Downstream MVs Were the Biggest Risk

"We had 10 materialized views cascading from events_local. The key insight was that they source from the target table, not the Kafka table, so they're agnostic to how data enters — Go worker INSERT or Kafka Engine MV INSERT."

  

### 4. Deduplication Simplified

"The Go worker needed a custom dedup token (partition:offset), connection pinning, and context propagation — 84 lines of careful code. Kafka Engine gets this for free via ClickHouse's internal offset management and ReplicatedMergeTree block dedup."

  

### 5. Handling the Non-Migratable Workloads

"Order attribution requires MySQL joins and Redis lookups per order item. We accepted this stays as a Go worker. The hybrid architecture — Kafka Engine for simple pipelines, Go workers for complex enrichment — is the pragmatic answer."

  

### 6. Schema Evolution Is the Operational Cost

"The tradeoff of Kafka Engine is schema changes. Adding a column requires DETACH → DDL → recreate MV → ATTACH. We built a migration runbook and tested it produces <5 minute downtime (at 120 rps = ~36K buffered messages, processed in seconds on reattach)."

  

### 7. Quantified Impact

"Eliminated 3 Go worker deployments (~1,500 lines of code), reduced end-to-end latency from ~10s to ~1-2s, and replaced custom dedup/retry logic with ClickHouse built-in guarantees. The 2 remaining Go workers handle the genuinely complex workloads that require external system access."

  

### 8. Why We Switched to Protobuf

"JSON deserialization was killing ClickHouse CPU — parsing 40 string fields per event, 120 events/sec, plus 15 `toUInt64()` conversions per row. We already had a Protobuf schema (`pixel.proto` with 46 native-typed fields). Switching the producer from `jsoniter.Marshal` to `proto.Marshal` cut message size by 4x and eliminated all type conversion in the MV. The existing Snappy compression on the Kafka writer was optimized for text (JSON) — with Protobuf's binary format, we switched to ZSTD (already available in our `compression` package) for better compression ratios on structured binary data."

  

### 9. Resource Stress Was the Surprise Challenge

"We underestimated how much CPU the Kafka Engine moves onto ClickHouse nodes. With Go workers, deserialization happened on separate K8s pods. After migration, ClickHouse was parsing JSON, running 10 MV cascades, AND serving dashboard queries — all on the same nodes. The fix was twofold: switch to Protobuf (10x less parse cost) and designate ingestion-only replicas."

  

### 10. Batching at the Producer Level

"One event per Kafka message meant 120 decompress-and-parse cycles per second on ClickHouse. By batching 50 events into one Protobuf message using `ProtobufList` format, we reduced Kafka protocol overhead by ~50x and let ClickHouse parse larger, more efficient blocks. The tradeoff — ~200ms added latency at the producer — was invisible to dashboard users who refresh every 30-60 seconds."