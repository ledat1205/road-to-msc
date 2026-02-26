# TIKI
## Recommendation
## 1. System Goal & Scale

Build a scalable, low-latency recommendation serving system ($<15\text{ms}$ target) capable of handling $1,000+$ requests/sec.

- **Scale:** 5M `master_ids` (semantic products) and 30M `pids` (seller-specific).
    
- **Freshness:** Shift from hours (batch) to seconds (streaming) for user intent and category trends.
    
- **Constraints:** Deduplication at `master_id` level; mandatory "Exactly-Once" consistency for state.
    

---

## 2. Tiered Data Strategy (The "Storage Matrix")

To balance speed and durability, data is partitioned based on access frequency and volatility:

|**Component**|**Storage Backend**|**Logic / Use Case**|
|---|---|---|
|**User Session Intent**|**Flink (RocksDB)**|Decaying category scores based on current clicks.|
|**Hot Serving Data**|**Redis Cluster**|Recent `spids`, category best-sellers, hot `master:pids` maps.|
|**Persistent Dimensions**|**ScyllaDB**|Global semantic relations, historical `reco_by_recency_long`.|
|**Real-Time Analytics**|**Apache Druid**|5-minute rollups for "Top Sold" items per category.|

---

## 3. Real-Time Personalization Pipeline (Flink)

The "Brain" of the system is an Apache Flink job that processes the Kafka clickstream.

### A. Internal Keyed State (The "Why RocksDB" Argument)

We use `EmbeddedRocksDBStateBackend` because:

- **Scale:** Storing intent for millions of active users exceeds RAM; RocksDB spills to local SSD.
    
- **Consistency:** Flink checkpoints RocksDB state and Kafka offsets together, ensuring that if the system crashes, user intent isn't "double-counted" or lost.
    

### B. User Intent & Category Decay

We track what a user is interested in _right now_. If a user searches "Shoes" but clicks "High Heels," we must pivot the recommendations instantly.

- **Logic:** When a user interacts with Category $A$:
    
    1. `Score(A) = Score(A) + 1.0`
        
    2. `Score(Others) = Score(Others) * 0.8` (Decay)
        
- **State Structure:** `MapState<String, Double> userIntent` keyed by `user_id`.
    

---

## 4. Recommendation Logic & Data Modeling

### A. Semantic & Category Expansion

We combine three signals to build the candidate list:

1. **Semantic Similarity:** From ScyllaDB (e.g., "iPhone 15" $\approx$ "Samsung S24").
    
2. **Category Co-Views:** Real-time relations (e.g., People viewing "Laptops" also view "Laptop Sleeves").
    
3. **User Recency:** Items from `reco_by_recency_short` (Last 15 mins).
    

### B. The "Master-to-Pid" Resolution

Since users buy specific `pids` but we recommend `master_ids`, we maintain a mapping:

- **Storage:** Redis Hash `master:pids:{mid}`.
    
- **Fields:** `price`, `rating`, `stock_status`, `seller_id`.
    
- **Benefit:** Allows the Go layer to pick the "best" PID (cheapest or highest rated) for a recommended Master ID.
    

---

## 5. Serving Layer (Go Backend)

The Go layer performs the final "Rerank & Filter" in under $15\text{ms}$.

1. **Parallel Fetch:**
    
    - Fetch `UserIntent` (Redis).
        
    - Fetch `RecentSPIDs` (Redis).
        
    - Fetch `LongTermCandidates` (ScyllaDB/Redis).
        
2. **Scoring & Bias:**
    
    - $FinalScore = BaseScore \times IntentScore_{category}$.
        
    - Items in a "decayed" category (low intent score) are pushed to the bottom.
        
3. **Business Filtering (Redis Hash Lookups):**
    
    - Check `promo:pid123` for `out_of_stock` or `ad_boost`.
        
4. **Deduplication:** Use a Go `map` to ensure only one PID per `master_id` is shown.
    

---

## 6. Maintenance & Experiments (Amplitude/Airflow)

- **Savepoints:** Used for manual updates. Before deploying new Go or Flink code, a Savepoint is taken to ensure the "User Intent" state survives the migration.
    
- **A/B Testing:** Variants are bucketed in Amplitude (e.g., 50% see "High Decay" vs. 50% "Low Decay").
    
- **Recovery:** If Redis fails, the system falls back to ScyllaDB for `reco_by_recency_long`.

