# TIKI
## Recommendation

| Component                                                                                                                          | Typical Use in Your System                                                               | Frequent Updates?                       | Frequent Retrieval?                           | Best Backend / Store                              | Why Not the Other?                                                                                          |
| ---------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | --------------------------------------- | --------------------------------------------- | ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Flink internal keyed state** (e.g., decaying co-view counts per category, top-K maintenance, windowed aggregations)              | Per-key counters/maps/priority queues during streaming aggregation                       | Yes (every event or window)             | Yes (every update/read in Flink)              | **RocksDB** (Flink's EmbeddedRocksDBStateBackend) | Redis would add network latency, lose exactly-once guarantees, complicate checkpoints                       |
| **Hot / volatile serving data** (e.g., user:recent_cats, promo:pid123 boosts, cat:bestsellers, master:pids mappings for hot items) | Cached in Redis for sub-ms reads by Go serving layer; also for async enrichment in Flink | Yes (real-time promos, session changes) | Extremely high (every recommendation request) | **Redis**                                         | RocksDB is local to Flink tasks → not directly queryable from Go; no native pub/sub, TTL, or sorted set ops |
| **Persistent dimension data** (e.g., master_semantic_relations, category_co_view_relations, master_to_pids)                        | Durable storage, occasional batch updates, fallback reads                                | Low (daily batch or async from Flink)   | Medium (Flink async lookup + Go fallback)     | **ScyllaDB**                                      | Redis is volatile; RocksDB not shared/distributed                                                           |
| **Final serving hot paths**                                                                                                        | Go backend reads for recommendations                                                     | —                                       | Extremely high                                | **Redis** (primary) + Scylla fallback             | RocksDB inaccessible from Go                                                                                |
### Goal
Build a scalable, low-latency recommendation serving system capable of handling 1,000+ requests/sec with end-to-end inference latency reduced from hours (batch-only) to seconds. The system combines daily batch training with real-time streaming updates using Flink, Redis for caching/hot features, ScyllaDB for persistent dimension data, and Druid for real-time rollups of top-sold products.

### Key Constraints & Scale
- ~5 million unique master_ids (semantically equivalent products)  
- ~30 million seller-specific pids  
- Master-to-pid mapping required for final resolution  
- Deduplication at master_id level in final recommendation lists  
- Frequent online updates for volatile signals (promos, user interests, category trends)  
- Cold masters (no recent interactions) must retain old semantic inference results

### 1. Related Products Recommendations (Semantic + Category Expansion)

**Rationale**  
Daily model training produces semantic similarity relations at master_id level. Product-level co-views are too voluminous → shift to category-level co-occurrences (thousands of categories) for dynamic expansion. This enables three complementary signals:  
1. Semantic related masters  
2. Best-sellers in currently viewed category  
3. Best-sellers in co-viewed related categories  

**Data Modeling & Storage**  
- **Semantic Relations** (daily, static-ish):  
  - ScyllaDB:  
    ```cql
    CREATE TABLE master_semantic_relations (
        master_id uuid PRIMARY KEY,
        related_masters frozen<map<uuid, float>>,
        inference_ts timestamp,
        inference_source text
    );
    ```
  - Redis mirror (hot masters): sorted sets `master:sem:mid123` (TTL 24h)  

- **Category Co-View Relations** (real-time, decaying):  
  - ScyllaDB: `category_co_view_relations (cat_id text PRIMARY KEY, related_cats map<text, float>)`  
  - Redis: sorted sets `cat:co_view:electronics` (top-20 related categories)  

- **Master-to-Pid Mapping**:  
  - ScyllaDB: `master_to_pids (master_id uuid PRIMARY KEY, pids list<uuid>, metadata map<uuid, text>)`  
  - Redis: hashes `master:pids:mid123` (hot masters only)  

**Update Strategy for Semantic Relations**  
- Daily batch job (Airflow/Spark):  
  1. Identify "hot" masters (recent interactions from Druid rollup or Flink/Redis active set)  
  2. Compute new embeddings/relations only for hot subset  
  3. Update **only** those rows in ScyllaDB:  
     ```cql
     UPDATE master_semantic_relations
     SET related_masters = {... new map ...},
         inference_ts = now(),
         inference_source = 'daily-batch-v3'
     WHERE master_id = ?;
     ```
  4. Inactive/cold masters are never touched → old frozen map remains indefinitely  
- Serving fallback: if `inference_ts` > 3 months old → log/alert, still serve old map, optionally trigger background refresh  

**Online Category Co-View Updates**  
- Flink job: consumes view events from Kafka  
- Uses **RocksDBStateBackend** for keyed state (MapState or custom top-K structure per category_id)  
- Sliding window (15 min window / 5 min slide) or session window (30 min gap)  
- Decay: score *= 0.95 per window/timer  
- Prune to top-20 via min-heap  
- Sink deltas to Redis (`ZINCRBY`) + async batch to ScyllaDB  

**Serving Logic (Go)**  
1. Fetch user recent views → map to masters/categories  
2. Get semantic relations for masters (Redis first, Scylla fallback)  
3. Expand categories via co-view relations (`ZREVRANGE`)  
4. Merge scores (weighted: semantic 0.6 + co-view boost 0.4)  
5. Deduplicate at master_id level (Go map)  
6. Prune to top-1000 (heap)  
7. Resolve to best pid (price/rating from Redis hash)  
**Latency target**: <15 ms  

### 2. Best-Seller Recommendations in Current / Related Categories

**Rationale**  
Real-time intra-day trends (flash sales, promotions) require fresh top-sold products per category. Use Druid for 5-minute rollups. Expand using current category + weekly co-view related categories + user recent interests (decaying).

**Pipeline**  
- FE/App → Kafka → Flink (enrich) → Druid ingestion  
- Druid: real-time rollup (GROUP BY category, ORDER BY sales DESC, LIMIT 50 per category)  
- Serving query: Druid SQL or API for top-N per category  

**User Recent Interests**  
- Redis scored set: `ZADD user:recent_cats:uid score cat1` (score = timestamp or decaying weight)  
- TTL 1–7 days + periodic prune of low-score entries  
- Used for homepage diversity (no current category)  

**Serving Flow**  
1. Get current category from request  
2. Expand: current + co-view related (Redis) + user recent (Redis)  
3. Multi-query top sellers (Redis cache first, Druid fallback)  
4. Union, deduplicate masters, limit  
**Latency target**: <100 ms (Druid queries dominate)  

### 3. Rerank / Filter Layer

**Rationale**  
Long candidate list (semantic + best-sellers + expanded) must be adapted for business rules: out-of-stock filtering, ad bidding, flash sale boosts, widget-specific prioritization.

**Implementation**  
- Redis hashes: `promo:pid123` with fields (boost, out_of_stock, ad_bid, flash_sale)  
- In Go:  
  - Pipeline HMGET for all candidates  
  - Filter out-of-stock  
  - Apply boosts (multiplicative + additive)  
  - Widget-specific logic (e.g., higher bid weight for sponsored slots)  
  - Final sort + top-K (heap)  
**Latency**: <5 ms for 1000 candidates  

### 4. System-Wide Architecture & Optimizations

- **Streaming Layer**: Apache Flink (RocksDBStateBackend for keyed state, async I/O for Redis/Scylla lookups)  
- **Serving Cache & Hot Features**: Redis Cluster (sharded, TTL, pub/sub for invalidations)  
- **Durable Dimensions**: ScyllaDB (wide-column, shard-per-core, predictable latency)  
- **Real-time Analytics Rollups**: Apache Druid (5-min top-sold per category)  
- **Monitoring**: Prometheus (Flink backpressure, Redis hit rate >95%, Druid query latency)  
- **Micro-batch Experiments**: Airflow orchestrates incremental retraining (e.g., only new/changed products) every 1–4 hours  

### 5. A/B Testing & Validation (Amplitude)

- Variants: baseline batch vs. streaming + category decay vs. different rerank weights  
- Bucketing: random 50/50 or segmented (new vs. returning users)  
- Key metrics: CTR on recommendation widgets, conversion rate, revenue per session  
- Run 1–2 weeks, evaluate statistical significance in Amplitude  

This design balances real-time freshness, scale (30M pids, 5M masters), cost efficiency, and cold-start protection while keeping serving latency low and predictable.


## Analytics Engine





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

