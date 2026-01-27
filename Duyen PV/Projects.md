# TIKI
## Recommendation

This document outlines the detailed technical implementation and optimizations for an e-commerce product recommendation system, leveraging Apache Flink for streaming processing, Redis for low-latency caching, ScyllaDB for durable storage, Apache Druid for real-time analytics rollups, and a Go backend for serving. The system handles high concurrency (1,000 req/sec), reduces latency to seconds, and incorporates hybrid batch/streaming with daily training. Key features include semantic product relations, category-based expansions, best-seller rankings, reranking/filtering, and user interest decay.

The design accounts for an Amazon-like structure: ~5M master_ids (unique products) and ~30M seller-specific pids. Mappings between master_id and pid are maintained for resolution, with deduplication at master_id level to ensure clean recommendation lists.

#### 1. **Optimizing Related Products Recommendations (Model-Inferred, Up to 1000 per Product)**
- **Rationale**: Daily training outputs semantic similarities (from titles/embeddings) at master_id level. For dynamic signals, shift from product-level co-views (too voluminous at 30M pids) to category-level co-occurrences (thousands of categories, more tractable). This precomputes relations for fast serving, avoiding on-the-fly DB scans, while enabling expansion via co-viewed categories.
- **Technical Implementation**:
    - **Data Modeling** (Separate Semantic and Category Co-View):
        - **Semantic Relations** (Static, Daily): Master-level for efficiency.
            - ScyllaDB Schema: `CREATE TABLE master_semantic_relations (master_id uuid PRIMARY KEY, related_masters frozen<map<uuid, float>>);` — Top-1000 with similarity scores.
            - Redis: Sorted sets `ZADD master:sem:mid123 score1 related_mid1 score2 related_mid2` (TTL=1 day).
        - **Category Co-View Relations** (Dynamic, Real-Time): Pairwise co-occurrences from sessions.
            - ScyllaDB: `CREATE TABLE category_co_view_relations (cat_id text PRIMARY KEY, related_cats map<text, float>);` — Scores for top-20 related categories.
            - Redis: Sorted sets `ZADD cat:co_view:electronics score1 phone_cases score2 chargers`.
        - **Master-to-Pid Mapping**: For resolving recs to seller pids.
            - ScyllaDB: `CREATE TABLE master_to_pids (master_id uuid PRIMARY KEY, pids list<uuid>, metadata map<uuid, text>);` — Includes price/seller data.
            - Redis: Hashes `HMSET master:pids:mid123 pid1:price1:seller1 pid2:price2:seller2` (for hot masters).
    - **Batch Loading**: Daily Spark/Airflow job computes embeddings (e.g., FAISS ANN), loads semantic relations to Scylla/Redis. Bootstrap category co-views from logs if needed.
    - **Online Updates** (Category Co-View Only): Flink consumes view events (Kafka), enriches with categories.
        - Session Windows: 30-min inactivity gap or sliding (15-min window, 5-min slide).
        - Aggregation: Key by category, maintain map<related_cat, score> with decay (0.95 per window).
        - Prune: Top-20 per category via min-heap in state.
        - Writes: Redis `ZINCRBY`, async Scylla batch.
        - Scale: RocksDB state backend; tiny footprint with thousands of categories.
    - **Serving in Go**: Merge semantic + category-expanded lists with dedup.
        - Fetch recent views (Redis `LRANGE user:views:uid 0 19`), map to masters/categories.
        - Get semantic for masters; expand categories via co-view `ZREVRANGE`.
        - Merge scores (weighted: semantic 0.6 + co-view boost 0.4), dedup masters (map), prune top-1000 heap.
        - Resolve pids (best by price/rating), fallback Scylla.
    - **Latency Impact**: <15ms total (Redis multi-get + in-memory merge).
    - **Code Snippet (Flink for Category Co-View Updates)**:
        ```java
        // Flink: Aggregate category co-occurrences
        DataStream<ViewEvent> views = ...; // Kafka, category-enriched
        views.keyBy(event -> event.getCategoryId())
            .window(SlidingEventTimeWindows.of(Time.minutes(15), Time.minutes(5)))
            .aggregate(new CategoryCoViewAggregator()) // Map accumulator, increment/decay pairs
            .addSink(new RedisCategorySink() {
                public void invoke(CategoryRelations value) {
                    Pipeline pipe = jedis.pipelined();
                    value.related.forEach((relCat, score) -> 
                        pipe.zadd("cat:co_view:" + value.catId, score, relCat));
                    pipe.sync();
                    // Async Scylla upsert
                }
            });
        ```
    - **Trade-offs**: Category-level loses product granularity but scales better; hybrid possible for hot items.

