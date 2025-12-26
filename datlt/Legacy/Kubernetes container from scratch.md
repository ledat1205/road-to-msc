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