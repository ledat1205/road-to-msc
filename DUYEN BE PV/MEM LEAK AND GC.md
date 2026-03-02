### 1. Java (Platform Threads + Virtual Threads / Project Loom)

**Common leak scenarios**:

- **Traditional platform threads**:
    - Forgotten thread pools (e.g., ExecutorService created but never shut down) → threads stay alive forever.
    - ThreadLocal variables in thread pools → values retained across tasks → classic memory leak if not cleared (e.g., user context in web apps).
    - Unfinished tasks in ExecutorService → threads blocked/waiting indefinitely.
- **Virtual threads (Java 21+ Loom)**:
    - Much rarer due to low overhead (virtual threads are heap objects collected by GC when no longer referenced).
    - Pinning issues (pre-JDK 24/25 fixes): Long synchronized blocks or native calls pin carrier threads → reduces effective concurrency, but not a true "leak" (fixed in recent JDKs with better pinning avoidance).
    - StructuredTaskScope misuse: Scope not closed/joined properly → tasks keep running, holding references → potential memory retention.
    - ThreadLocal → ScopedValue migration mistakes: Old ThreadLocal code in virtual thread context can cause unexpected retention (Scoped Values are designed to avoid this).
    - Unhandled exceptions or infinite loops in virtual threads → they keep running, holding stack/heap objects until GC (but GC eventually cleans up if no strong references remain).

**When it happens most**:

- Long-running servers with thread pools (Spring Boot, Tomcat).
- Migration from platform to virtual threads without auditing ThreadLocal usage.
- Forgetting to close StructuredTaskScope or join tasks.

**Detection & fix**:

- Use JFR (Java Flight Recorder), VisualVM, or heap dumps to spot retained objects.
- Prefer StructuredTaskScope (JDK 21+) for scoped tasks — it enforces cleanup.
- Migrate ThreadLocal → Scoped Values (designed for virtual threads, no leaks across tasks).

### 2. Kotlin (kotlinx.coroutines)

**Common leak scenarios**:

- **GlobalScope** usage (biggest rookie mistake): GlobalScope.launch { ... } creates coroutines that live for the entire JVM process — they never cancel automatically → zombie coroutines after activity/fragment/viewmodel destruction.
- **Custom CoroutineScope without proper lifecycle**:
    - val scope = CoroutineScope(Dispatchers.Default) in a class → if the class instance is GC-eligible but scope/job not cancelled → coroutines keep running, holding references to captured variables (e.g., large bitmaps, contexts).
    - Scope inheritance issues: Child coroutines retain parent references → memory leak if parent not cancelled.
- **Uncancelled jobs**: launch without storing Job → can't cancel → runs forever (e.g., background polling after screen destroy).
- **Continuation retention bugs** (rare, older versions): In some 1.4–1.6 coroutines versions, suspended continuations held local variables longer than expected → large allocations (e.g., ByteArray) not GC'ed until next suspension.
- **Channel/Flow leaks**: Unclosed channels or infinite flows without collection cancellation.

**When it happens most**:

- Android (ViewModel/Activity leaks via GlobalScope).
- Long-lived services without lifecycle-aware scopes.
- Forgetting withContext, supervisorScope, or proper cancel().

**Detection & fix**:

- Use structured concurrency: coroutineScope { ... }, supervisorScope, viewModelScope, lifecycleScope.
- Never use GlobalScope in production code.
- Store Job and cancel in onCleared() / close().
- Tools: LeakCanary (Android), VisualVM/heap dumps for JVM.

### 3. Go (goroutines)

**Common leak scenarios** (most frequent concurrency bug in Go):

- **Blocked forever on channels**:
    - Send to unbuffered channel with no receiver → sender blocks indefinitely.
    - Receive from channel that never gets sent to or closed → receiver blocks.
    - for range ch {} on never-closed channel → loop hangs.
- **Missing context cancellation**:
    - Goroutine launched with context.Background() (no cancel) → ignores shutdown.
    - HTTP handler spawns goroutine without passing request context → runs after request cancel.
- **Unbounded goroutine creation**:
    - Infinite loop spawning goroutines (e.g., recursive fan-out without bound).
    - Event listeners/callbacks that never deregister.
- **Mutex/condvar deadlocks or permanent waits** → goroutine blocked forever.
- **Forgotten WaitGroup** misuse or missing wg.Done() → wait forever.

**When it happens most**:

- HTTP servers: handlers spawn goroutines without context propagation.
- Long-running workers without shutdown signaling.
- Channel patterns without close or select with default/case.

**Detection & fix**:

- Use context.WithCancel, errgroup, sync.WaitGroup properly.
- Always pass context.Context and respect cancellation.
- pprof: go tool pprof http://localhost:6060/debug/pprof/goroutine — look for growing NumGoroutine().
- Tools: golang.org/x/sync/errgroup, context everywhere.
- Best practice: Bound concurrency (semaphore pattern), close channels, use select with timeouts.

### Quick Summary Table

|Language|Most Common Leak Type|Root Cause Example|How to Avoid / Mitigate|Detection Tool|
|---|---|---|---|---|
|**Java**|ThreadLocal retention, unfinished tasks|Thread pools + ThreadLocal not cleared|Scoped Values, StructuredTaskScope, shutdown pools|JFR, VisualVM, heap dumps|
|**Kotlin**|Zombie coroutines after lifecycle end|GlobalScope, uncancelled jobs|Structured scopes, viewModelScope, cancel jobs|LeakCanary, VisualVM|
|**Go**|Blocked goroutines forever|Send/receive on unclosed/unbounded channels|Context cancellation, errgroup, close channels|pprof goroutine profile|

