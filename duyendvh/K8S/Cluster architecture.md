The architecture is split into two main roles:
![[Pasted image 20251004210006.png]]
### 1. The Control Plane (The Brain)

These components make global decisions about the cluster (like scheduling) and maintain the desired state of the cluster.

- **kube-apiserver:** The front end of the Control Plane. All communication (internal and external) goes through this component.
    
- **etcd:** A consistent and highly-available key-value store that holds all the cluster state data (configuration, current status, etc.).
    
- **kube-scheduler:** Watches for newly created Pods and assigns them to a healthy Node based on resource requirements and other policies.
    
- **kube-controller-manager:** Runs the various controller processes (like Replication, Namespace, and ServiceAccount controllers) which continuously loop to match the cluster’s current state to the desired state.
    
- **cloud-controller-manager:** Integrates the cluster with underlying cloud provider APIs for managing resources like load balancers and persistent storage volumes.
    

### 2. The Nodes (The Workers)

These are the machines (VMs or physical servers) that run your actual applications.

- **kubelet:** An agent on each Node that ensures containers are running in a Pod. It communicates with the Control Plane and executes instructions.
    
- **kube-proxy:** A network proxy that maintains network rules on Nodes. This allows for network communication to and from your Pods, both internal and external.
    
- **Container Runtime:** The software responsible for running containers (like Docker, containerd, or CRI-O).

### 3. Addons

Addons are cluster-level features that use standard Kubernetes objects (like Deployments and DaemonSets) and typically run within the **`kube-system`** namespace.

- **DNS (Crucial):** Every Kubernetes cluster should have **Cluster DNS**, which serves DNS records specifically for Kubernetes Services, enabling service discovery for containers.
    
- **Networking:** **Network Plugins** implement the Container Network Interface (CNI), responsible for allocating IP addresses to Pods and allowing them to communicate.
    
- **Management:** Includes the **Web UI (Dashboard)** for managing the cluster, as well as tools for **Container resource monitoring** and **Cluster-level logging**.
    

---

### 4. Architecture variations

While the core components remain the same, Kubernetes is highly flexible in how they are deployed:

- **Control Plane Deployment Options:**
    
    - **Traditional:** Components run as standard services (e.g., systemd) on dedicated machines.
        
    - **Static Pods:** Components are managed directly by the Kubelet on a Node (common setup for tools like Kubeadm).
        
    - **Self-hosted:** The control plane itself runs as regular Pods managed by Deployments within the cluster.
        
    - **Managed Services:** Cloud providers abstract and manage the control plane completely.
        
- **Workload Placement:** In production, Control Plane components are often placed on **dedicated Nodes**, separate from user workloads, for stability and security.
    
- **Customization:** Kubernetes allows for high extensibility, including deploying **Custom Schedulers** and extending the API using **CustomResourceDefinitions (CRDs)**.