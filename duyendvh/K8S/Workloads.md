
A **Workload** is simply an application running on Kubernetes, and it is run within one or more **Pods**.

### 1. Pods (The Smallest Deployable Unit)

- A Pod represents a set of running **containers** on a Node.
    
- Pods have a defined **lifecycle**. If the Node running the Pod fails, that specific Pod fails _permanently_ and must be recreated.
    
- They are the fundamental unit for deployment.
    

---

### 2. Workload Resources (The Managers)

You typically manage Pods using a higher-level **workload resource**. These resources utilize **controllers** to ensure the cluster's actual state matches the desired state you specify:

| Workload Resource           | Primary Use Case                                                                  | Key Function                                                                                                         |
| --------------------------- | --------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **Deployment & ReplicaSet** | **Stateless applications** (e.g., a web frontend) where Pods are interchangeable. | Manages ReplicaSets to ensure a consistent, desired number of replica Pods are running.                              |
| **StatefulSet**             | **Stateful applications** (e.g., databases) where Pods need unique identities.    | Matches each Pod with dedicated resources (like a **PersistentVolume**) to track state and maintain stable identity. |
| **DaemonSet**               | **Node-local services** (e.g., networking, monitoring).                           | Runs exactly one Pod on every Node (or a selected subset of Nodes).                                                  |
| **Job**                     | **Tasks that run to completion** (once).                                          | Creates one or more Pods that execute a task and terminate successfully.                                             |
| **CronJob**                 | **Scheduled tasks** (recurring).                                                  | Manages Jobs according to a defined, recurring schedule.                                                             |

### 3. Extensibility and Management

- **Customization:** The architecture is flexible. You can use **Custom Resource Definitions (CRDs)** to create third-party workload resources with custom behaviors.
    
- **Garbage Collection:** This automatically removes dependent objects (like Pods) when their **owning resource** (like a Deployment) is removed.
    
- **Time-to-Live (TTL):** A controller automatically removes **Jobs** once a specified time has passed after they complete.