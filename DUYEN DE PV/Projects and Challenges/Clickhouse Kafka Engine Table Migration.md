

## 1. The Problem: Bimodal Duplicate Distribution

  

```

Distribution of duplicate event_ids over time gap (in seconds)

═══════════════════════════════════════════════════════════════

  

Count of event_ids

│

│ ╭─────────────────╮

│ │ KAFKA RETRY │ 4,243 event_ids

│ │ DUPLICATES │ (0 seconds apart)

│ │ Gap: 0s │

│ ╰─────────────────╯

│ ▲

│ │

│ │

│ 5,000├─────●

│ │ │

│ │ │

│ │ │ NOTHING HERE!

│ │ │

│ │ │ (Redis working)

│ │ │

│ 3,600│──────●────────────────────────────────────╮

│ │ │

│ │ ╭──────────────────────╮ │

│ │ │ BOT REPLAY AFTER │ 29,021 event_ids

│ │ │ REDIS TTL EXPIRES │ Gap: 1h-30d

│ 1,000├─────│ (1h-30d apart) │

│ │ ╰──────────────────────╯

│ │

│ 0 └──────────────────────────────────────────────┬────── Time Gap (s)

│ 0 1h 1d 7d 30d

  

KEY INSIGHT:

• Gap at 0s → Kafka fault tolerance (at-least-once retries)

• Gap at 1h+ → Business problem (bot replays after TTL)

• NO gap between 1s-3599s → Redis is working perfectly!

```

  

---

  

## 2. Data Flow: Before vs. After

  

### BEFORE (Broken)

```

User searches Ad served Pixel fired

│ │ │

▼ ▼ ▼

┌─────────────────┐ ┌────────────────┐ ┌──────────────────┐

│ Searcher │ │ Pixel Service │ │ Client │

│ │ │ │ │ Browser/App │

│ Generate UUID │ │ Redis dedup │ │ │

│ event_id=XYZ │ │ (1h TTL) │ │ User clicks │

└────────┬────────┘ │ ✓ Check key │ │ pixel URL │

│ │ ✗ If missing │ │ Multiple times │

│ │ → write │ │ (or bot replays) │

│ │ to Kafka │ │ │

│ └────────┬───────┘ │ │

│ │ └────────┬─────────┘

└────────────────────┴────────────────────┘

│

▼ Kafka Topic

┌───────────────────┐

│ [msg1: event_id] │ ← partition:offset = 5:10000

│ [msg2: event_id] │ ← partition:offset = 5:10001

│ [msg3: event_id] │ ← partition:offset = 5:10002

└────────┬──────────┘

│

WORKER CRASHES HERE

(before committing offset)

│

┌────────▼────────────┐

│ Redelivers same │

│ messages again │

│ (partition:offset │

│ still 5:10000-02) │

└────────┬────────────┘

│

┌──────────────────┼──────────────────┐

│ │ │

▼ ▼ ▼

┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐

│ ClickHouse │ │ BigQuery │ │ Logs (audit) │

│ │ │ │ │ │

│ PROBLEM: │ │ PROBLEM: │ │ ✓ Has complete │

│ ✗ Duplicates │ │ ✗ Duplicates │ │ record │

│ inserted │ │ (1min │ │ (but slow) │

│ again │ │ window) │ │ │

│ │ │ ✗ Silent │ │ BOTTLENECK: │

│ event_id used │ │ data loss │ │ Slow to query │

│ as dedup key │ │ (quota) │ │ Real-time not │

│ (WRONG!) │ │ │ │ possible │

│ │ │ InsertID= │ │ │

│ Row count: │ │ event_id │ │ │

│ 6 + 3 = 9 │ │ (WRONG!) │ │ │

│ (should be 6) │ │ │ │ │

│ │ │ Row count: │ │ │

│ │ │ Varies │ │ │

│ │ │ (missing │ │ │

│ │ │ days on │ │ │

│ │ │ quota) │ │ │

└──────────────────┘ └──────────────┘ └──────────────────┘

```

  

### AFTER (Fixed)

