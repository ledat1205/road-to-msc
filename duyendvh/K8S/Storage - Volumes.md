## 🗄️ Volumes: Why and How

**Why They're Important:**

1. **Data Persistence:** On-disk files in a container are lost when the container is stopped or crashes. Volumes preserve this data across **container restarts**.
    
2. **Shared Storage:** They provide a simple way for **multiple, tightly coupled containers** within the same Pod to share a filesystem and communicate.
    

**How They Work:**

- A volume is a **directory** that is accessible to the containers in a Pod.
    
- You define volumes in the Pod's `.spec.volumes` and then specify where to mount them within each container using `.spec.containers[*].volumeMounts`.
    
- **Ephemeral Volumes** (like `emptyDir`) are deleted when the Pod is removed.
    
- **Persistent Volumes** exist beyond the life of any single Pod.
    

---

## 💾 Key Volume Types & Purposes

The document outlines many types, categorized by their primary function:

|Volume Type|Purpose|Key Feature|
|---|---|---|
|**`emptyDir`**|Temporary scratch space or sharing files between containers.|Created when the Pod is scheduled; deleted when the Pod is deleted. Can be backed by **disk** or **Memory (tmpfs)**.|
|**`configMap`, `secret`**|Injecting configuration or sensitive data into the container's filesystem as files.|Mounted as **read-only**.|
|**`downwardAPI`**|Exposing Pod metadata (like Pod name, namespace) as files to the containers.|Mounted as **read-only**.|
|**`hostPath`**|Mounting a file or directory directly from the **Node's filesystem**.|**Caution:** Considered a security risk and generally discouraged for most apps.|
|**`persistentVolumeClaim` (PVC)**|The abstract way for a Pod to request **durable, network-backed storage**.|Requires a separate PersistentVolume (PV) to exist and handles the storage details outside the Pod definition.|

Xuất sang Trang tính

**Important Note:** Many older cloud-specific volume types (like `awsElasticBlockStore`, `azureDisk`) are now **deprecated or removed**. Kubernetes strongly suggests using **CSI (Container Storage Interface) drivers** for cloud and external storage instead of the built-in in-tree drivers.