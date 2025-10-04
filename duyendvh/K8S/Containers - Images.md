## Container Images: Concepts & Naming

- **Definition:** An image is a binary, executable software bundle containing an application and all its dependencies. It is the core unit deployed by Kubernetes.
    
- **Registries:** Images are stored in **Container Registries** (like Docker Hub). If no registry hostname is specified in the image name, Kubernetes defaults to the Docker public registry.
    
- **Image Names:** The full name can include the registry hostname and port (e.g., `fictional.registry.example:10443/imagename`).
    
- **Tags vs. Digests:**
    
    - **Tag** (`:v1.0`): Identifies different versions of an image, but the tag can be **moved** to point to different binary content over time. The `:latest` tag is discouraged for production use for this reason.
        
    - **Digest** (`@sha256:...`): A unique, immutable hash of the image content. Specifying an image by digest guarantees you always pull the exact same binary.
        

---

## Private Registry Access

Accessing images in private, authenticated registries is handled using several methods:

1. **`imagePullSecrets` (Recommended):** You create a Kubernetes **Secret** of type `kubernetes.io/dockerconfigjson` containing the registry credentials. This Secret is then referenced in the Pod's `imagePullSecrets` field. This is configured per-Pod and per-Namespace.
    
2. **Node Configuration:** A cluster administrator can configure credentials directly on the worker Nodes. This allows all Pods on that Node to access the registry.
    
3. **Kubelet Credential Provider (Advanced):** A plugin that allows the Kubelet to dynamically fetch credentials for private registries, which is especially useful for **Static Pods** that can't reference external Secrets.
    

---

## Image Pull Policies & Defaults

The `imagePullPolicy` dictates when the **Kubelet** attempts to download an image from the registry.

|Policy|Behavior|Default Behavior|
|---|---|---|
|**`Always`**|The Kubelet **always** queries the registry to check for a new digest.|Default if the image tag is `:latest` or no tag is specified.|
|**`IfNotPresent`**|The image is pulled **only if** it is not already present locally on the Node.|Default if the image tag is explicitly set to anything **other than** `:latest`.|
|**`Never`**|The Kubelet **never** tries to fetch the image and relies on it being pre-loaded.|Only set manually.|


### Defaulting and Overrides

- If you omit the `imagePullPolicy`, Kubernetes sets a default based on the tag as noted above.
    
- If you change an image tag on an existing resource (like changing from `:v1` to `:latest`), the `imagePullPolicy` **does not automatically change**; you must update it manually.
    
- **`ImagePullBackOff`:** This is the error status you see when the Kubelet fails to pull an image (due to wrong name, bad credentials, etc.). It indicates Kubernetes is retrying with an increasing delay (up to a limit of 5 minutes).
    

---

##  Advanced Image Management

- **Serial vs. Parallel Pulls:** By default, the Kubelet pulls images one at a time (**serially**). You can configure `serializeImagePulls: false` in the Kubelet config to enable parallel pulls.
    
- **Multi-Architecture Images:** Registries can serve a **container image index**, allowing a single image name (e.g., `pause`) to automatically resolve to the correct architecture-specific binary (e.g., `pause-amd64` or `pause-arm64`).