```

User searches Ad served Pixel fired

│ │ │

▼ ▼ ▼

┌─────────────────┐ ┌────────────────┐ ┌──────────────────┐

│ Searcher │ │ Pixel Service │ │ Client │

│ │ │ │ │ Browser/App │

│ Generate UUID │ │ Redis dedup │ │ │

│ event_id=XYZ │ │ (1h TTL) │ │ User clicks │

└────────┬────────┘ │ ✓ Check key │ │ pixel URL │

│ │ ✗ If missing │ │ Multiple times │

│ │ → write │ │ (or bot replays) │

│ │ to Kafka │ │ │

│ └────────┬───────┘ │ │

│ │ └────────┬─────────┘

└────────────────────┴────────────────────┘

│

▼ Kafka Topic

┌───────────────────┐

│ [msg1] │ ← partition:offset = 5:10000

│ [msg2] │ ← partition:offset = 5:10001

│ [msg3] │ ← partition:offset = 5:10002

└────────┬──────────┘

│

PROCESSOR READS & COMPUTES

┌──────────────────────────┐

│ Token = "p5:o10000-10002"│

│ (partition:offset range) │

│ (deterministic!) │

│ (unique per batch!) │

└────────┬─────────────────┘

│

WORKER CRASHES HERE

(before committing offset)

│

┌────────▼────────────┐

│ Redelivers same │

│ messages again │

│ (SAME TOKEN!) │

│ Token = "p5:o10000 │

│ -10002" (identical) │

└────────┬────────────┘

│

┌────────────┼──────────────┐

│ │ │

▼ ▼ ▼

┌─────────────┐ ┌──────────────┐ ┌─────────────┐

│ ClickHouse │ │ BigQuery │ │ Logs │

│ │ │ │ │ │

│ ✓ Token set │ │ ✓ InsertID │ │ ✓ Complete │

│ in │ │ = partition│ │ record │

│ session │ │ :offset │ │ │

│ │ │ (unique) │ │ ✓ No more │

│ ✓ Duplicates│ │ │ │ gaps │

│ detected │ │ ✓ All-fail │ │ │

│ & skipped │ │ returns │ │ │

│ (dedup │ │ error │ │ │

│ works!) │ │ (no silent │ │ │

│ │ │ loss!) │ │ │

│ ✓ Row count │ │ │ │ │

│ 6 │ │ ✓ Row count │ │ │

│ (correct) │ │ 6 │ │ │

│ │ │ (correct) │ │ │

└─────────────┘ └──────────────┘ └─────────────┘

```

  

---

  

## 3. Root Causes: Side-by-Side Comparison

  

