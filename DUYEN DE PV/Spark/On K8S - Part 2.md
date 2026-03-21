### 1. Why Spark Doesn't Run "Like a Normal K8s App" (Deployments/StatefulSets/Jobs)

Spark applications have a **unique, dynamic lifecycle** unlike long-running microservices:

- One **Driver** pod spawns and manages multiple **Executor** pods dynamically.
- Executors are **not identical replicas** — they register with the Driver, handle tasks/shuffles, and can scale up/down.
- The application is **transient** (batch) or **long-running but stateful** (streaming) — not fixed replicas.
- Standard K8s objects (Deployment/StatefulSet) can't handle a parent pod creating/managing children via API calls.

**Result**: Plain YAMLs for Deployments fail — you lose dynamic scaling, Driver-Executor RPC, shuffle safety, and automatic cleanup.

**Solution paths**:

- **Native spark-submit** (imperative, built-in since Spark 2.3).
- **Spark Operator** (declarative CRD, production standard).

### 2. Native spark-submit on Kubernetes (Built-in, No Extra Components)

**Official status (Spark 4.1.1, 2026)**: Fully supported, mature, uses native K8s scheduler. Only **cluster mode** is production-recommended.

**Technical flow when you run spark-submit**:

1. spark-submit client (local/CI) parses flags → builds SparkConf.
2. Authenticates to K8s API (in-cluster token or ~/.kube/config).
3. Builds Driver Pod spec in memory (image, command = KubernetesDriverMain, env vars from conf, ports, volumes for jars/files).
4. Uploads app JARs/py-files to scratch space (emptyDir/PVC or init container).
5. Creates Driver pod via K8s API (POST /pods).
6. Driver pod starts → runs KubernetesDriverMain → re-builds Spark context.
7. Driver creates Executor pods dynamically via K8s API (same image usually).
8. Executors connect to Driver (pod IP + RPC port via K8s network).
9. Driver launches your code (main class/script).
10. Job runs → dynamic allocation scales pods.
11. Finish → Driver deletes Executors → Driver pod completes (logs persist).
12. spark-submit client exits when Driver completes.

**Key configs (via --conf flags)**:

- spark.kubernetes.container.image=yourcompany/spark:4.1.0
- spark.kubernetes.namespace=spark-jobs
- spark.kubernetes.authenticate.driver.serviceAccountName=spark-sa (RBAC)
- spark.dynamicAllocation.enabled=true
- spark.dynamicAllocation.shuffleTracking.enabled=true ← **mandatory** on K8s for safe down-scaling
- spark.dynamicAllocation.minExecutors=2, maxExecutors=50

**Pros**:

- Zero dependencies — just Spark binary + K8s cluster.
- Native scheduler integration.
- Simple for batch/CI.

**Cons**:

- Imperative (scripts needed).
- No single K8s object for job status.
- Manual retry/cleanup/monitoring.

### 3. Dynamic Allocation on Kubernetes (Cost Savings + Elasticity)

**Core idea**: Driver requests/removes Executor pods automatically based on workload.

**How it works**:

- Pending tasks/backlog → Driver creates more Executor pods (up to maxExecutors).
- Idle executors (after executorIdleTimeout) → Driver removes them (scales down).
- **On K8s**: Driver calls K8s API to create/delete pods.

**Critical requirement: Shuffle safety** Shuffle files (map outputs) are written to local disk. If an Executor dies while holding needed shuffle data → FetchFailedException → stage retry (slow, wasteful).

**Old solution (YARN)**: External Shuffle Service (ESS) — separate daemon serves files after Executor death.

**Kubernetes reality**: No native ESS support (no per-node auxiliary services like YARN).

**Modern solution (Spark 3.0+)**: **Shuffle Tracking**

- Driver tracks shuffle file locations (via MapOutputTracker).
- Before killing idle Executor: check if it holds active shuffle data.
- If yes → delay removal until shuffle phase ends or timeout.
- Key configs:
    - spark.dynamicAllocation.shuffleTracking.enabled=true (must-have on K8s)
    - spark.dynamicAllocation.shuffleTracking.timeout=300s (or infinity; default relies on GC)

