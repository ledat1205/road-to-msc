The core concept is that a container image is a **read-only template** (or blueprint) containing everything needed to run your application.

---

### 1. Image Sources and Naming

- **Images and Registries:** Images are stored in **Container Registries** (like Docker Hub or a private registry). The **Kubelet** uses the full image name, which includes the registry and repository, to locate and download the image.
    
- **Tags:** Images are identified by a **tag**, which usually points to a specific version of the application. If no tag is specified, Kubernetes defaults to the `:latest` tag.
    
If you don't specify a registry hostname, Kubernetes assumes that you mean the Docker public registry
### 2. Image Pull Policy

This is a critical setting that tells the Kubelet when it should try to download the image from the registry. The three main policies are:

|Policy|Behavior|
|---|---|
|**`IfNotPresent`**|Pulls the image **only if** it is not already present locally on the Node. (This is the default for most non-`:latest` tags).|
|**`Always`**|The Kubelet **always** checks the registry to ensure it's using the latest version of the image. (This is the default for the `:latest` tag).|
|**`Never`**|The Kubelet **never** tries to pull the image and expects it to be pre-loaded on the Node.|

### 3. Private Registry Access

To use images stored in a **private registry**, the document explains you must provide the necessary credentials (username/password) by creating a special Kubernetes object called a **Secret** of type `kubernetes.io/dockerconfigjson`. This Secret is then referenced in the Pod specification.