```

╔════════════════════════════╦═════════════════════════════╦══════════════════════════════╗

║ ROOT CAUSE 1 ║ ROOT CAUSE 2 ║ ROOT CAUSE 3 ║

║ Kafka At-Least-Once ║ Bot Replay After TTL ║ Silent Data Loss in BQ ║

╠════════════════════════════╬═════════════════════════════╬══════════════════════════════╣

║ TIME GAP: 0 seconds ║ TIME GAP: 1h-30d ║ TIME GAP: Variable ║

║ COUNT: ~4,243 event_ids ║ COUNT: ~29,021 event_ids ║ COUNT: Entire days lost ║

║ ║ ║ ║

║ MECHANISM: ║ MECHANISM: ║ MECHANISM: ║

║ ║ ║ ║

║ 1. Worker processes batch ║ 1. Bot fires pixel at T=0s ║ 1. Worker sends 2K rows to ║

║ 2. Inserts to CH/BQ ✓ ║ 2. Redis key set (1h TTL) ║ BigQuery Inserter.Put() ║

║ 3. CRASHES (mid-commit) ║ 3. Message sent to Kafka ║ 2. Quota exceeded ║

║ 4. Offset NOT committed ║ 4. Inserted to CH/BQ ║ 3. Returns PutMultiError ║

║ 5. Kafka redelivers same ║ ║ (all rows rejected) ║

║ batch (partition:offset ║ 5. At T=3600s: Redis key ║ 4. Code logs error but ║

║ unchanged) ║ expires ║ doesn't return it ║

║ 6. Same rows inserted ║ 6. Bot fires again ║ 5. Falls through → returns ║

║ again with 0s gap ║ 7. Passes repetition ║ nil (success!) ║

║ ║ detector → new Kafka ║ 6. Offset committed ║

║ SOLUTION: ║ message ║ 7. ALL rows silently ║

║ ║ 8. Inserted again ║ dropped! ║

║ Use partition:offset as ║ (1h+ gap) ║ ║

║ dedup token. When batch ║ ║ SOLUTION: ║

║ redelivered, token is ║ SOLUTION: ║ ║

║ IDENTICAL. ClickHouse/BQ ║ ║ Check if len(multiErr) == ║

║ dedup tokens → skips ║ Option 1 (preferred): ║ len(queue). If yes, return ║

║ duplicate insert. ║ Accept as fraud measurement ║ error → don't commit offset ║

║ ║ (same ad, different day = ║ → batch retried on restart. ║

║ VERIFICATION: ║ legitimate view) ║ ║

║ ║ ║ VERIFICATION: ║

║ Kill worker mid-batch. ║ Option 2 (future): ║ ║

║ Restart. Row count = ║ Use ReplacingMergeTree to ║ Check CloudLogging for ║

║ same (idempotent!). ║ auto-dedupe on key. ║ RESOURCE_EXHAUSTED errors. ║

║ ║ ║ ║

║ ║ Option 3 (future): ║ After fix, no more gaps ║

║ ║ Increase Redis TTL to 24h ║ in BQ dashboard. ║

║ ║ (reduces window, not ║ ║

║ ║ eliminates). ║ ║

╚════════════════════════════╩═════════════════════════════╩══════════════════════════════╝

```

  

---

  

## 4. The Dedup Token Pattern

  

```

┌─────────────────────────────────────────────────────────────────────────┐

│ DEDUP TOKEN GENERATION │

└─────────────────────────────────────────────────────────────────────────┘

  

Kafka Messages (batch)

═════════════════════════════

┌──────────┐

│ msg0 │ Partition: 5

│ Offset: 10000

└──────────┘

┌──────────┐

│ msg1 │ Partition: 5

│ Offset: 10001

└──────────┘

┌──────────┐

│ msg2 │ Partition: 5

│ Offset: 10002

└──────────┘

│

▼

┌─────────────────────────┐

│ batchDedupToken() │

│ │

│ first = msg[0] │

│ last = msg[len-1] │

│ │

│ token = sprintf( │

│ "p%d:o%d-%d", │

│ first.Partition, │

│ first.Offset, │

│ last.Offset) │

│ │

│ token = "p5:o10000-10002"

└─────────────────────────┘

│

▼

┌─────────────────────────┐

│ Attach to context │

│ │

│ ctx = WithDedupToken( │

│ ctx, token) │

│ │

│ Carry through pipeline: │

│ Processor → Writer │

└─────────────────────────┘

│

▼

┌─────────────────────────┐

│ Extract from context │

│ │

│ token, ok := │

│ dedupTokenFromCtx(ctx)│

│ │

│ if ok { │

│ writeWithDedupToken() │

│ } │

└─────────────────────────┘

│

▼

┌───────────────────────────────────────────────────────────────────────────┐

│ ClickHouse Session-Level Dedup │

│ ═══════════════════════════════════════════ │

│ │

│ Step 1: Pin exclusive connection │

│ ──────────────────────────────────── │

│ conn := db.Conn(ctx) │

│ defer conn.Close() │

│ │

│ Step 2: Set token in THIS connection's session │

│ ──────────────────────────────────────────── │

│ conn.ExecContext(ctx, │

│ "SET insert_deduplication_token='p5:o10000-10002'") │

│ │

│ Step 3: Begin transaction IN THE SAME SESSION │

│ ─────────────────────────────────────────────── │

│ tx := conn.BeginTx(ctx, nil) ← Same connection, same session │

│ │

│ Step 4: Insert rows (inherits SET from earlier) │

│ ────────────────────────────────────────────── │

│ For each row: │

│ prepare.Exec(values...) ← Runs in session with token set │

│ │

│ Step 5: Commit transaction │

│ ──────────────────────────── │

│ tx.Commit() │

│ │

│ RESULT: ClickHouse dedup table records (token, block_hash) │

│ ══════════════════════════════════════════════════════════════ │

│ On 1st call: INSERT → rows added, token stored │

│ On 2nd call: INSERT with same token → rows skipped (idempotent!) │

│ │

└───────────────────────────────────────────────────────────────────────────┘

```

  