#### 2. **Optimizing Best-Seller Recommendations in Current User Categories (1-10 Categories, Dynamic)**
- **Rationale**: Categories are dynamic per session; best-sellers reflect real-time sales/views (intra-day trends like flash sales). Use Druid for 5-min rollups of top sold products per category. Expand via current viewed category + co-view related categories (weekly updated). For homepage diversity, incorporate user's recent interested categories with time-based decay.
- **Technical Implementation**:
    - **Real-Time Aggregation**: Orders/views from FE/app → Kafka → Flink (enrichment) → Druid for ingestion.
        - Druid: Real-time rollups every 5 min (e.g., GROUP BY category, ORDER BY sales DESC, LIMIT 50 per category). Use Druid's Kafka supervisor for streaming ingestion.
        - Query: Serving layer (Go) calls Druid API/SQL for top products: `SELECT pid, sales FROM best_sellers WHERE category = ? AND time > NOW() - INTERVAL '1' HOUR ORDER BY sales DESC LIMIT 50`.
    - **Category Expansion**:
        - Current: From request body (viewed product's category ID).
        - Related: Co-view categories (from section 1/3, updated weekly via batch job analyzing session logs).
        - User's Recent Interests: Redis set `SADD user:recent_cats:uid cat1 cat2` (TTL=1-7 days, decay by score or expiry; updated on views/orders). For homepage (no specific view), use this for diverse recs.
    - **3 Ways to Generate Related Products**:
        1. Semantic inferred list (from section 1).
        2. Top sold in current/viewed category (Druid query).
        3. Top sold in co-view related categories (multi-query Druid, union results).
    - **Storage**: Redis sorted sets `ZADD cat:bestsellers:cat1 salesScore1 pid1 salesScore2 pid2` (synced from Druid via Flink sink or periodic job). Scylla for backups.
    - **Serving**: In Go, fetch categories (request + Redis user set + co-view expansion), multi-query Druid/Redis for best-sellers, union/dedup masters.
    - **Dynamic Updates**: Flink detects category changes from events, expires old user sets. Decay user interests: Use scored sets `ZADD user:recent_cats:uid score cat1` (score = timestamp, prune low scores).
    - **Latency Impact**: Druid queries <50ms; Redis fallback <2ms; total <100ms.
    - **Code Snippet (Go Serving Example)**:
        ```go
        func GetBestSellers(ctx context.Context, userID, currentCat string) ([]ProductID, error) {
            // Get expanded cats: current + co-view + user recent
            coViewKey := "cat:co_view:" + currentCat
            coViews, _ := redisClient.ZRevRangeWithScores(ctx, coViewKey, 0, 19).Result()
            userKey := "user:recent_cats:" + userID
            userCats, _ := redisClient.ZRevRangeWithScores(ctx, userKey, 0, 9).Result() // Decay by score

            allCats := unionAndDedup(currentCat, coViews, userCats) // Map for dedup

            // Multi-get from Redis/Druid
            var union []ProductScore
            for _, cat := range allCats {
                bestKey := "cat:bestsellers:" + cat.id
                if exists := redisClient.Exists(ctx, bestKey).Val(); exists == 1 {
                    union = append(union, redisClient.ZRevRangeWithScores(ctx, bestKey, 0, 49).Val()...)
                } else {
                    // Fallback Druid: Query top-50
                    druidRes := queryDruid("SELECT pid, sales FROM ... WHERE category='" + cat.id + "' ...")
                    union = append(union, druidRes...)
                }
            }
            // Dedup masters, limit
            return dedupAndLimitByMaster(union, 100), nil
        }
        ```
    - **Trade-offs**: Weekly co-view updates may lag; Druid adds dependency but excels at rollups.

#### 3. **Optimizing Related Categories Expansion (Co-Viewed Categories)**
- **Rationale**: Co-viewed categories (e.g., "electronics" with "accessories") expand recs without product-level volume. Updated weekly from session logs for stability.
- **Technical Implementation**:
    - **Co-Occurrence Computation**: Weekly batch (Airflow/Spark) analyzes logs, counts pairs with decay.
    - **Storage**: Redis hashes/sorted sets `ZADD cat:rels:cat1 score cat2`; Scylla map table.
    - **Expansion in Serving**: For current/user cats, fetch related (top-5–10), pull best-sellers/semantics.
    - **Latency Impact**: <1ms extra Redis lookups.
    - **Trade-offs**: Weekly cadence vs. real-time (use Flink for intra-week trends if needed).

#### 4. **Integrating Rerank/Filter Layer (Boosted Products, Flash Sales, Ads)**
- **Rationale**: Post-generation rerank/filter adapts the long candidate list (from sections 1–3) for business rules, ensuring relevance and monetization.
- **Technical Implementation**:
    - **Boost Data**: Redis hashes `HSET promo:pid123 boost 2.0 expiry 3600 out_of_stock false ad_bid 1.5 flash_sale true`.
        - Ingest via Flink/ETL from promo events.
    - **Reranking Algorithm**: In Go, after candidate fetch (e.g., 1000+ from semantic/best-sellers/co-view expansions):
        - Filter: Remove out-of-stock (`if HGET promo:pid out_of_stock == "true"`).
        - Boost: NewScore = BaseScore * (ad_bid + flash_sale_bonus) (e.g., +2.0 for flash, multiplicative for bids).
        - Heap/sort for top-K (e.g., 50 per widget).
        - Widget-specific: Sponsored slots prioritize ads.
    - **Online Updates**: Promo changes pub/sub to Redis, invalidate caches.
    - **Code Snippet (Go Rerank)**:
        ```go
        func RerankCandidates(candidates []ProductScore, widget string) []ProductScore {
            pipe := redisClient.Pipeline()
            for _, c := range candidates {
                pipe.HMGet("promo:" + c.ProductID, "boost", "out_of_stock", "ad_bid", "flash_sale")
            }
            res, _ := pipe.Exec(ctx)

            filtered := []ProductScore{}
            for i, r := range res {
                hm := r.(*redis.StringSliceCmd).Val()
                if hm[1] == "true" { continue } // Out of stock
                boost, _ := strconv.ParseFloat(hm[0], 64)
                bid, _ := strconv.ParseFloat(hm[2], 64)
                flash := hm[3] == "true"
                candidates[i].Score *= boost
                if flash { candidates[i].Score += 2.0 }
                if widget == "sponsored" { candidates[i].Score *= bid }
                filtered = append(filtered, candidates[i])
            }
            // Sort, apply filters
            sort.SliceStable(filtered, func(i, j int) bool { return filtered[i].Score > filtered[j].Score })
            return filtered[:50]
        }
        ```
    - **Latency Impact**: <5ms for 1000 candidates.
    - **Trade-offs**: Over-boosting degrades UX; A/B test (see section 6).

#### 5. **General Online Feature Updates and System-Wide Optimizations**
- **Rationale**: Streaming keeps features fresh; unify for consistency.
- **Implementation**:
    - **Feature Store**: Redis (volatile: sessions, recent cats) + Scylla/Druid (persistent: relations, rollups).
    - **Monitoring/Fallbacks**: Prometheus (Flink backpressure, Redis hits >95%, Druid query time). Go circuit breakers.
    - **Scaling**: Redis Cluster sharding; Flink/Kubernetes autoscaling on events; Druid multi-node for rollups.
    - **End-to-End**: Event → Kafka/Flink (1-5s) → Redis/Scylla/Druid (<1ms) → Go (<5ms total).
    - **Micro-Batch Training Experiments**: Using Airflow to orchestrate micro-batch jobs (e.g., every 1-4 hours) for semantic relations/co-view updates. Experimenting with incremental model retraining (e.g., update embeddings on new products only) to reduce full daily compute, integrated with Flink for hybrid batch-stream.

#### 6. **A/B Testing with Amplitude**
- **Rationale**: Validate optimizations (e.g., co-view expansion impact, rerank boosts) on metrics like click-through rate (CTR), conversion, session time.
- **Implementation**:
    - **Setup**: Amplitude for event tracking (e.g., "rec_viewed", "rec_clicked", "purchase"). Define variants: A (baseline batch), B (streaming with category decay).
    - **Targeting**: Random user bucketing (e.g., 50/50 split via Go hash(userID)), or segmented (new vs. returning users).
    - **Metrics**: Primary: CTR on rec widgets; Secondary: Revenue per session, bounce rate.
    - **Execution**: Run tests 1-2 weeks, use Amplitude dashboards for significance (t-tests). Integrate with serving: If variant=B, apply new logic.
    - **Examples**: Test decay rates for user recent cats (e.g., 1-day vs. 7-day TTL); rerank flash sale boosts (1.5x vs. 2.5x).
    - **Trade-offs**: Overhead in code (variant flags); ensure statistical power with large samples.




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

