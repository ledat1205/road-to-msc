## RECOMMENDATION

#### 1. **Optimizing Related Products Recommendations (Model-Inferred, Up to 1000 per Product)**

- **Rationale**: Daily training outputs semantic similarities (from titles) and co-view lists. To serve fast, precompute these as graphs or lists, but update incrementally for real-time co-views (e.g., from user sessions). This avoids full DB scans per request.
- **Technical Implementation**:
    - **Data Modeling**: In ScyllaDB, store as a graph-like structure using wide rows. E.g., table product_relations with product_id as partition key, and clustering columns for related products sorted by relevance score (from model or co-view count).
        - Schema: CREATE TABLE product_relations (product_id uuid PRIMARY KEY, related_products frozen<map<uuid, float>>); (map limits to 1000 entries, float for similarity scores).
    - **Batch Loading**: Daily training job (e.g., Spark or Airflow) writes semantic lists to Scylla. For co-views, use Flink to aggregate real-time views from event streams (Kafka), updating scores with exponential decay (e.g., recent views weigh more).
        - Flink Operator: Use keyed state (ProcessFunction) to maintain a priority queue (top-100) per product, evicting low-score items.
    - **Online Updates**: Flink consumes view events, increments co-view counts, and writes deltas to Redis (as sorted sets: ZADD product:rel:123 score1 related1 score2 related2) with TTL=1 day, and async to Scylla for persistence.
        - Cache Warmup: Preload top products' relations into Redis via a Flink bootstrap job.
    - **Serving in Go**: Query Redis first for ZREVRANGE product:rel:pid 0 99 WITHSCORES; fallback to Scylla. Intersect with user's recent views (stored in Redis as a list: LPUSH user:views:uid pid1 pid2, limit to last 10-20).
    - **Latency Impact**: Reduces computation from O(n) model inference to O(1) cache lookup. Update latency: <5s via Flink.
    - **Code Snippet (Flink for Co-View Updates)**:
        
        ```
        // Flink job for real-time co-view aggregation
        DataStream<ViewEvent> views = env.fromSource(kafkaSource, ...);
        views.keyBy(ViewEvent::getProductId) // Key by anchor product
            .window(TumblingProcessingTimeWindows.of(Time.minutes(5))) // Micro-batch every 5 min
            .reduce((acc, event) -> {
                // Increment co-view scores with decay (e.g., alpha=0.9 for recency)
                acc.relatedProducts.merge(event.coViewedProduct, event.score * 0.9 + acc.getOrDefault(event.coViewedProduct, 0f));
                // Prune to top-100 using a min-heap or sorted map
                acc.pruneToTopK(100);
                return acc;
            })
            .addSink(new RedisSink() { // ZADD to Redis, then async Scylla upsert
                @Override
                public void invoke(RelatedProducts value) {
                    jedis.zadd("product:rel:" + value.productId, value.relatedProducts);
                    // Async Scylla: Batch insert with TTL
                }
            });
        ```
        
    - **Trade-offs**: Memory in Flink state (use RocksDB with compression). Handle skew (popular products) by over-provisioning task slots.

#### 2. **Optimizing Best-Seller Recommendations in Current User Categories (1-10 Categories, Dynamic)**

- **Rationale**: Categories change per session; best-sellers need real-time ranking based on sales/views. Daily batches miss intra-day trends (e.g., flash sales).
- **Technical Implementation**:
    - **Real-Time Aggregation**: Flink streams sales/view events, maintaining top-K (e.g., top-50) per category using sliding windows (e.g., 1-hour window, advance every 5 min).
        - Use Flink's KeyedProcessFunction with timers for efficient top-K via priority queues (heap) in state.
        - Categories per user: Track in Redis as a set: SADD user:cats:uid cat1 cat2 (TTL=session duration, e.g., 30 min), updated on view events via Flink or directly in Go.
    - **Storage**: Store category best-sellers in Redis as sorted sets: ZADD cat:bestsellers:cat1 salesScore1 pid1 salesScore2 pid2. Flink updates scores (e.g., score = views * 0.5 + sales * 1.0).
        - Scylla Backup: Periodic snapshots (every hour) for recovery.
    - **Serving**: In Go, fetch user's categories from Redis, then multi-get best-sellers (MULTI pipeline: ZREVRANGE for each cat), union and dedup.
    - **Dynamic Updates**: On category change (detected in Flink from view events), expire old user sets and recompute.
    - **Latency Impact**: Aggregations in <10s; serving <2ms with Redis multi-get.
    - **Code Snippet (Go Serving Example)**:
        
        ```
        func GetBestSellers(ctx context.Context, userID string) ([]ProductID, error) {
            catsKey := "user:cats:" + userID
            cats, err := redisClient.SMembers(ctx, catsKey).Result() // Get 1-10 cats
            if err != nil { return nil, err }
        
            pipe := redisClient.Pipeline()
            var cmds []*redis.SliceCmd
            for _, cat := range cats {
                key := "cat:bestsellers:" + cat
                cmds = append(cmds, pipe.ZRevRange(ctx, key, 0, 49)) // Top-50
            }
            _, err = pipe.Exec(ctx)
            if err != nil { return nil, err }
        
            var union []ProductID
            for _, cmd := range cmds {
                union = append(union, cmd.Val()...) // Merge, dedup with a map if needed
            }
            // Dedup and limit
            return dedupAndLimit(union, 100), nil
        }
        ```
        
    - **Trade-offs**: Window size affects freshness vs. compute (smaller windows = more CPU in Flink).