---

  

## 5. Idempotency Guarantee Timeline

  

```

SCENARIO: Worker crashes mid-commit, then restarts

  

Timeline (Real Wall Clock)

═════════════════════════════════════════════════════════════════════════════

  

T=0s

├─ Kafka Consumer polls

│ └─ Receives: [msg0(p5:o10000), msg1(p5:o10001), msg2(p5:o10002)]

│ (3 messages from partition 5, offsets 10000-10002)

│

├─ Processor.bulkProcess()

│ └─ Computes: token = "p5:o10000-10002"

│ └─ Calls: ctx = WithDedupToken(ctx, token)

│

├─ Writer.Write(ctx)

│ └─ Extracts: token from context ✓

│ └─ Calls: writeWithDedupToken(ctx, token, messages)

│

├─ writeWithDedupToken()

│ ├─ Pins connection: conn := db.Conn(ctx)

│ │

│ ├─ Set token: conn.ExecContext("SET insert_deduplication_token=...")

│ │ └─ ClickHouse server stores token in session

│ │

│ ├─ Begin transaction: tx := conn.BeginTx()

│ │ └─ Transaction begins in SAME session

│ │

│ └─ Execute inserts

│ ├─ prepare.Exec(row1) → ClickHouse inserts row1

│ │ └─ ClickHouse records (token, block_hash_1) in dedup table

│ ├─ prepare.Exec(row2) → ClickHouse inserts row2

│ │ └─ ClickHouse records (token, block_hash_2) in dedup table

│ └─ prepare.Exec(row3) → ClickHouse inserts row3

│ └─ ClickHouse records (token, block_hash_3) in dedup table

│

├─ tx.Commit() → rows persisted ✓

│

├─ Reader.CommitMessages() → offset marked as processed

│ └─ ... BUT CRASHES BEFORE COMPLETING!

│

└─ 💥 WORKER CRASHED 💥

  

═════════════════════════════════════════════════════════════════════════════

  

T=10s

├─ Worker restarts

│ └─ Reconnects to Kafka broker

│

├─ Consumer Group rebalancing

│ └─ Consumer requests offset for partition 5

│ └─ Kafka returns: offset 10000 (NOT COMMITTED, so rollback to last commit)

│

├─ Kafka Consumer polls

│ └─ Receives: [msg0(p5:o10000), msg1(p5:o10001), msg2(p5:o10002)]

│ (SAME messages, SAME partition:offset)

│

├─ Processor.bulkProcess()

│ └─ Computes: token = "p5:o10000-10002"

│ (SAME TOKEN! Because same partition:offset)

│

├─ Writer.Write(ctx)

│ └─ Calls: writeWithDedupToken(ctx, token, messages)

│

├─ writeWithDedupToken()

│ ├─ Pins connection: conn := db.Conn(ctx)

│ │

│ ├─ Set token: conn.ExecContext("SET insert_deduplication_token=...")

│ │ └─ ClickHouse server stores token in session

│ │

│ ├─ Begin transaction: tx := conn.BeginTx()

│ │

│ └─ Execute inserts

│ ├─ prepare.Exec(row1) → ClickHouse checks dedup table

│ │ └─ (token, block_hash_1) already exists!

│ │ └─ ❌ SKIPPED (not inserted)

│ ├─ prepare.Exec(row2) → ClickHouse checks dedup table

│ │ └─ (token, block_hash_2) already exists!

│ │ └─ ❌ SKIPPED (not inserted)

│ └─ prepare.Exec(row3) → ClickHouse checks dedup table

│ └─ (token, block_hash_3) already exists!

│ └─ ❌ SKIPPED (not inserted)

│

├─ tx.Commit() → 0 rows inserted (but transaction succeeds!)

│

├─ Reader.CommitMessages() → offset marked as processed

│ └─ ✓ SUCCEEDS this time

│

└─ 🎉 IDEMPOTENCY ACHIEVED 🎉

  

═════════════════════════════════════════════════════════════════════════════

  

VERIFICATION (after T=10s):

─────────────────────────

  

Before crash (T=0-5s):

SELECT count() FROM tiki_ads.events → 3 rows

  

After restart (T=10s):

SELECT count() FROM tiki_ads.events → still 3 rows (not 6!)

  

Conclusion: Same batch processed twice → same final row count

= IDEMPOTENT = SAFE ✓

```

  

