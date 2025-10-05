## What is a Pod?

A Pod is the **smallest deployable unit** you can create and manage. It models an "application-specific logical host" by encapsulating a group of one or more containers that are designed to work together.

- **Shared Context:** All containers within a Pod are always **co-located and co-scheduled** onto the same Node. They run in a shared context, meaning they share the same **network namespace** (IP address, ports) and **storage volumes**.
    
- **Containers:** A Pod can contain:
    
    - **Application Containers:** The main processes of your application (the most common use is one per Pod).
        
    - **Init Containers:** Run to completion _before_ application containers start, often for setup tasks.
        
    - **Sidecar Containers (Advanced):** Tightly coupled helper containers that run alongside the main application for auxiliary services (e.g., logging or proxies).
        
    - **Ephemeral Containers:** Can be injected into a running Pod for **debugging**.
        

---

## ⚙️ Management and Lifecycle

Pods are not typically created individually. They are designed to be ephemeral and disposable, managed by higher-level workload resources:

- **Workload Resources:** Controllers (like **Deployments, StatefulSets,** or **Jobs**) manage Pods, handling replication and auto-healing (creating a new Pod if a Node fails).
    
- **Pod Templates:** Workload resources use a **PodTemplate** to define the specification for the Pods they create. Modifying a template forces the controller to create **replacement Pods**; existing Pods are **not updated in place**.
    
- **Security Context:** The **`securityContext`** field allows you to define granular security constraints for the Pod or individual containers (e.g., forcing them to run as a non-root user).
    
- **Probes:** The **Kubelet** periodically performs diagnostic checks (**ExecAction, TCPSocketAction, HTTPGetAction**) on containers to determine their health status.
    
- **Static Pods:** A unique type of Pod managed directly by the **Kubelet** on a Node, primarily used to run the Kubernetes Control Plane components themselves.