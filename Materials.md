### Faster-Whisper + CTranslate2 (Core for STT in C++)

CTranslate2 is the fast C++/Python inference engine behind Faster-Whisper. Start here for model conversion and C++ usage.

- **Official Faster-Whisper Repo** (SYSTRAN/faster-whisper): Primary source with benchmarks, quantization (int8/float16), and Python examples you can adapt to C++. Includes Whisper model specifics.
    - GitHub: [https://github.com/SYSTRAN/faster-whisper](https://github.com/SYSTRAN/faster-whisper)
- **CTranslate2 Official Repo & Docs**: Deep dive into C++ API for Whisper inference (models::WhisperModel, generate(), options like initial_prompt). Covers CUDA/GPU setup, quantization, and performance.
    - GitHub: [https://github.com/OpenNMT/CTranslate2](https://github.com/OpenNMT/CTranslate2)
    - Benchmarks & Whisper support: Excellent for your GPU edge server use.
- **Conversion Tutorial**: "Converting Your Fine-Tuned Whisper Model to Faster-Whisper Using CTranslate2" — step-by-step ct2-transformers-converter command + performance comparison.
    - Medium: [https://medium.com/@balaragavesh/converting-your-fine-tuned-whisper-model-to-faster-whisper-using-ctranslate2-b272063d3204](https://medium.com/@balaragavesh/converting-your-fine-tuned-whisper-model-to-faster-whisper-using-ctranslate2-b272063d3204)
- **Video**: "Faster Whisper CTranslate2 (Speech to Text)" — practical demo with turbo models and low-memory tips.
    - YouTube: [https://www.youtube.com/watch?v=eFmql0NqacU](https://www.youtube.com/watch?v=eFmql0NqacU)

### ONNX Runtime C++ for BERT-Tiny Inference

Focus on C++ API basics and BERT examples.

- **ONNX Runtime Official Tutorials – API Basics & C++ Examples**: Start with inferencing basics, then check C/C++ inference examples repo (includes BERT-like models).
    - Docs: [https://onnxruntime.ai/docs/tutorials/api-basics.html](https://onnxruntime.ai/docs/tutorials/api-basics.html)
    - GitHub examples: [https://github.com/microsoft/onnxruntime-inference-examples/tree/main/c_cxx](https://github.com/microsoft/onnxruntime-inference-examples/tree/main/c_cxx)
- **ONNX Runtime C++ Inference Tutorials**: Full guide for loading .onnx models, sessions, and running inference (perfect for your fine-tuned BERT-Tiny).
    - Main docs: [https://onnxruntime.ai/docs/tutorials](https://onnxruntime.ai/docs/tutorials)
- **Video**: "Inference ML with C++ and ONNXRuntime" — console app example (ResNet, but easily adaptable to BERT).
    - YouTube: [https://www.youtube.com/watch?v=imjqRdsm2Qw](https://www.youtube.com/watch?v=imjqRdsm2Qw)
- **Practical BERT Service Example**: "Practical Implementation of BERT Model Service Based on bRPC and ONNX Runtime" — C++ service setup.
    - Blog: [http://oreateai.com/blog/practical-implementation-of-bert-model-service-based-on-brpc-and-onnx-runtime/3ec54552fb449ad42fcdb83c53fa2c5c](http://oreateai.com/blog/practical-implementation-of-bert-model-service-based-on-brpc-and-onnx-runtime/3ec54552fb449ad42fcdb83c53fa2c5c)

### Paho MQTT C++ Client

Essential for your audio ingress from wearables.

- **Eclipse Paho MQTT C++ Official Page & Examples**: Synchronous/async publish/subscribe basics.
    - [https://eclipse.dev/paho/clients/cpp](https://eclipse.dev/paho/clients/cpp)
- **Comprehensive Guide**: "Using MQTT with C++: A Comprehensive Guide for IoT Developers" — publisher/subscriber examples with EMQX broker.
    - EMQX Blog: [https://www.emqx.com/en/blog/using-mqtt-with-cpp](https://www.emqx.com/en/blog/using-mqtt-with-cpp)
- **CMake Project Tutorial**: "How to Use Paho MQTT Client Library in C++ CMake Project" — full sample with all MQTT features in one file.
    - Cedalo Blog: [https://cedalo.com/blog/implement-paho-mqtt-c-cmake-project](https://cedalo.com/blog/implement-paho-mqtt-c-cmake-project)
- **Video**: "MQTT - Subscribe and publish programmatically with paho and c/++" — practical coding walkthrough.
    - YouTube: [https://www.youtube.com/watch?v=7aS44IRmi5w](https://www.youtube.com/watch?v=7aS44IRmi5w)

### gRPC Bidirectional Streaming in C++

Critical for your audio chunk piping from bridge to Whisper service.

- **Official gRPC C++ Basics Tutorial**: Covers bidirectional streaming RPCs (stream keyword in proto), client/server setup.
    - [https://grpc.io/docs/languages/cpp/basics](https://grpc.io/docs/languages/cpp/basics)
- **Async Bidirectional Client Guide**: "How to create a bi-directional gRPC client in C++" — full code for streaming events/commands.
    - The Social Robot Blog: [https://www.thesocialrobot.org/posts/grpc-brain-2](https://www.thesocialrobot.org/posts/grpc-brain-2)
- **Best Practices & Async Examples**: Official docs on streaming lifecycle, holds, and bi-di usage.
    - [https://grpc.io/docs/languages/cpp/best_practices](https://grpc.io/docs/languages/cpp/best_practices)
- **Video**: "streaming in gRPC from server perspective" — covers bi-directional with client/server code.
    - YouTube: [https://www.youtube.com/watch?v=3mlk3bPyKxY](https://www.youtube.com/watch?v=3mlk3bPyKxY)

### Piper TTS (ONNX-based, C++ Engine)

For fast local voice feedback.

- **Official Piper Repo**: Core C++ source, ONNX runtime integration, voice models on Hugging Face.
    - GitHub: [https://github.com/rhasspy/piper](https://github.com/rhasspy/piper) (check archived status, but still widely used; forks active)
- **Training & Export Guide**: "Training a Tiny Piper TTS Model for Any Language" — includes ONNX export for C++ usage.
    - Neurlang Blog: [https://blog.hashtron.cloud/post/2025-09-28-training-a-a-tiny-piper-tts-model-for-any-language](https://blog.hashtron.cloud/post/2025-09-28-training-a-a-tiny-piper-tts-model-for-any-language)
- **Minimal Training Example**: "Training a new AI voice for Piper TTS with only 4 words" — export to ONNX + JSON.
    - Cal Bryant Blog: [https://calbryant.uk/blog/training-a-new-ai-voice-for-piper-tts-with-only-4-words](https://calbryant.uk/blog/training-a-new-ai-voice-for-piper-tts-with-only-4-words)
- **Video Tutorial**: "TEXT TO SPEECH | Piper TTS on Windows AI voice 10x faster Realtime!" — command-line usage, adaptable to C++ integration.
    - YouTube: [https://www.youtube.com/watch?v=GGvdq3giiTQ](https://www.youtube.com/watch?v=GGvdq3giiTQ)

### Multi-Tenant Edge Deployment with Kubernetes/Helm + NVIDIA MIG

For your client edge kits and cloud backend.

- **NVIDIA MIG in Kubernetes Official Docs**: Enable MIG, partition GPUs (A100/H100), integrate with GPU Operator.
    - [https://docs.nvidia.com/datacenter/cloud-native/kubernetes/latest/index.html](https://docs.nvidia.com/datacenter/cloud-native/kubernetes/latest/index.html)
    - Full MIG User Guide: [https://docs.nvidia.com/datacenter/tesla/mig-user-guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide)
- **Helm for Multi-Tenancy Patterns**: "Multi-Tenancy Patterns with Helm" — namespace isolation, quotas, RBAC via charts.
    - OneUptime Blog: [https://oneuptime.com/blog/post/2026-01-17-helm-kubernetes-multi-tenancy-patterns/view](https://oneuptime.com/blog/post/2026-01-17-helm-kubernetes-multi-tenancy-patterns/view)
- **Edge AI Kubernetes Blueprint**: Best practices for immutable OS, zero-touch, fleet management at edge.
    - WWT Research: [https://www.wwt.com/wwt-research/edge-ai-kubernetes-enterprise-blueprint](https://www.wwt.com/wwt-research/edge-ai-kubernetes-enterprise-blueprint)
- **Video**: "Tutorial: Run multiple workloads using a single GPU" — MIG partitioning demo (relevant for your cloud GPU sharing).
    - YouTube: [https://www.youtube.com/watch?v=KbB5e_V6THw](https://www.youtube.com/watch?v=KbB5e_V6THw)

### Recommendation Industry Case Studies & Real-World Architectures

- **Alibaba/Taobao Real-Time Personalization with Flink** Alibaba heavily uses Flink for real-time recommendations in retail/e-commerce, including during massive events like Double 11. They evolved to Flink + Paimon (unified stream-batch) with Kafka ingestion, Redis caching, and low-latency serving.
    - Blog: [Apache Flink: Powering Real-Time Personalization in Retail and E-Commerce](https://www.alibabacloud.com/blog/apache-flink-powering-real-time-personalization-in-retail-and-e-commerce_602072)
    - Taobao's real-time data warehouse using Flink + message queues + OLAP (similar to your streaming layer).
    - Blog: [How Taobao uses Apache Fluss (Incubating) for Real-Time Processing in Search and RecSys](https://fluss.apache.org/blog/taobao-practice)
- **Netflix Real-Time Recommendation & Streaming Architecture** Netflix's system handles real-time updates via event streams (Kafka-like), distributed processing, and hybrid offline/online computation. Great for understanding candidate generation, reranking, and session context (analogous to your dynamic categories/views).
    - Netflix Tech Blog series: [How and Why Netflix Built a Real-Time Distributed Graph](https://netflixtechblog.com/how-and-why-netflix-built-a-real-time-distributed-graph-part-1-ingesting-and-processing-data-streams-at-internet-scale) (focus on ingesting/processing streams at scale).
    - Older foundational: [Netflix System Architectures for Personalization and Recommendation](https://netflixtechblog.com/system-architectures-for-personalization-and-recommendation-e081aa94b5d8) (Cassandra storage + Spark/Flink-like processing).
    - Modern insights often reference Kafka for events and low-latency serving.
- **Tripadvisor Real-Time Personalization with ScyllaDB** Directly relevant to ScyllaDB usage for real-time personalization at massive scale (similar to your storage layer).
    - Resource: [Inside Tripadvisor’s Real-Time Personalization with ScyllaDB + AWS](https://resources.scylladb.com/scylladb-user-stories/inside-tripadvisor-s-real-time-personalization-with-scylla-aws) (video + details on low-latency reads/writes).
- **Grab (Ride-Hailing/E-commerce) with ScyllaDB** Uses Scylla for real-time counters and high-throughput event processing (billions/day), applicable to your online feature updates.
    - Blog: [Grab App at Scale with ScyllaDB](https://www.scylladb.com/2021/06/23/grab-app-at-scale-with-scylla/)
- **DoorDash Gigascale ML Feature Store with Redis** Excellent for Redis as online feature store for real-time ML inference (tens of millions reads/sec), mirroring your caching for user sessions/categories/boosts.
    - Blog: [Building a Gigascale ML Feature Store with Redis](https://careersatdoordash.com/blog/building-a-gigascale-ml-feature-store-with-redis/)

### Practical Blogs & Tutorials on Similar Architectures

- **Tinybird: What it takes to build a real-time recommendation system** Covers end-to-end: streaming ingestion (Kafka), processing (Flink or SQL-based), caching (Redis), storage alternatives (ClickHouse/Cassandra-like), and serving <100ms. Discusses hybrid batch + real-time, very close to your Flink + Redis + Scylla setup.
    - Blog: [What it takes to build a real-time recommendation system](https://www.tinybird.co/blog/real-time-recommendation-system)
- **Redis + Vector Search for Real-Time Product Recs** Tutorial on content-based filtering with Redis vectors (good for your product-related lists from semantics/co-views).
    - Blog: [How To Build a Real-Time Product Recommendation System Using Redis and DocArray](https://redis.io/blog/real-time-product-recommendation-docarray)
- **General Scalable RecSys Design** Step-by-step with streaming (Flink/Kafka), feature stores, Redis caching, Cassandra/DynamoDB, and reranking.
    - Guide: [Recommendation System Design: (Step-by-Step Guide)](https://www.systemdesignhandbook.com/guides/recommendation-system-design)
- **Instacart's Journey to Real-Time ML** Transition from batch to real-time features/inference (e-commerce/grocery context).
    - Blog: [Lessons Learned: The Journey to Real-Time Machine Learning at Instacart](https://www.instacart.com/company/how-its-made/lessons-learned-the-journey-to-real-time-machine-learning-at-instacart/)
- **Shaped.ai Interview on Building Real-Time RecSys at Scale** Insights from building production recsys (candidate gen + ranking).
    - Blog: [Building Real-Time Recommendation Systems at Scale with Jason Liu](https://www.shaped.ai/blog/building-real-time-recommendation-systems-at-scale)

### Additional High-Quality Overviews

- **Databricks: A Practical Guide to Building an Online Recommendation System** Focus on online serving, candidate gen, feature retrieval, filtering/reranking—great for your rerank/boost layer.
    - Blog: [A Practical Guide to Building an Online Recommendation System](https://www.databricks.com/blog/guide-to-building-online-recommendation-system)
- **AWS/Rockset: Real-Time Rec Engine with MSK (Kafka) + Rockset** Streaming architecture pattern (replace Rockset with your Scylla/Redis if needed).
    - Blog: [Building a real-time recommendation engine with Amazon MSK and Rockset](https://aws.amazon.com/blogs/awsmarketplace/building-real-time-recommendation-engine-amazon-msk-rockset)

### ScyllaDB Resources

ScyllaDB is Cassandra-compatible but optimized in C++ for better performance and predictability—ideal for your persistent recommendation storage (e.g., product relations, user data).

- **Official Documentation** (Start here for everything): [https://docs.scylladb.com/](https://docs.scylladb.com/) — Comprehensive user guide covering architecture, installation, data modeling, drivers, operations, monitoring, and best practices. Includes sections on schema design for high-write/high-read workloads like yours.
- **Getting Started / First Steps**: [https://docs.scylladb.com/stable/get-started](https://docs.scylladb.com/stable/get-started) — Step-by-step for installation, basic queries, and connecting from languages like Go (gocqlx driver).
- **Tutorials and Example Projects**: [https://docs.scylladb.com/stable/get-started/develop-with-scylladb/tutorials-example-projects.html](https://docs.scylladb.com/stable/get-started/develop-with-scylladb/tutorials-example-projects.html) — Hands-on examples (e.g., gaming leaderboards, which share patterns with recommendation lists).
- **ScyllaDB University** (Free courses): [https://university.scylladb.com/](https://university.scylladb.com/) — Structured learning paths with videos, labs, and certifications on data modeling, performance tuning, and migration.
- **ScyllaDB vs. Apache Cassandra Comparison & Migration**: [https://www.scylladb.com/compare/scylladb-vs-apache-cassandra/](https://www.scylladb.com/compare/scylladb-vs-apache-cassandra/) — Detailed benchmarks, why Scylla often outperforms Cassandra (e.g., shard-per-core, lower latency). Migration guides: Look for "Cassandra to ScyllaDB" paths in docs or [https://www.scylladb.com/tech-talk/cassandra-to-scylladb-technical-comparison-and-the-path-to-success](https://www.scylladb.com/tech-talk/cassandra-to-scylladb-technical-comparison-and-the-path-to-success).
- **Blog & Use Cases**: ScyllaDB Blog[](https://www.scylladb.com/blog/) — Search for "recommendation", "personalization", or "real-time" posts (e.g., Tripadvisor's real-time personalization case).

### Redis Resources

Redis shines as your L1 cache, online feature store (user sessions, categories, boosts, top-K lists), and for sub-ms serving in recsys.

- **Official Documentation** (Core reference): [https://redis.io/docs/latest](https://redis.io/docs/latest) — Covers all commands, data structures (hashes, sorted sets, lists—perfect for your related products, best-sellers), clients, best practices, and modules (e.g., Redis Query Engine for vectors if you expand to embeddings).
- **Best Practices & Patterns**: [https://redis.io/docs/latest/develop/best-practices/](https://redis.io/docs/latest/develop/best-practices/) — Sections on memory management, high availability (Sentinel/Cluster), security, and performance tuning. Specific to feature stores/recsys: [https://redis.io/feature-store](https://redis.io/feature-store) — Explains using Redis as an online feature store for real-time inference (fraud, recommendations, dynamic pricing).
- **Redis as Feature Store / Recommendation Use Cases**: [https://redis.io/blog/building-feature-stores-with-redis-introduction-to-feast-with-redis](https://redis.io/blog/building-feature-stores-with-redis-introduction-to-feast-with-redis) — Integrating with Feast for online serving. DoorDash case: Search for "Building a Gigascale ML Feature Store with Redis" (detailed engineering post on scaling to millions of reads/sec).
- **Redis University** (Free courses): [https://university.redis.com/](https://university.redis.com/) — Courses on Redis for caching, real-time apps, security, and AI/ML use cases.

### Apache Flink Resources

Flink handles your streaming layer for incremental updates (co-views, category best-sellers, real-time aggregations) from events to Redis/Scylla sinks.

- **Official Documentation** (Primary source): [https://nightlies.apache.org/flink/flink-docs-stable](https://nightlies.apache.org/flink/flink-docs-stable) — Full reference: Concepts, APIs (DataStream, Table/SQL, PyFlink), state management, checkpoints, connectors (Kafka, Cassandra/Scylla-compatible, Redis), and operations.
- **Hands-On Tutorials / Learn Flink**: [https://nightlies.apache.org/flink/flink-docs-stable/docs/learn-flink/overview/](https://nightlies.apache.org/flink/flink-docs-stable/docs/learn-flink/overview/) — "Learn Flink" training series with exercises on streaming ETL, state, time, and event-driven apps. Quick starts: Fraud detection (DataStream API), real-time reporting (Table API), PyFlink intro.
- **Connectors & Sinks** (Relevant to your setup): Search docs for "Cassandra Connector" (works with Scylla) and custom Redis sinks. Blog example: "Flink Integration To ScyllaDB - Supercharge Your RealTime Analytics" — PyFlink pipelines to Scylla.
- **Use Cases & Case Studies** (Recsys focus): [https://flink.apache.org/what-is-flink/use-cases/](https://flink.apache.org/what-is-flink/use-cases/) — Event-driven apps, data pipelines (e.g., real-time search index building in e-commerce). Alibaba/Taobao: Real-time personalization with Flink (search their tech blog or Flink site for "Blink" evolution). Zalando, UberEats, Netflix examples in Flink community posts for recommendation streaming.