## Analytics Engine
### Why We Had to Change

The old GCP setup worked well until mid-2024, when several pain points became business blockers:

- **Latency unacceptable for business users** Marketing and growth teams needed to see campaign performance and user funnel changes within minutes — not hours. Daily/hourly batches meant reacting to yesterday’s problems today.
- **Cost scaling poorly** As daily events crossed 1.5–2 million, BigQuery storage + query costs grew faster than revenue. We were paying a premium for managed convenience we no longer needed at full price.
- **Limited tuning surface** High-cardinality dimensions (user_id + device + campaign_id + geo + …) caused query explosions and slot contention. We couldn’t fine-tune partitioning, rollups, or ingestion compression the way we needed.

### Simple Example: E-commerce Analytics

Imagine a table with ~2 billion rows of user events:

|event_time|user_id|device_type|campaign_id|geo_country|action|revenue|
|---|---|---|---|---|---|---|
|2026-01-27 10:00|uuid-1234-abcd|mobile|camp-XYZ|VN|add_to_cart|0|
|2026-01-27 10:01|uuid-5678-efgh|desktop|camp-ABC|US|purchase|49.99|
|... (billions more rows)|...|...|...|...|...|...|

Typical cardinalities in your data:

- user_id: **extremely high** (~ tens to hundreds of millions unique users) → very high cardinality
- geo_country: **low** (~200–250 unique countries)
- campaign_id: **medium-high** (~10k–500k campaigns running at once)
- device_type: **very low** (~5–10 values: mobile, desktop, tablet, …)
- action: **low** (~10–20: view, add_to_cart, purchase, …)

#### Bad (Explosive) Query – Causes "query explosion" + slot contention

SQL

```
-- Druid or ClickHouse style
SELECT
  user_id,
  geo_country,
  campaign_id,
  device_type,
  COUNT(*) AS events,
  SUM(revenue) AS total_revenue
FROM user_events
WHERE event_time >= '2026-01-01'
GROUP BY user_id, geo_country, campaign_id, device_type
```

**What happens internally?**

- The GROUP BY key has **cardinality ≈ #unique users × #countries × #campaigns × #devices** → Potentially **hundreds of millions to billions** of unique combinations (even if many are sparse).
- Druid must build huge hash tables / aggregation buffers per segment → memory spikes, segments fan out across many Historical nodes → query reads from hundreds/thousands of segments → **slot contention** on brokers.
- ClickHouse builds massive aggregation states in memory → can hit max_rows_to_group_by limit or OOM if not using GROUP BY with SETTINGS group_by_two_level_threshold.
- Result: Query takes 30–120+ seconds, consumes huge RAM/CPU, blocks other queries (contention), or fails.

#### Good (Controlled) Version – Much Faster, Less Contention

SQL

```
-- Better approach
SELECT
  geo_country,              -- low cardinality first
  device_type,
  campaign_id,
  COUNT(DISTINCT user_id) AS unique_users,   -- approximate with HLL if exact not needed
  COUNT(*) AS events,
  SUM(revenue) AS total_revenue
FROM user_events
WHERE event_time >= '2026-01-01'
  AND campaign_id IN ('camp-XYZ', 'camp-ABC')   -- strong early filter
GROUP BY geo_country, device_type, campaign_id
```

**Why this is better:**

