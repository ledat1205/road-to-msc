## 1. Context — What System Are We Talking About?

I worked on an **ad attribution platform** for an e-commerce marketplace. The core job of the system: when a customer places an order, determine which ad click or view contributed to that purchase, and record it so advertisers can measure their ROI.

At a high level, three event streams are involved:

```
[Ad Click / View]  ──Kafka──►  adevents worker  ──► Redis + MySQL (pixel index)
[User Login]       ──Kafka──►  trackity worker   ──► Redis (trackity→customerID mapping)
[Order Placed]     ──Kafka──►  order worker      ──► lookup Redis/MySQL → write ClickHouse
```

Scale context: hundreds of thousands of ad events per day, tens of thousands of orders per day, **each missed attribution = direct revenue measurement loss for an advertiser**.

---

## 2. The Architecture — How It Was Built

### Pixel storage (the "source" side)

When a user clicks an ad, a pixel event is emitted. The `adevents` worker saves it into two places:

**Redis** — a hash per customer keyed by `(customer_id → {business_id: pixel, product_id: pixel})`, TTL 30 days for clicks, 1 day for views. This is the hot path for attribution lookup.

**MySQL** — an `attribution_indices` table as a durable fallback, using an `INSERT ... ON DUPLICATE KEY UPDATE` with a delay guard ([conversion_mysql.go:199-204](vscode-webview://0m86505sen8vucr97vr66ck79o50o2hpvpcgbb5j13llrr2u488g/golang/pkg/tiki/ad/conversion/conversion_mysql.go#L199-L204)):

```sql
-- only update if the incoming pixel is at least 5 minutes newer (for view pixels)
if(VALUES(received_time) >= received_time + interval 300 second, VALUES(pixel), pixel)
```

This intentional delay on view pixels was a **rate-limiting design** — views are far more frequent than clicks, and writing every view immediately would stress MySQL. Only the latest pixel per `(k1, k2)` key is kept.

### Attribution creation (the "sink" side)

The `order` worker consumes the `oms.order` Kafka topic. For each `cho_in` (order confirmed) event:

1. For each item in the order, fetch `(seller_id, product_id)` → resolve `business_ids`
2. Query Redis for the latest pixel: **click first, then view as fallback** ([get_attributed_pixels_impl.go:24-36](vscode-webview://0m86505sen8vucr97vr66ck79o50o2hpvpcgbb5j13llrr2u488g/golang/pkg/tiki/ad/conversion/get_attributed_pixels_impl.go#L24-L36))
3. Determine `is_direct = (pixel.product_id == item.product_id)` — direct vs indirect attribution
4. Write attributions to Redis (as buffer + dedup key), then to ClickHouse with retry

---

## 3. The Problems — Where Exactly-Once and Late Event Handling Break Down

This is where the interview story gets interesting. Let me walk through three concrete failure modes I identified.

---

### Problem A: The "Zero-Once" Attribution Loss (Exactly-Once Violation)

**What can go wrong:**

The write sequence in the order worker is ([processing.go:299-307](vscode-webview://0m86505sen8vucr97vr66ck79o50o2hpvpcgbb5j13llrr2u488g/golang/cmd/worker/order/order/processing.go#L299-L307)):

```
Step 1: Redis.Set(order_cho_in → attributions)   ← success
Step 2: ClickHouse.Insert() with retry x3        ← all 3 attempts fail
Step 3: Redis.Del() to revert step 1             ← THIS CAN ALSO FAIL
Step 4: return error, do NOT commit Kafka offset
```

On the surface, step 4 looks safe — if we don't commit the offset, Kafka will redeliver and we retry. But if step 3 fails, Redis still has the key. On the next delivery, `shouldWeProcess()` checks Redis, finds the key, and returns `false` — **the message is silently skipped and the Kafka offset is committed**.

The attribution is permanently lost with no error log and no alert. This is an **at-most-once** guarantee masquerading as exactly-once.

**Root cause:** Two independent storage systems (Redis + ClickHouse) with no shared transaction coordinator. There is no atomic "commit both or neither."

---

### Problem B: The Trackity Race Condition (Late Event / Out-of-Order)

**The scenario:**

```
t=0   User (not logged in) clicks an ad → pixel stored under trackity_id in Redis
t=1   User places an order (still not logged in, but cookie is present)
t=2   Order Kafka event arrives at order worker → GetLatestPixel(customer_id=0)
         → customer_id is 0 → falls through to trackity lookup
         → BUT the order event has no trackity/cookie, only customer_id
         → attribution = nil → LOST

t=3   User logs in → trackity→customer merge runs
         → too late, order already processed
```

The deeper version:

```
t=0   User clicks ad (logged out, stored under trackity_id)
t=1   User logs in → UpdateTrackity2Customer runs → pixels migrated to customer_id key
t=2   User places order → Kafka order event published
t=3   Login Kafka event arrives at trackity worker (delayed, out of order)
t=4   Order Kafka event arrives at order worker
         → lookup customer_id key → empty (merge hasn't happened yet at t=3)
         → attribution lost
```

The system has **no mechanism to wait for or correlate these two streams**. The order worker is a pure sequential consumer — it looks up state synchronously at the moment of processing and moves on.

**Root cause:** This is a classic **stream join with late arrival** problem. The attribution computation depends on the join of three streams (click, login, order) but the system processes them independently with no time coordination.

---

### Problem C: Source Non-Idempotency in MySQL Pixel Index

The MySQL `insertOrUpdate` uses `ON DUPLICATE KEY UPDATE` which is correct for upsert semantics. However, the **delay guard logic** creates a subtle non-idempotency:

```sql
if(VALUES(received_time) >= received_time + interval 300 second, VALUES(pixel), pixel)
```

If a view pixel arrives, gets written, and then the same pixel is replayed (e.g., Kafka consumer group rebalance), the replay will be ignored if it arrives within 5 minutes of the first write — which is the correct behavior. But if the process crashes after writing to MySQL but before committing the Kafka offset, on restart the pixel arrives again and is silently dropped by the delay guard. This means **MySQL has the pixel but Redis does not** (since Redis is written by a separate code path), creating a split-brain state between the two pixel stores.

---

## 4. Why This Matters — Quantifying the Impact

- **Duplicate attributions**: if Redis evicts a key (OOM eviction in Redis cluster), the dedup key is gone. The same order can be attributed twice on retry.
- **Lost attributions**: the zero-once scenario above is triggered by any ClickHouse instability (node failover, disk pressure). In a distributed CH cluster, this is not rare.
- **Wrong attribution**: the race condition is proportional to the volume of guest checkout users who log in around purchase time — a significant segment in e-commerce.
- **No observability**: none of these failures produce an explicit error. The only indicator is a gradual count divergence caught by the monitoring cron ([attributions_events_alert.py](vscode-webview://0m86505sen8vucr97vr66ck79o50o2hpvpcgbb5j13llrr2u488g/cron/alert/attributions_events_alert.py)) which runs once per day and fires when the gap exceeds 40%.

---

## 5. What the Proper Solution Looks Like — Flink

### Exactly-Once via Chandy-Lamport Checkpointing

The fundamental insight: **Kafka offset advancement and state mutation must be atomic**. Flink achieves this through distributed snapshots:

```
Kafka Source → [Flink Operators with RocksDB state] → ClickHouse Sink
                          ↑
              Checkpoint barrier injected by JobManager
              Snapshot = { Kafka offsets, all operator state, sink pre-commit handle }
              Stored atomically to S3/HDFS
```

The write protocol to ClickHouse becomes a **2-phase commit**:

- **Pre-commit**: buffer rows, begin ClickHouse transaction, save transaction handle into checkpoint state
- **Checkpoint completes** (Kafka offset + transaction handle saved atomically to durable storage)
- **Commit**: Flink commits the ClickHouse transaction

If the job crashes after pre-commit but before commit, on restart Flink recovers the transaction handle from the checkpoint and re-issues the commit. If it crashes before the checkpoint completes, the transaction is rolled back and the Kafka offset has not advanced — the messages are replayed from the last good offset.

**The critical difference from the current system:** In the current system, "Redis updated" and "Kafka offset committed" are separate operations with a gap between them where a crash leaves inconsistent state. In Flink, the Kafka offset only advances when the checkpoint that includes the corresponding ClickHouse write completes — they move together or not at all.

### Sink Deduplication (Practical Exactly-Once for ClickHouse)

True 2PC requires ClickHouse transaction support, which is experimental. In practice, the pattern is:

1. Each attribution is assigned a **deterministic UUID** derived from `(order_id, item_id, pixel.event_id)`
2. Flink writes with at-least-once to ClickHouse (simpler, more stable)
3. ClickHouse uses `ReplacingMergeTree` on that UUID key — duplicate inserts on replay are collapsed at merge time
4. Queries use `FINAL` modifier or `GROUP BY` dedup to read the deduplicated view

This gives **effectively exactly-once** without needing experimental transactions.

### Late Event Handling via Event-Time Joins

The trackity race condition is solved by modeling attribution as a **temporal stream join**:

```
Click stream   (keyed by customer_id / trackity_id)
Login stream   (keyed by trackity_id → emits customer_id mapping)
Order stream   (keyed by customer_id)
```

In Flink:

- **Click events** are stored in a keyed state store (RocksDB), partitioned by `customer_id`, with a 30-day TTL
- **Login events** trigger a state migration: copy all pixels from `trackity_id` key to `customer_id` key
- **Order events** look up the state store — but crucially, they are held in a **processing buffer** until the watermark passes the order's event time

The watermark is set to `event_time - allowed_lateness` (e.g., 5 minutes). An order event arriving at Flink will wait up to 5 minutes for out-of-order upstream events (like a login event) before being processed. This directly addresses the race condition.

For orders where the login event arrives after the allowed lateness, Flink routes them to a **side output (late event stream)** which can be stored and reprocessed when the login event eventually arrives — zero permanent data loss.

---

## 6. Summary Table — Current vs. Flink

|Concern|Current System|Flink-Based System|
|---|---|---|
|**Exactly-once guarantee**|At-most-once (silent loss on Redis revert failure)|Exactly-once via checkpointed 2PC|
|**Source idempotency**|Redis HMSet overwrites (last-write-wins)|State backend with deterministic keying|
|**Sink deduplication**|None (duplicate rows on retry)|Deterministic UUID + ReplacingMergeTree|
|**Dedup coordination**|Redis key (volatile, evictable)|Flink RocksDB state (durable, checkpointed)|
|**Late event handling**|None (race condition → silent loss)|Watermarks + allowed lateness + side output|
|**Stream join**|Synchronous point lookup at processing time|Stateful temporal join with buffering|
|**Replay / reprocessing**|`recovery_mode` (in-memory, fragile)|Replay from any Kafka offset|
|**Failure observability**|Daily batch monitoring with 40% threshold|Per-checkpoint failure with exact offset|

---

## 7. Key Concepts to Emphasize in the Interview

When the interviewer probes, here are the technical anchors:

**On exactly-once:**

> "Exactly-once in a distributed system is not about preventing retries — it's about making retries safe. The current system uses Redis as a distributed lock, but that lock is not durable. Flink's checkpoint makes the lock durable by co-locating the Kafka offset and the sink state in the same atomic snapshot."

**On source idempotency:**

> "The pixel storage uses Redis HMSet which is last-write-wins per key. This is idempotent only if message ordering is guaranteed. In practice, Kafka rebalances and consumer restarts can cause out-of-order replays. A truly idempotent source needs a monotonic version check — which we had partially in MySQL via the delay guard, but not consistently in Redis."

**On sink deduplication:**

> "ClickHouse doesn't support distributed transactions in production. The practical solution is to push the deduplication responsibility to the sink: assign each attribution a deterministic ID at the computation layer and let ClickHouse's ReplacingMergeTree handle dedup on storage. This moves the exactly-once guarantee from 'transactional' to 'idempotent' — a more pragmatic and operationally stable approach."

**On late events:**

> "The trackity race condition is fundamentally a stream join problem where the join key (trackity→customer mapping) can arrive after the fact it's needed. Watermarks are the standard solution: they let the engine make a bounded-lateness commitment — 'I will wait up to N minutes for late events, after which I emit and route remaining late arrivals to a side output for separate handling.' The current system has no such contract — it just processes whatever state is available at lookup time."

---

## 8. One-Line Pitch to Open the Story

> _"I worked on a real-time attribution pipeline where an ad click had to be correctly credited to a purchase across three independent Kafka streams — clicks, logins, and orders. The system worked well at low traffic, but under failure conditions it violated exactly-once semantics in a way that was completely silent: no errors, no alerts, just wrong numbers in the advertiser dashboard. Walking through how I identified that and what a proper solution looks like is what I want to talk about today."_




### First, clarify what "waiting" actually means

Flink does not literally pause and sleep. Instead, it uses **event time** (the timestamp embedded in the message itself, not the wall clock) and a **watermark** — a signal that says:

> _"I am confident that no event with event_time < W will arrive on this stream anymore."_

When a watermark W passes through the pipeline, Flink knows it is safe to **finalize any computation that depends on events with event_time ≤ W**.

---

### The concrete scenario from your system

Three streams, all keyed by `customer_id`:

```
Click stream:  { event_time, customer_id, business_id, product_id, pixel_data }
Login stream:  { event_time, trackity_id → customer_id }
Order stream:  { event_time, customer_id, items[] }
```

---

### Example: the race condition that currently causes data loss

```
Wall clock    Event                                    Arrives at Kafka consumer
──────────────────────────────────────────────────────────────────────────────
10:00:00      User clicks ad (logged out, trackity=T1)     → adevents worker
10:01:00      User logs in (T1 → customer_id=42)           → trackity worker
10:01:05      User places order (customer_id=42)           → order worker

Kafka delivery (slightly out of order due to partition lag):
10:01:06      order worker receives ORDER event (event_time=10:01:05)
10:01:09      order worker receives LOGIN event (event_time=10:01:00)  ← 3s late
```

In the **current system**, the order worker processes the ORDER event at 10:01:06. At that moment, the pixel lookup for `customer_id=42` returns nothing — the login merge hasn't run yet. Attribution = lost. The LOGIN event arrives at 10:01:09 and updates the state, but the order is already committed to Kafka and to ClickHouse.

---

### How Flink handles this with event-time + watermarks

**Step 1: Assign event timestamps and generate watermarks**

When events enter Flink from Kafka, you assign a watermark strategy:

```
allowed_lateness = 5 minutes

watermark(t) = max_event_time_seen_so_far - allowed_lateness
```

So if the latest event Flink has seen has `event_time = 10:05:00`, the current watermark is `10:00:00`. This means Flink guarantees: _"all events with event_time < 10:00:00 have already arrived."_

**Step 2: The join operator holds events in state until the watermark advances**

The attribution join operator is a **interval join** (or a custom process function with a timer):

```
Rule: join ORDER with LOGIN where LOGIN.event_time ∈ [ORDER.event_time - 5min, ORDER.event_time]
```

When an ORDER event with `event_time=10:01:05` arrives:

- Flink stores it in a **keyed state buffer** (key = `customer_id=42`)
- Flink registers a **timer** to fire when `watermark >= 10:01:05` (i.e., when the system is confident all events up to 10:01:05 have arrived)
- The ORDER event **is not yet processed** — it sits in state

```
State for customer_id=42:
  pending_orders:  [{ event_time=10:01:05, items=[...] }]
  known_pixels:    { business_id=7: { click, event_time=10:00:00 } }   ← from click stream
  timer:           fire when watermark >= 10:01:05
```

**Step 3: The login event arrives**

At wall clock 10:01:09, the LOGIN event (`event_time=10:01:00`, `T1 → customer_id=42`) arrives. Flink processes it:

- Pixel data previously stored under `trackity_id=T1` is merged into the state for `customer_id=42`

```
State for customer_id=42:
  pending_orders:  [{ event_time=10:01:05, items=[...] }]
  known_pixels:    { business_id=7: { click, event_time=10:00:00 } }   ← now under customer key
  timer:           fire when watermark >= 10:01:05
```

**Step 4: Watermark advances past 10:01:05 — timer fires**

A short time later (say 10:06:10 wall clock), the max event_time seen across all partitions reaches `10:06:05`. The watermark advances to `10:01:05`. The timer fires:

- Flink picks up the pending ORDER event
- Looks up `known_pixels` for `customer_id=42` → **finds the click pixel** (merged in step 3)
- Computes attribution → emits to ClickHouse sink
- Clears the order from the pending buffer

**The login event arrived 3 seconds late in wall clock time, but within the 5-minute allowed lateness in event time — so the ORDER event waited in state long enough to see it.**

---

### Visualized as a timeline

```
Event time →    9:56  9:57  9:58  9:59  10:00  10:01  10:02  10:03
                                               click  login  order

Watermark at wall clock 10:01:06:  W = 9:56  (5min behind latest event 10:01:05)
  → ORDER(10:01:05) buffered, timer set for W >= 10:01:05
  → LOGIN(10:01:00) arrives, state updated

Watermark at wall clock 10:06:10:  W = 10:01:05  (5min behind latest event 10:06:05)
  → Timer fires for ORDER(10:01:05)
  → Pixel lookup succeeds → attribution emitted ✓
```

---

### What happens if the login event is MORE than 5 minutes late?

```
Wall clock    Event
10:01:05      User places order    → order Kafka event published
10:08:00      User logs in         → login Kafka event published (event_time=10:08:00... 
                                     but the login HAPPENED at 10:07:00, 
                                     which is > 5min after order at 10:01:05)
```

Here the login `event_time=10:07:00` is outside the join window `[10:01:05 - 5min, 10:01:05]`. When the watermark passes `10:01:05`, Flink fires the timer. The login state is not present yet. Two choices:

**Option A: Emit with best available data (no attribution)**

- Flink emits the order with `pixel = nil` → recorded as organic (no ad attribution)
- The late LOGIN event goes to a **side output**

**Option B: Side output → reprocessing**

- Flink routes the ORDER event to a side output (a secondary Kafka topic: `late_orders`)
- When the LOGIN event eventually arrives, a separate job joins `late_orders` with the now-populated pixel state and recomputes attribution
- Emits a **corrective record** to ClickHouse (which, using `ReplacingMergeTree` on the attribution UUID, replaces the nil attribution with the correct one)

Option B is zero data loss but adds operational complexity. In practice, for your system the 5-minute window covers the vast majority of cases — the login→order lag is usually seconds, not minutes.

---

### Why the current system cannot do any of this

The current `order` worker is a **streaming processor with no state persistence between events**:

```go
// processing.go:148-170 — pure sequential, stateless per message
for {
    m := p.reader.FetchMessage(ctx)   // get one message
    p.process(ctx, e)                  // look up current state, write, done
    p.reader.CommitMessages(ctx, m)    // advance offset — no going back
}
```

There is no concept of "buffer this event and come back to it later." Each message is processed once, immediately, against whatever state happens to be in Redis at that moment. There is no timer, no watermark, no way to express "wait until I'm confident the join partner has arrived."

Flink's key contribution here is making **time-bounded waiting a first-class primitive** in the processing model, backed by durable state so that "waiting" survives process restarts.