---

  

## 6. Error Handling Logic

  

```

┌─────────────────────────────────────────────────────┐

│ BigQuery Write with Error Handling │

└─────────────────────────────────────────────────────┘

  

Worker.writeToBQ(batch of 2K rows)

│

▼

bigquery.Inserter.Put(rows)

│

┌─────────────┴──────────────┐

│ │

✓ nil ❌ error (some or all rows rejected)

│ │

return nil Inspect error type

│ │

offset │

committed ┌─────┴─────────┐

│ │

Type A Type B

(Network, Auth) (PutMultiError)

error rows failed

│ │

return err ┌─────┴──────────┐

│ │ │

offset NOT len(err) == len(batch)

committed (all rows rejected)

│ │

▼ ▼

RETRY return err

on restart (quota exceeded,

schema mismatch,

table locked)

│

offset NOT

committed

│

▼

RETRY

on restart

  

len(err) < len(batch)

(partial failure,

some rows invalid)

│

▼

Log failures:

for _, rowErr := range err {

log(rowErr)

}

│

▼

return nil

(implicit success)

│

▼

offset

committed

(accept loss

of bad rows)

```

  

---

  

## 7. Interview Flow (12-Minute Version)

  

```

╔════════════════════════════════════════════════════════════════════════════╗

║ INTERVIEW PRESENTATION OUTLINE ║

║ (12 minutes total) ║

╠════════════════════════════════════════════════════════════════════════════╣

║ ║

║ [0:00-1:30] PROBLEM STATEMENT ║

║ ─────────────────────────────────── ║

║ • "We observed ~33,000 duplicate event_ids across ClickHouse & BigQuery" ║

║ • Created bimodal distribution query (show the data) ║

║ • Found two different problems: 0s gap and 1h+ gap ║

║ → Shows: investigative thinking, data-driven approach ║

║ ║

├─────────────────────────────────────────────────────────────────────────────┤

║ ║

║ [1:30-3:30] ROOT CAUSE ANALYSIS ║

║ ─────────────────────────────── ║

║ • Kafka at-least-once retries (0s gap) ║

║ └─ Worker crashes mid-commit → batch redelivered ║

║ ║

║ • Bot replay after Redis TTL expires (1h+ gap) ║

║ └─ Redis key expires → bot fires same event_id again ║

║ ║

║ • Silent data loss in BigQuery ║

║ └─ PutMultiError logged but not returned → offset committed anyway ║

║ → Shows: deep understanding of distributed systems ║

║ ║

├─────────────────────────────────────────────────────────────────────────────┤

║ ║

║ [3:30-5:00] WHY event_id DOESN'T WORK ║

║ ──────────────────────────────────── ║

║ • event_id = business key (ad impression) ║

║ • Can be fired multiple times (bot, network retry, user re-engagement) ║

║ • Using it as dedup key → hides fraud, hides infrastructure bugs ║

║ • Need transport-level key: partition:offset (unique, deterministic) ║

║ → Shows: separation of concerns, system design thinking ║

║ ║

├─────────────────────────────────────────────────────────────────────────────┤

║ ║

║ [5:00-7:30] CLICKHOUSE SOLUTION ║

║ ──────────────────────────────── ║

║ • ClickHouse has insert_deduplication_token (but session-level) ║

║ • Show code: context injection pattern ║

║ • Show code: connection pinning (why needed!) ║

║ • Explain: SET and INSERT must share same session ║

║ → Shows: hands-on implementation, SQL session semantics ║

║ ║

├─────────────────────────────────────────────────────────────────────────────┤

║ ║

║ [7:30-9:00] BIGQUERY SOLUTION ║

║ ────────────────────────────── ║

║ • Changed InsertID from event_id to partition:offset ║

║ • Fixed error handling: distinguish all-fail vs. partial-fail ║

║ • Show code: error handling before/after ║

║ • Explain: PutMultiError semantics ║

║ → Shows: attention to error handling, observability ║

║ ║

├─────────────────────────────────────────────────────────────────────────────┤

║ ║

║ [9:00-10:30] TESTING & VERIFICATION ║

║ ──────────────────────────────────── ║

║ • Built local Docker Compose environment ║

║ • Produce test events (6 events: 3 unique, 3 duplicates) ║

║ • Kill worker mid-batch, restart ║

║ • Verify: row count unchanged (idempotent!) ║

║ → Shows: pragmatism, reproducible testing ║

║ ║

├─────────────────────────────────────────────────────────────────────────────┤

║ ║

║ [10:30-12:00] RESULTS & TRADEOFFS ║

║ ───────────────────────────────── ║

║ • 100% dedup of Kafka retries (4,243 event_ids) ║

║ • Bot replays preserved (29,021 event_ids, fraud measurement intact) ║

║ • No silent data loss (error handling fixed) ║

║ • Deferred: ReplacingMergeTree (would dedupe bot replays too) ║

║ └─ But requires schema migration, adds complexity ║

║ → Shows: systems thinking, pragmatic tradeoffs ║

║ ║

╚════════════════════════════════════════════════════════════════════════════╝

```

  