#### 3. **Optimizing Related Categories Expansion (Co-Viewed Categories)**

- **Rationale**: Categories viewed together (e.g., "electronics" with "accessories") allow expanding recs. Build a co-occurrence graph updated online.
- **Technical Implementation**:
    - **Co-Occurrence Computation**: Flink processes session views (windowed per user session), counting pairs (catA, catB). Use matrix or graph: Aggregate globally with keyed state on cat pairs.
        - Decay: Use time-based decay (e.g., count *= 0.95 per window) for trending relations.
    - **Storage**: Redis graphs (via hashes: HINCRBY cat:rels:cat1 cat2 1) or dedicated graph module if using RedisGraph. Threshold to top-5 related per cat.
        - Scylla: CREATE TABLE cat_relations (cat_id text PRIMARY KEY, related_cats map<text, int>);
    - **Expansion in Serving**: For user's current cats, fetch related, then pull best-sellers/related products from those.
    - **Latency Impact**: Adds <1ms to serving (extra Redis lookups).
    - **Trade-offs**: Pair explosion (limit to frequent cats); use sampling in Flink for high volume.

#### 4. **Integrating Rerank/Filter Layer (Boosted Products, Flash Sales, Ads)**

- **Rationale**: Post-inference reranking ensures promoted items appear in widgets (e.g., top slots). Do this in-memory in Go for speed, sourcing boosts from Redis.
- **Technical Implementation**:
    - **Boost Data**: Store promotions in Redis hashes: HSET promo:pid123 boost 2.0 expiry 3600 (multiplier or additive score). Flink or a separate ETL ingests promo events.
    - **Reranking Algorithm**: In Go, after fetching candidate lists (from above), apply boosts: NewScore = BaseScore * Boost (if exists). Use a heap for top-K rerank.
        - Filters: Widget-specific (e.g., ads only in "sponsored" widget); use rules in config.
    - **Online Updates**: Real-time promo changes trigger Redis pubs to invalidate user caches if needed.
    - **Code Snippet (Go Rerank)**:
        
        Go
        
        ```
        func RerankCandidates(candidates []ProductScore, widget string) []ProductScore {
            pipe := redisClient.Pipeline()
            for _, c := range candidates {
                pipe.HGet("promo:" + c.ProductID, "boost")
            }
            res, _ := pipe.Exec(context.Background())
        
            for i, r := range res {
                if boost, _ := r.(*redis.StringCmd).Float64(); boost > 0 {
                    candidates[i].Score *= boost
                }
            }
            // Heap or sort for top-K, apply widget filters
            sort.SliceStable(candidates, func(i, j int) bool { return candidates[i].Score > candidates[j].Score })
            return candidates[:50] // Example limit
        }
        ```
        
    - **Latency Impact**: <5ms for 100-500 candidates (in-memory sort).
    - **Trade-offs**: Boost over-use can degrade relevance; A/B test.

#### 5. **General Online Feature Updates and System-Wide Optimizations**

- **Rationale**: Emphasize streaming to keep features fresh beyond daily training.
- **Implementation**:
    - **Feature Store**: Unified in Redis (volatile features like sessions) and Scylla (persistent like relations). Flink acts as the online feature pipeline.
    - **Monitoring/Fallbacks**: Use Prometheus for latency (e.g., Flink backpressure, Redis hit rate >95%). Circuit breakers in Go for Scylla fallbacks.
    - **Scaling**: Redis sharding for hot keys; Flink dynamic scaling on event rate.
    - **End-to-End**: Full pipeline: Event -> Flink (1-5s) -> Redis/Scylla (<1ms) -> Go serve (<5ms total).


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

