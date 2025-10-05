## ⚖️ Types of Scaling

The document defines two distinct ways to scale a workload (a set of replicas):

1. **Horizontal Scaling:** Adjusting the **number of replicas** (Pods).
    
    - _Manual method:_ Use `kubectl scale`.
        
2. **Vertical Scaling:** Adjusting the **CPU and memory resources** assigned to containers in the replicas.
    
    - _Manual method:_ Use `kubectl patch` to modify the resource definition.
        

---

## 🤖 Automatic Scaling Workloads

Automatic scaling allows a controller to periodically adjust the number of replicas or their resources.

### Horizontal Autoscalers

|Autoscaler|Core/External|What it Scales|Basis for Scaling|
|---|---|---|---|
|**HorizontalPodAutoscaler (HPA)**|**Core Kubernetes**|**Number of replicas**|Observed resource utilization (e.g., CPU, memory) or custom metrics.|
|**Cluster Proportional Autoscaler**|External (GitHub Project)|**Number of replicas**|Size of the cluster (number of schedulable nodes and cores).|
|**Event Driven Autoscaler (KEDA)**|External (CNCF Project)|**Number of replicas**|External event sources (e.g., messages in a queue).|
|**Scheduled Autoscaling**|External (Via KEDA Cron scaler)|**Number of replicas**|Defined time schedules (e.g., daily off-peak hours).|

Xuất sang Trang tính

### Vertical Autoscalers

|Autoscaler|Core/External|What it Scales|Key Modes/Features|
|---|---|---|---|
|**VerticalPodAutoscaler (VPA)**|External (GitHub Project)|**Resource requests** (CPU/Memory)|Automatically recommends or sets resource limits based on usage history.|
|**Cluster Proportional Vertical Autoscaler**|External (GitHub Project - Beta)|**Resource requests**|Size of the cluster (number of nodes/cores).|

Xuất sang Trang tính

**VPA Specifics:**

- Requires the **Metrics Server** to be installed.
    
- Currently operates in four modes: `Auto`, `Recreate`, `Initial`, and `Off`.
    
- Note: In the `Recreate` mode, the VPA **evicts existing Pods** to apply the new resource recommendations.
    

---

## 🏢 Scaling Cluster Infrastructure

If scaling the applications (workloads) isn't enough, the cluster itself can be scaled. This involves the separate concept of **Node autoscaling**—the process of automatically adding or removing worker Nodes from the cluster.