---

  

## 8. Key Numbers to Memorize

  

```

╔══════════════════════════════════════════════════════════════════════╗

║ QUICK REFERENCE NUMBERS ║

╠══════════════════════════════════════════════════════════════════════╣

║ ║

║ DUPLICATES BEFORE FIX: ║

║ • ClickHouse: ~33,264 event_ids ║

║ └─ 4,243 with 0s gap (Kafka retries) ║

║ └─ 25,713 with 1h-1d gap (bot replay) ║

║ └─ 3,308 with 1d-30d gap (bot replay older) ║

║ ║

║ • BigQuery: ~26,494 event_ids ║

║ └─ 2,795 with 0s gap ║

║ └─ 21,026 with 1h-1d gap ║

║ └─ 2,673 with 1d-30d gap ║

║ ║

║ DEDUPLICATION AFTER FIX: ║

║ • 0s gap duplicates: 100% deduped (Kafka retries) ║

║ • 1h+ gap duplicates: Preserved (fraud measurement) ║

║ ║

║ SYSTEM CHARACTERISTICS: ║

║ • Throughput: ~120 events/sec sustained ║

║ • Batch size: 2K messages (tunable) ║

║ • Batch window: 60 seconds (tunable) ║

║ • ClickHouse latency: < 100ms per batch ║

║ • BigQuery latency: ~40 seconds per batch ║

║ • Dedup overhead: < 1ms (negligible) ║

║ ║

║ REDIS TTL (CONTEXT): ║

║ • Repetition detector TTL: 1 hour ║

║ • After expiry: bot can fire same event_id again ║

║ ║

║ BIGQUERY InsertID WINDOW: ║

║ • Dedup window: ~1 minute ║

║ • Perfect for catching Kafka retries (within seconds) ║

║ • Allows bot replays (after 1 min boundary) ║

║ ║

║ CODE CHANGES: ║

║ • Total insertions: 84 lines ║

║ • Total deletions: 7 lines ║

║ • Files modified: 5 ║

║ ║

╚══════════════════════════════════════════════════════════════════════╝

```

  