**Benefits on K8s**:

- No extra infrastructure.
- True cost savings (pay only active pods; great with spot/preemptible nodes).
- Safe down-scaling → no recomputes.

**Tradeoff**: Executors with shuffle data stay alive longer → slightly less aggressive scaling than pure ESS.

### 4. Spark Operator (Declarative, Production-Grade)

**Purpose**: Makes Spark first-class K8s citizen via CRD SparkApplication.

**Two main operators in 2026**:

- **Kubeflow Spark Operator** (GoogleCloudPlatform/kubeflow/spark-operator) — veteran, battle-tested, Helm chart, supports Spark 2.3–4.x, webhook, Prometheus.
- **Official Apache Spark Kubernetes Operator** (apache/spark-kubernetes-operator) — launched May 2025, Apache governance, modern K8s features, faster releases, recommended for new setups.

**How Operator works**:

- Install via Helm (one command).
- Apply SparkApplication YAML → Operator watches → internally runs spark-submit-like logic.
- Manages Driver + Executors, status in CRD, restart, cleanup.

**Real production SparkApplication YAML example** (streaming job, 2026 style):

YAML

```
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: fraud-streaming-v2
  namespace: spark-prod
spec:
  type: Python
  mode: cluster
  image: "yourcompany/spark:4.1.0-python3.12"
  mainApplicationFile: "s3a://apps/fraud_detector.py"
  sparkVersion: "4.1.0"
  restartPolicy:
    type: OnFailure
    onFailureRetries: 3
  driver:
    cores: 2
    memory: "4Gi"
    serviceAccount: spark-prod-sa
  executor:
    cores: 3
    instances: 5  # initial
    memory: "6Gi"
  dynamicAllocation:
    enabled: true
    minExecutors: 2
    maxExecutors: 60
    shuffleTrackingTimeout: 300s
  sparkConf:
    "spark.sql.streaming.checkpointLocation": "s3a://checkpoints/fraud/"
    "spark.dynamicAllocation.shuffleTracking.enabled": "true"
    "spark.sql.shuffle.partitions": "800"
    "spark.kubernetes.driver.pod.name": "fraud-driver"  # optional
  volumes:
    - name: secrets
      secret:
        name: kafka-aws-creds
  monitoring:
    prometheus:
      jmxExporterJar: "/prometheus/jmx_prometheus_javaagent.jar"
      port: 8090
```

**Helm role**:

- Installs Operator + CRDs/RBAC.
- Your team can create custom Helm charts templating SparkApplication (values.yaml for envs: dev/prod executors, buckets).
- GitOps (Argo CD/Flux) syncs → zero manual kubectl.

### 5. Production Patterns & Best Practices (2026)

- **Use Operator** for >10 jobs, streaming, GitOps.
- **Native submit** for simple batch/CI.
- **Always enable shuffle tracking** on K8s.
- **Dynamic allocation + spot nodes** → huge cost savings (nodeSelector/tolerations in YAML).
- **Secrets**: Mount via volumes (never in sparkConf).
- **Monitoring**: JMX Prometheus exporter sidecar → Grafana (stages, shuffle, GC).
- **Debugging**: kubectl describe pod, driver logs, pending → quotas/taints.
- **Scheduling**: Airflow SparkKubernetesOperator, Argo Workflows, or ScheduledSparkApplication.

**Comparison Summary (2026)**

|Feature|Native spark-submit|Spark Operator (Apache/Kubeflow)|
|---|---|---|
|Style|Imperative CLI|Declarative CRD/YAML|
|Job as K8s object|No|Yes|
|GitOps|Hard|Excellent|
|Restart/Cron|Manual|Built-in|
|Dynamic alloc + shuffle|Yes (tracking)|Yes (tracking)|
|Production choice|Simple/legacy|Most teams (scalability + observability)|