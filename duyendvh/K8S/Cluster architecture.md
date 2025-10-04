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