---

  

## 9. Potential Follow-Up Questions & Answers

  

```

Q: "Why not just use a dedup table / distributed cache?"

A: "We could, but that adds latency (extra read before write) and

complexity (distributed consistency). ClickHouse's built-in token

is simpler, faster, and atomic."

  

Q: "What about clock skew / timestamp ordering issues?"

A: "We don't rely on timestamps for dedup—we use partition:offset,

which is monotonic and issued by Kafka brokers. Timestamps only

matter for data columns (received_time), set by the pixel service."

  

Q: "Why is the INSERT happening, but later being skipped?"

A: "The INSERT statement still runs, but ClickHouse silently skips it

at commit time. This is ClickHouse's dedup mechanism—it checks the

dedup table before committing, not before executing."

  

Q: "What if same event_id gets different row data in the retry?"

A: "Kafka message bytes are identical on retry (deterministic), so

all columns will match. ClickHouse dedup is at block level

(all rows in INSERT), so as long as the JSON is identical, dedup works."

  

Q: "How does this scale to multi-partition Kafka?"

A: "Token is built from first + last message in batch. If batch spans

multiple partitions, token is 'p1:o1000-p2:o2000' or similar.

Scaling is linear (more partitions = more workers)."

  

Q: "What about the ReplacingMergeTree approach?"

A: "It would deduplicate on a key (event_id + zone_id + received_date),

covering both Kafka retries AND bot replays. But it requires:

- Schema migration (downtime)

- FINAL keyword on all queries (slower)

- Background merge timing (eventual consistency)

Current solution is faster, simpler, and good enough for our use case."

  

Q: "How do you know the offset wasn't committed?"

A: "When worker restarts, it asks Kafka for the last committed offset.

If crash happened before commit, Kafka returns the previous offset,

not the new one. Consumer rewinds and redelivers same messages."

  

Q: "What if BigQuery quota is temporarily exceeded, then recovered?"

A: "Our fix returns an error when all rows fail, preventing offset commit.

Worker retries from Kafka. When BQ quota recovers, retry succeeds."

  

Q: "Can you guarantee no duplicates at all?"

A: "We guarantee idempotency for Kafka retries (0s gap). Bot replays

(1h+ gap) are intentional—they represent separate measurement events.

This is correct behavior for fraud detection."

```

  

---

  

## 10. One-Slide Summary (If Time is Short)

  

```

╔════════════════════════════════════════════════════════════════════════════╗

║ ║

║ ClickHouse Kafka Streaming Pipeline ║

║ ║

║ PROBLEM: 33K duplicate events due to at-least-once Kafka retries ║

║ ║

║ ROOT CAUSE: ║

║ • event_id (business key) ≠ partition:offset (transport key) ║

║ • Worker crash mid-commit → batch redelivered → duplicates ║

║ • Silent data loss in BigQuery (PutMultiError not returned) ║

║ ║

║ SOLUTION: ║

║ 1. Use partition:offset as dedup token (deterministic, unique) ║

║ 2. ClickHouse: session-level insert_deduplication_token ║

║ (with connection pinning to ensure SET + INSERT in same session) ║

║ 3. BigQuery: InsertID = partition:offset (instead of event_id) ║

║ + fix error handling (all-fail → return error, partial → log) ║

║ ║

║ RESULT: ║

║ ✓ 100% idempotent (Kafka retries deduped, idempotency guaranteed) ║

║ ✓ Bot replays preserved (fraud measurement intact) ║

║ ✓ No silent data loss (error handling fixed) ║

║ ✓ Minimal code changes (84 lines across 5 files) ║

║ ✓ Fully testable locally (Docker Compose) ║

║ ║

║ IMPACT: ║

║ • Attribution accuracy improved ║

║ • Dashboard consistency (CH ≈ BQ) ║

║ • Safe worker restarts ║

║ • Observable, debuggable pipeline ║

║ ║

╚════════════════════════════════════════════════════════════════════════════╝

```