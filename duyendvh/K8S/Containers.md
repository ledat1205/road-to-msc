
The document covers the fundamental concepts of what a container is and how it functions within the Kubernetes ecosystem:

- **Definition:** A container image is a **ready-to-run software package** that includes everything needed to execute an application: the code, runtime, system tools, and libraries.
    
- **Design Philosophy:** Containers are designed to be **stateless and immutable**. This means you should treat a running container as fixed. If you need to make changes, the correct procedure is to build a _new image_ with the updates and replace the old container with the new one.
    

### Key Components

|Component|Summary|
|---|---|
|**Container Runtimes**|The software responsible for **managing the execution and lifecycle** of the containers on a Node. Kubernetes supports any runtime that adheres to the **Container Runtime Interface (CRI)**, such as containerd or CRI-O.|
|**Container Environment**|This describes the various settings a container receives, including its **environment variables**, **command line arguments**, and **resource limits** (CPU, memory) defined in the Pod specification.|
|**Container Lifecycle Hooks**|These are actions that are triggered at specific points in a container's life, such as **postStart** (immediately after creation) and **preStop** (before the container is terminated).|

#### Container Environaments
- **The Filesystem:** A combination of the container **image** (read-only) and one or more **volumes** (for read/write data, which we'll cover later).
    
- **Container-Specific Information:** The container's **hostname** is set to the Pod name, and the Pod's name and namespace are exposed as environment variables using the **Downward API**.
    
- **Cluster Information (Service Discovery):** All running Services (in the same namespace, plus control plane services) are automatically exposed to the container in two ways:
    
    - **Environment Variables:** Variables like `FOO_SERVICE_HOST` and `FOO_SERVICE_PORT` are set for each Service.
        
    - **DNS:** Services are resolvable via DNS, which is the preferred and more flexible method.