In production (fintech/backend), **Go leaks** are the most common silent killers (growing goroutine count → OOM), **Kotlin** leaks hit hard in Android/lifecycle code, and **Java** leaks are more "classic" (ThreadLocal) but less severe with virtual threads.


### Comparison Table: Garbage Collection in Python, Java/Kotlin, Go

| Language                | GC Type / Main Mechanism                                                                                                    | Pause Times (Typical)                                                                                    | Throughput Impact                                              | CPU Overhead                                                   | Key Strengths                                                                           | Key Weaknesses / Pain Points                                                                                  | 2025–2026 Updates / Notes                                                                                                                            | Fintech Relevance                                                                                                             |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- | -------------------------------------------------------------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Python (CPython)**    | **Reference Counting** (primary) + **Generational Cyclic GC** (for cycles)                                                  | Usually very short (microseconds for refcount), but full GC cycles can pause 10–100+ ms under heavy load | High allocation/deallocation cost due to refcount ops          | Moderate–high (refcount increments/decrements everywhere)      | Predictable deallocation for refcounted objects; no long random pauses                  | Cycles require full GC (slow); refcounting hurts perf; GIL ties GC to single-thread safety                    | No-GIL (PEP 703, maturing in 3.13–3.14+) uses biased/deferred refcounting → better multi-thread perf, but GC still needed for cycles                 | ML/risk batch jobs: refcounting helps quick cleanup; APIs: full GC can spike latency if many cycles                           |
| **Java / Kotlin (JVM)** | **Tracing GC** (mark-sweep-compact or mark-sweep) with multiple collectors: G1 (default), ZGC, Shenandoah, Parallel, Serial | G1: 10–200 ms pauses; ZGC/Shenandoah: sub-1 ms to ~10 ms (near-pauseless)                                | High throughput with tuning; generational helps young-gen fast | Moderate (stop-the-world phases + concurrent marking)          | Tunable (flags like -XX:+UseZGC); generational + concurrent → excellent for large heaps | GC pauses still possible (even ZGC has rare ms pauses); warmup time; pinning issues (mostly fixed by JDK 24+) | JDK 25/26: better G1/Parallel/Serial; ZGC generational matures; virtual threads reduce pinning impact                                                | Trading/risk engines: ZGC for low-latency; large heaps for in-memory caching; virtual threads help scale without GC thrashing |
| **Go**                  | **Concurrent tri-color mark-sweep** (non-generational, with write barriers)                                                 | Very short STW (~1–10 ms for mark start); full GC usually <10 ms even under load                         | Good throughput; low allocation stall                          | Low–moderate (optimized write barriers); pacer balances GC CPU | Predictable, low pauses; scales with cores; no generational complexity                  | Non-generational → worse for long-lived objects; allocation-heavy code can spike CPU                          | Go 1.25/1.26: "Green Tea" GC (experimental → default in 1.26) → lower latency, 30–40% faster cleanup, reduced CPU overhead via barrier optimizations | Payment gateways, microservices: predictable pauses; high-allocation APIs benefit hugely from Green Tea                       |

### Deeper Explanation of Each

1. **Python (CPython) GC**
    - **Reference Counting** (primary): Every object has a refcount. When it hits 0 → immediate deallocation (no pause). Fast for most cases.
    - **Cyclic GC** (generational): Detects reference cycles (e.g., A points to B, B to A) that refcounting misses. Runs periodically (after N allocations) → can pause the entire process (GIL held).
    - **GIL impact**: Refcounting is non-atomic → GIL protects it. In no-GIL Python (emerging 2026), uses **biased refcounting** (owner thread bias) + **deferred refcounting** → reduces contention.
    - **Real pain**: Full GC cycles under memory pressure → visible pauses in long-running services (e.g., FastAPI with many objects).
    - **Mitigation**: Avoid cycles (use weakref), tune gc.set_threshold(), or use PyPy (better GC).
2. **Java / Kotlin (JVM GC)**
    - **Multiple collectors** (tunable via flags):
        - **G1** (default): Region-based, generational, concurrent marking → good balance.
        - **ZGC / Shenandoah**: Concurrent compaction → sub-millisecond pauses even on terabyte heaps (great for low-latency trading).
        - **Parallel / Serial**: Throughput-focused.
    - **Virtual Threads impact**: Virtual threads are heap objects → GC collects them naturally. Pinning (carrier thread stuck) was reduced/fixed in JDK 24+ (JEP 491). No major GC regression.
    - **2026 reality**: JDK 25/26 improves G1/Parallel/Serial (better startup, new formats); ZGC generational → even lower pauses.
    - **Fintech tuning**: Use -XX:+UseZGC for APIs/gateways; monitor with JFR.
3. **Go GC**
    - **Concurrent tri-color mark-sweep**: Mark phase concurrent (app runs while marking); short stop-the-world (STW) only at start/end.
    - **Pacer**: Dynamically adjusts GC frequency based on live heap growth → aims for ~25% CPU in GC by default (GOGC=100).
    - **Green Tea GC** (introduced experimental in 1.25, default/improved in 1.26): Optimized write barriers → lower CPU overhead (up to 40% less in allocation-heavy code), lower tail latency. Huge win for backend APIs/microservices.
    - **Pain**: Non-generational → long-lived objects scanned every cycle.
    - **Mitigation**: Tune GOGC (lower = more frequent GC, less memory); GOMEMLIMIT for hard caps.

### Quick Fintech Takeaways (2026)

- **Low-latency critical** (HFT gateways, real-time pricing): Go (Green Tea) or Java (ZGC) → sub-10 ms predictable pauses.
- **Large heaps / batch** (risk aggregation, caching): Java/Kotlin (ZGC generational) or Go → scale to GBs without long pauses.
- **Python services**: Use offloading (Celery/Ray) + monitor GC; no-GIL helps mixed workloads but cycles still need tuning.