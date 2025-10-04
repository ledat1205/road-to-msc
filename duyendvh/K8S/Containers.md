
The document covers the fundamental concepts of what a container is and how it functions within the Kubernetes ecosystem:

- **Definition:** A container image is a **ready-to-run software package** that includes everything needed to execute an application: the code, runtime, system tools, and libraries.
    
- **Design Philosophy:** Containers are designed to be **stateless and immutable**. This means you should treat a running container as fixed. If you need to make changes, the correct procedure is to build a _new image_ with the updates and replace the old container with the new one.
    

### Key Components

|Component|Summary|
|---|---|
|**Container Runtimes**|The software responsible for **managing the execution and lifecycle** of the containers on a Node. Kubernetes supports any runtime that adheres to the **Container Runtime Interface (CRI)**, such as containerd or CRI-O.|
|**Container Environment**|This describes the various settings a container receives, including its **environment variables**, **command line arguments**, and **resource limits** (CPU, memory) defined in the Pod specification.|
|**Container Lifecycle Hooks**|These are actions that are triggered at specific points in a container's life, such as **postStart** (immediately after creation) and **preStop** (before the container is terminated).|

