### What is OCI?
- to create open industry standards for container formats and runtimes
- ensure interoperability and portability across different container tools and platforms

OCI focuses on 3 main specifications:
1. OCI Image Specification ([image-spec(opens in a new tab)](https://github.com/opencontainers/image-spec)):
The format for a container image. This includes how an image is structured on disk, its layers, manifest (metadata), and configuration
Image build from a tool or platform followed OCI can portable to other platform also followed OCI

2. OCI Runtime Specification ([runtime-spec(opens in a new tab)](https://github.com/opencontainers/runtime-spec)):
How a container runtime should execute a _filesystem bundle_ (an unpacked container image) and manage its lifecycle (create, start, stop, delete, etc.). It specifies the `config.json` file, which describes how the container process should be run (eg. entrypoint, environment variables, resource limits, security settings)
It ensures that different container runtimes can produce consistent execution environments for containers

3. OCI Distribution Specification ([distribution-spec(opens in a new tab)](https://github.com/opencontainers/distribution-spec)):
An API protocol for distributing container images. This standardizes how container registries (eg. Docker Hub, GCR, ECR, Harbor) store, pull, and push container images
It allows various container tools to interact with different registries, promoting a unified ecosystem for image distribution

### What is runc ?
is a lightweight, portable, low-level container runtime that serves as the reference implementation of the OCI Runtime Specification.

`runc` interacts directly with the Linux kernel's low-level features, specifically:

- Namespaces: Provide process isolation (PID, network, mount, IPC, UTS, user namespaces)
- Cgroups: Enforce resource limits (CPU, memory, I/O) on the container process
- `pivot_root`/`chroot`: Change the root filesystem of the process to the container's rootfs bundle
- Seccomp, AppArmor, SELinux: Apply security profiles for granular control over system calls and permissions

Example workflow in Docker:
1. `docker run nginx`: Docker CLI -> Docker daemon 
2. dockerd -.> containerd: Docker used containerd as its runtime. Dockerd send gRPC request to containerd
3. containerd prepare OCI bundle
	Note: create rootfs and config.json 
4. containerd executes: `runc create <container_id>`
5. `runc` does:
	1. `clone()`: create namespace
	2. `setns()`: join namespace
	3. `pivot_root()`: switch rootfs
	4. `cgroupfs()`: apply limits
	5. `execve()`: start user process
		After all runc exits, container process keep running

Kubernetes is kinda same with Docker's workflow. The difference is only `kubelet` will send request to `containerd`.

### What is CRI ?

`kubelet` is called is Container Runtime Interface (CRI). A Kubernetes API that standardizes how `kubelet` communicates with container runtimes.

CRI defines a gRPC API with two main services:

1. **ImageService**: Manages container images
    
    - Pull, list, remove, and inspect images
    - Image filesystem usage statistics
2. **RuntimeService**: Manages pods and containers
    
    - Create, start, stop, remove, and inspect containers
    - Create, stop, remove, and inspect pods (sandbox containers)
    - Execute commands in containers
    - Attach to containers
    - Port forwarding
The CRI acts as a translation layer, converting `kubelet`'s high-level requests (eg. create a pod) into the appropriate low-level operations that the container runtime can understand and execute.