- Grouping on **low-to-medium cardinality** columns first → fewer unique groups (e.g., 250 countries × 10 devices × 100 active campaigns = ~250k groups instead of billions).
- Early filtering on campaign_id skips most data.
- Using **approximate distinct** (Druid's APPROX_COUNT_DISTINCT_DS_THETA(user_id) or ClickHouse uniqCombined64(user_id)) avoids materializing every user_id.
- Result: Query runs in 1–5 seconds, reads far fewer segments/parts, uses 5–20× less memory, leaves slots free for other users.

We decided to keep two complementary engines:

- **Druid** — for time-series aggregations with high-cardinality dimensions (campaign performance, user segments, revenue attribution)
- **ClickHouse** — for fast ad-hoc exploration of raw-ish logs and very wide tables (user journeys, affiliate deep dives)

Both would ingest from the same Kafka topics in real time.

### New Architecture at a Glance

```
[Web/App/Backend Services]
         ↓ (producers)
      Kafka (self-hosted on FPT VMs)
         ↓ (exactly-once consumers)
   ┌───────────────┴───────────────┐
   │                               │
Druid (on Kubernetes)         ClickHouse (on VMs)
   │                               │
deep storage (S3-compatible)   replicated MergeTree tables
   │                               │
 Superset / internal BI tools   Superset / internal BI tools
```

Key design decisions made early:

- Kafka as single source of truth with 7-day retention
- Dual ingestion: most high-value streams go to both Druid and ClickHouse (different use-cases)
- Batch producers at source: 500–1,200 events per Kafka message (Protobuf batches or RawBLOB → RowBinary)
- Exactly-once semantics wherever possible
- Multi-AZ / multi-rack replication from day one

### How We Converted Batch → Streaming

Most pipelines followed this transformation pattern:

**Old (GCP batch)**

text

```
Log files / events → GCS → Dataflow (hourly) → BigQuery table
→ Airflow DAG → materialized views / scheduled queries
```

**New (FPT streaming)**

text

```
Event producers (Node.js / Go / Python services)
  ↓ batch 500–1200 rows, serialize → Protobuf / RowBinary
Kafka topic (partitioned by event_type + date_bucket)
  ↓ Kafka Indexing Service (Druid) / Kafka Engine + Materialized View (ClickHouse)
Druid segments / ClickHouse parts
  ↓ background compaction / merges
Query layer (Superset, internal API)
```

The single biggest throughput win came from **batching at the producer** and switching to **RawBLOB → RowBinary** on ClickHouse side (inspired by Mux’s public optimization work). Parsing individual messages killed us; moving parsing to materialized views after ingestion gave us an order-of-magnitude improvement.

### Hardest Technical Challenges & Solutions

#### 1. Ingestion backpressure & consumer lag during traffic spikes

- **Symptom**: Kafka consumer lag → 10–30 min during flash sales / big campaigns
- **Root cause**: ClickHouse insert thread saturation + merge queue backlog; Druid task queue overflow
- **Fixes applied**
    - Increased kafka_flush_interval_ms to 2000–3000 ms
    - Switched ClickHouse to RawBLOB + deferred parsing in MV
    - Added consumer-side sampling (drop debug-level events when lag > 5 min)
    - Implemented dynamic consumer scaling (Keda-like on K8s for Druid)
    - Result: lag rarely exceeds 60 seconds even at 2.5× peak

#### 2. High-cardinality explosion killing Druid compaction & query performance

- **Symptom**: Segments refused to compact → storage ballooned; P95 query latency > 8s
- **Fixes**
    - Aggressive rollup on ingestion (drop unnecessary dimensions for revenue & ads tables)
    - hash partitioning strategy + fixed numShards=80–120 on high-cardinality sources
    - Separate data sources for “hot” (last 7 days) vs “cold” (historical)
    - Query laning: dedicated broker threads for sub-second real-time queries
    - Result: compaction success rate > 98%, P95 dropped to ~1.8 s

#### 3. ClickHouse merge & CPU contention during heavy concurrent inserts + queries

- **Symptom**: background merges starved → part explosion → insert latency spikes
- **Fixes**
    - Increased background_pool_size and merge_tree settings carefully
    - Moved heavy ad-hoc exploration queries to readonly replicas
    - Implemented circuit-breaker pattern in API layer (fallback to aggregated view when P99 > 800 ms)
    - Result: stable insert latency < 300 ms even under mixed workload

#### 4. Cost control while scaling

- **Symptom**: initial FPT footprint was almost as expensive as GCP
- **Fixes**
    - Vertical right-sizing (moved from 64→48 GB nodes after profiling)
    - Storage tiering: hot SSD (last 30 days), cold HDD/S3 (older)
    - Compression codecs tuned per table (ZSTD vs LZ4 vs Gorilla)
    - Druid tiered storage + aggressive segment drop policies
    - Result: 30% net reduction (20% storage, 10% compute)

### Results – Quantified

|Metric|Before (GCP batch)|After (FPT streaming)|Improvement|
|---|---|---|---|
|Daily events processed|~1.4–1.8B|~2.0–2.5B (growing)|+40%|
|Query throughput (concurrent)|~90–120 qps|~280–350 qps|3×|
|P95 end-to-end query latency|6–12 s|1.5–2.2 s|~5× faster|
|Infrastructure cost (monthly run-rate)|Baseline|-30%|-30%|
|Uptime (last 12 months)|99.7%|99.95%|significantly higher|

Business impact was felt quickly:

- Marketing could pause under-performing campaigns within 10–15 minutes
- Revenue & fraud teams got near-real-time anomaly detection
- Affiliate reporting latency dropped from hours to seconds

### Lessons Learned – What I Would Do Differently

1. **Start measuring cost per query early** — not just total spend.
2. **Invest heavily in producer-side batching & format choice** — it’s the cheapest place to win throughput.
3. **Dual engine is powerful but expensive** — if I had to pick one today I’d lean ClickHouse for most e-commerce use-cases (faster ad-hoc, simpler operations).
4. **Chaos test from week 1** — we found many replication & failover edge cases only after injecting failures.
5. **Document ingestion specs religiously** — schema evolution across Kafka → Druid/ClickHouse is painful without it.

# TMA

## Speech to Text

### 1. System Topology

The system is divided into an Asynchronous Media Bridge that acts as the traffic controller between the field devices and the AI models.

- Ingress (MQTT): The wearable device publishes 200ms-500ms audio chunks (PCM 16kHz) to a unique topic: wearable/{device_id}/audio.
    
- The Bridge (C++): A multi-threaded service that subscribes to MQTT and pipes those chunks into a gRPC bidirectional stream.
    
- Egress (gRPC): The AI Service (Faster-Whisper) receives the stream and returns transcribed text segments.
---
### 2. Implementation Guidelines (Local Reproduction)

#### Step 1: Environment & Prerequisites

You need a C++20 build environment.

- Hardware: NVIDIA GPU with CUDA 12.x and cuDNN 8.x.
    
- Libraries: * CTranslate2 (Inference Engine)

- Paho MQTT C++ (Client for audio ingress)
    
- gRPC / Protobuf (Service communication)
    
- FFmpeg (Audio resampling if the wearable doesn't send 16kHz)
    

#### Step 2: Model Conversion (The "Artifact" Phase)

Convert the Whisper model into the CTranslate2 format to enable C++ inference.

##### Convert to int8 for CPU or float16 for GPU

```
ct2-transformers-converter --model openai/whisper-medium \

    --output_dir whisper-medium-ct2 --quantization int8
```

#### Step 3: Implement the C++ Media Bridge (Logic)

The core of your TMA project was the data handoff. You must handle the Asynchronous MQTT callback and forward data to the gRPC Stream.

##### A. MQTT Audio Callback (Paho C++):

```
void on_message(mqtt::const_message_ptr msg) override {

    // msg->get_payload() contains the raw audio bytes

    // Forward these bytes to the active gRPC stream for this device_id

    grpc_stream_map[device_id]->Write(audio_proto_packet);

}
```

##### B. Inference Code (CTranslate2):

```
#include <ctranslate2/models/whisper.h>


// Load model

ctranslate2::models::WhisperModel model("whisper-medium-ct2", ctranslate2::Device::CUDA);


// Transcription with 'initial_prompt' hint

auto options = ctranslate2::models::WhisperOptions();

options.initial_prompt = "Emergency, Help, Alpha-7, Warehouse-B"; // Contextual hints

  
auto result = model.generate(audio_features, options);

std::string text = result.segments[0].text;

```

#### Step 4: The Feedback Path (TTS)

Use **Piper** for the voice response. Since Piper is an ONNX-based C++ engine, it is significantly faster than standard Python TTS.

```
# Example of returning a voice confirmation
echo "Confirmed, calling support" | ./piper --model voice.onnx --output_file out.wav
```

---
### 4. Intent Recognition - BERT-Tiny
#### 1. Updated Implementation Logic (The Decision Flow)

In an industrial environment using smartwatches, the C++ backend must distinguish between **Urgent Commands** (latency-critical) and **Informational Queries** (knowledge-based).

1. **Preprocessing:** Convert the transcribed string from the watch's microphone into `input_ids` and `attention_mask` using a C++ WordPiece tokenizer.
    
2. **Inference:** Run the BERT-Tiny model via **ONNX Runtime (ORT)** on the edge server or the wearable gateway.
    
3. **Threshold Routing:**
    
    - **High Confidence (> 0.85) & Command Intent:** Direct execution via Team/VoIP API (e.g., "Call Maintenance").
        
    - **Low Confidence or "General" Intent:** Route to the Cloud LLM/RAG system (e.g., "What is the policy for hazardous spills?").
        

#### 2. Why BERT-Tiny for Wearables?

In a noisy factory or warehouse, speed and reliability are non-negotiable.

- **Model Efficiency:** `BERT-Tiny` (L=2, H=128) maintains a tiny memory footprint suitable for edge gateways or even high-end wearable SOCs.
    
- **Latency:** Inference takes **<10ms**, ensuring that a "Summon Team" command is processed before the worker even lowers their arm.
    
- **Domain-Specific Robustness:** By fine-tuning on industrial jargon, the model handles "slang" or "shortcuts" better than a general-purpose LLM.
    
    - _Input:_ "Get the floor lead here now."
        
    - _Intent:_ `SUMMON_TEAM` (Confidence: 0.98).
        

---

#### 3. Step-by-Step Technical Implementation

##### Step A: Fine-Tuning for Wearable Actions

The model is trained on specific intents relevant to team coordination and policy queries.

```
# Updated industrial wearable dataset
data = [
    ("Call technician Nguyen", "DIAL_PERSON"),
    ("Need a supervisor at loading dock 4", "SUMMON_TEAM"),
    ("Dial the safety officer", "DIAL_PERSON"),
    ("Summon cleaning crew to aisle 5", "SUMMON_TEAM"),
    ("What is the PPE policy for zone B?", "GENERAL_QUERY"), # Triggers LLM
    ("How do I report a broken scanner?", "GENERAL_QUERY")    # Triggers LLM
]
```

##### Step B: C++ Inference & Team Routing

The C++ bridge interprets the BERT output to trigger the correct communication protocol (VoIP, MQTT, or HTTP).

```
#include <onnxruntime_cxx_api.h>

// Logic to handle the recognized intent
void handleIntent(int intent_id, float probability, std::string original_text) {
    if (probability < 0.80 || intent_id == INTENT_GENERAL_QUERY) {
        // Route to RAG / Cloud LLM for policy answers
        forwardToCloudLLM(original_text);
    } else {
        // Execute instant local actions
        switch(intent_id) {
            case INTENT_SUMMON_TEAM:
                mqtt_client.publish("team/summon", "{\"location\": \"Zone_A\"}");
                break;
            case INTENT_DIAL_PERSON:
                voip_engine.initiateCall(extractName(original_text));
                break;
        }
    }
}
```

---

#### 4. Integration Architecture: From Wrist to Action

To succeed in your VinRobotics interview, emphasize how this fits into a **Hybrid Edge-Cloud** architecture:

| **Component**          | **Responsibility**                                | **Environment**   |
| ---------------------- | ------------------------------------------------- | ----------------- |
| **Wearable (Watch)**   | VAD (Voice Activity Detection) & Audio Streaming. | Edge (Wrist)      |
| **Media Bridge (C++)** | Whisper (STT) + BERT-Tiny (Intent Recognition).   | Local Edge Server |
| **Robot/Team API**     | VoIP calls, MQTT alerts, summoning bots.          | Local Network     |
| **Cloud LLM (RAG)**    | Complex policy questions & HR documentation.      | Private Cloud     |

> **Interview Tip:** If asked about "Noise," explain that BERT-Tiny's primary job here is to act as a **semantic filter**. Even if Whisper makes a typo (e.g., "Dial John" vs "File John"), the fine-tuned BERT model recognizes the semantic context of "John" as a `DIAL_PERSON` intent.


### 5. Overall Production Architecture (Hybrid Edge-Cloud)

Here's a **focused, concrete deployment guide** for your multi-tenant voice command system (C++ Media Bridge + Faster-Whisper STT + BERT-Tiny intent routing + Piper TTS feedback). As the solution provider in HCMC, you own the centralized cloud backend while each client (different companies) runs their own **edge deployment** at their factory/warehouse site.

The deployment splits into two clearly separated parts:

- **Client Edge** (per-company, on-prem or client VPC) → real-time, latency-critical.
- **Your Cloud** (centralized, multi-tenant) → shared intelligence for general queries.

#### 1. Client Edge Deployment (On-Prem / Client Site)

Each client gets a **standardized Helm chart** from you that deploys the full real-time stack on their local NVIDIA-equipped server (e.g., Jetson AGX Orin, Dell PowerEdge with RTX A6000, or industrial PC).

**Target Hardware per Site**:

- 1–2 servers with NVIDIA GPU (CUDA 12.x)
- Ubuntu 22.04 / 24.04 LTS
- Reliable Wi-Fi 6 or private 5G for wearables

**Helm Chart Structure** (you build & distribute this):

- Namespace: voice-edge
- Deployments:
    - Mosquitto / EMQX MQTT broker (persistent volume for retained messages)
    - Your C++ Media Bridge pod (multi-threaded, subscribes to wearable/*/audio)
    - Faster-Whisper inference sidecar or integrated (CTranslate2 model loaded on GPU)
    - BERT-Tiny ONNX Runtime inference (tiny model, CPU or GPU)
    - Piper TTS binary/container for local voice feedback
- ConfigMap: tenant_id, initial_prompt ("Emergency, Help, Warehouse-B, Alpha-7"), device mappings
- Secrets: MQTT credentials, cloud API key (for routed queries)
- Service: internal ClusterIP for gRPC between bridge & inference

**Deployment Steps for Client** (you guide them or remote assist):

1. Install NVIDIA drivers + CUDA toolkit.
2. Install Helm + kubectl (or use k3s lightweight Kubernetes).
3. helm repo add your-repo https://charts.yourdomain.vn
4. helm install voice-edge your-repo/voice-edge --set tenantId=companyA --set cloudEndpoint=https://api.yourdomain.vn/query
5. Verify: kubectl logs -f deploy/bridge shows MQTT connected and model loaded.

**Visual: Typical Edge Deployment Layout** This shows industrial IoT edge gateway setup with local processing — matches your MQTT + C++ bridge + GPU inference.

![IoT edge computing - what it is and how it is becoming more ...](https://iot-analytics.com/wp-content/uploads/2020/11/Siemens-Industrial-Edge-min.png)

[iot-analytics.com](https://iot-analytics.com/iot-edge-computing-what-it-is-and-how-it-is-becoming-more-intelligent/)

![Edge-Cloud Architecture in Distributed System - GeeksforGeeks](https://media.geeksforgeeks.org/wp-content/uploads/20240606183423/Edge-Cloud-Architecture-in-Distributed-System-image.webp)

[geeksforgeeks.org](https://www.geeksforgeeks.org/system-design/edge-cloud-architecture-in-distributed-system/)

![What Is an Industrial IoT Gateway? (Edge vs Cloud for ...](https://ncd.io/wp-content/uploads/2025/12/ncd_edge_computing.png)

[ncd.io](https://ncd.io/blog/what-is-an-industrial-iot-gateway-edge-vs-cloud-for-manufacturing/)

Helm chart examples (real-world packaging for edge apps):

![How to Use Helm for Frontend Kubernetes Deployments](https://blog.pixelfreestudio.com/wp-content/uploads/2024/08/HelmKubernetesDistro-1024x637.jpg)

[blog.pixelfreestudio.com](https://blog.pixelfreestudio.com/how-to-use-helm-for-frontend-kubernetes-deployments/)

![Helm 3: A More Secured and Simpler Kubernetes Package Manager](https://cdn.prod.website-files.com/5d2dd7e1b4a76d8b803ac1aa/5e2fcac5e5bd94397df70571_Helm%20charts.png)

[velotio.com](https://www.velotio.com/engineering-blog/helm-3)

#### 2. Your Centralized Cloud Deployment (Multi-Tenant Backend)

You run **one Kubernetes cluster** (recommended: GKE in asia-southeast1 or FPT Cloud / Viettel Cloud in VN for <50ms latency from HCMC).

**Cluster Setup**:

- GKE Autopilot or Standard with GPU node pool (NVIDIA A100/H100 or A10G).
- Enable Workload Identity + VPC-native.
- Install cert-manager, external-dns, ingress-nginx.

**Multi-Tenancy Isolation** (namespace-based — simplest & sufficient for most):

- Create namespace per client: company-a, company-b, etc.
- ResourceQuota per namespace (e.g., limit CPU/GPU/memory per tenant).
- NetworkPolicy: deny-all + allow only from ingress & monitoring.
- Separate PostgreSQL schemas or Weaviate collections for per-tenant RAG (company policies, HR docs).

**Core Cloud Components**:

- **Inference Gateway** (custom Go/Envoy or KServe): receives gRPC/HTTPS from edges, routes based on X-Tenant-ID.
- **LLM/RAG Service** (vLLM or TGI for Llama-3/Gemma): pooled inference with tenant context injection.
- **Vector DB**: Weaviate or PGVector — one instance, multi-collection.
- **Monitoring**: Prometheus + Grafana (per-namespace dashboards).

**GPU Sharing in Cloud** (critical for cost): Use **NVIDIA MIG** to partition physical GPUs into isolated instances — one A100/H100 serves 4–7 tenants concurrently.

![Multi-Instance GPU (MIG) | NVIDIA](https://www.nvidia.com/content/nvidiaGDC/us/en_US/technologies/multi-instance-gpu/_jcr_content/root/responsivegrid/nv_container_1533940318/nv_image.coreimg.100.1070.jpeg/1742146768038/hopper-mig-h100-ari.jpeg)

[nvidia.com](https://www.nvidia.com/en-us/technologies/multi-instance-gpu/)

![Getting most out of your GPUs using MIG — AI Infrastructure Leader ...](https://rajatpandit.com/_astro/multi-instance-gpu-tech-works-1cc-d-2x.BB78KDUm.png)

[rajatpandit.com](https://rajatpandit.com/getting-most-out-of-your-gpus/)

**Namespace-based Multi-Tenancy Patterns** (exactly what you need for SaaS backend):

![Implement multitenant SaaS on Kubernetes | Red Hat Developer](https://developers.redhat.com/sites/default/files/ci.png)

[developers.redhat.com](https://developers.redhat.com/articles/2022/08/12/implement-multitenant-saas-kubernetes)

![Designing Multi-Tenant Applications on Kubernetes | by Het Trivedi ...](https://miro.medium.com/v2/resize:fit:1400/1*HVpL4ERyeRha_vrUv5w0qA.png)

[medium.com](https://medium.com/@het.trivedi05/designing-multi-tenant-applications-on-kubernetes-f0470f8e641c)

![Practices of Kubernetes Multi-tenant Clusters - Alibaba Cloud ...](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQuXHXObgNuZptn-eezuMWpO10viWWOvR1urSbN3IxXZBoi6L0&s)

[alibabacloud.com](https://www.alibabacloud.com/blog/practices-of-kubernetes-multi-tenant-clusters_596178)

**GitOps for Updates** (zero-downtime model pushes to cloud & edges): Use ArgoCD — your Git repo is the source of truth for Helm values, manifests, model versions.

![⎈ A Hands-On Guide to ArgoCD on Kubernetes — PART-1 ⚙️ | by ...](https://miro.medium.com/1*63BNapaOyuP8hyA_NvDP7g.jpeg)

[medium.com](https://medium.com/@muppedaanvesh/a-hands-on-guide-to-argocd-on-kubernetes-part-1-%EF%B8%8F-7a80c1b0ac98)

![GitOps and Kubernetes: CI/CD for Cloud Native applications](https://blog.sparkfabrik.com/hubfs/Blog/cicd-push-based-deployment.png)

[blog.sparkfabrik.com](https://blog.sparkfabrik.com/en/gitops-and-kubernetes)

#### 3. Hybrid Edge-Cloud Connection (Real Flow)

Edge detects general query → sends text + tenant_id + context to your cloud API.

![Transforming industrial automation: voice recognition control via ...](https://media.springernature.com/lw685/springer-static/image/art%3A10.1038%2Fs41598-024-81172-w/MediaObjects/41598_2024_81172_Fig3_HTML.png)

[nature.com](https://www.nature.com/articles/s41598-024-81172-w)

![Quantifying the Benefits of AI in Edge Computing - SemiWiki](https://semiwiki.com/wp-content/uploads/2020/07/Architectures-for-Edge-computing-min-1024x697.png)

[semiwiki.com](https://semiwiki.com/eda/synopsys/288658-quantifying-the-benefits-of-ai-in-edge-computing/)

This keeps urgent commands fully local while offloading complex queries securely.

**VN-Specific Tips** (Jan 2026 context):

- Latency: GKE Singapore ≈40–70ms; Viettel/FPT VN regions ≈10–30ms.
- Cost: Start with 1× A100 node (~$3–4/hr on-demand), MIG → serve 5–10 clients.
- Compliance: Audio stays at edge; only text to cloud → easier PDPA alignment.

