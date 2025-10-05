## 📂 Organizing & Applying Resources

- **Grouping Resources:** You can manage multiple related resources (like a Deployment and its corresponding Service) in a single YAML file by separating them with `---`.
    
- **Creation Order:** When deploying multiple resources from one file using `kubectl apply`, they are created in the order they appear. It's recommended to list the **Service first** to help the scheduler optimize Pod placement.
    
- **Bulk Application:** `kubectl apply` accepts multiple files or URLs in one command, simplifying application deployment.
    
- **Recursive Operations:** The `--recursive` or `-R` flag allows `kubectl` to process manifest files across multiple subdirectories, which is essential for organized projects.
    

---

## 🛠️ Bulk Operations and Utility Tools

- **Bulk Deletion:** Resources can be deleted in bulk using:
    
    - `kubectl delete -f <file>` (deleting resources defined in a file).
        
    - `kubectl delete <resource>/<name>` (specifying resource names).
        
    - `kubectl delete -l <label-query>` (filtering by labels).
        
- **Chaining and Filtering:** `kubectl` outputs resource names in a format it accepts, allowing you to chain commands using `xargs` or `$()` for complex, filtered operations.
    
- **External Tools:**
    
    - **Helm:** A tool for managing **packages** of pre-configured Kubernetes resources, known as **Helm charts**.
        
    - **Kustomize:** A tool for declaratively customizing Kubernetes manifests to add, remove, or update configurations.
        

---

## ⬆️ Updating Your Application (Rollouts)

- **Rolling Updates:** Workload resources (Deployment, StatefulSet, DaemonSet) support rolling updates to gradually replace old Pods with new ones, ensuring no outage.
    
    - **Mechanism:** This process is declarative. You update the image tag (or other fields) using `kubectl edit` or `kubectl set image`, and the Deployment controller manages the progressive update based on defined **surge and unavailable limits**.
        
    - **Rollout Management:** You use `kubectl rollout status` to monitor the update, and can also **pause, resume, or cancel** a rollout.
        
- **Canary Deployments:** A technique for directing a small amount of live traffic to a new release (**canary**) alongside the stable release. This is achieved using **labels** (e.g., `track: stable` and `track: canary`) and having the Service select the common, overarching label.
    

---

## 🎯 Scaling and Patching

- **Manual Scaling:** You can manually scale a workload (e.g., a Deployment) by changing the replica count using `kubectl scale deployment/<name> --replicas=<count>`.
    
- **Automatic Scaling:** You can use `kubectl autoscale` to create a **HorizontalPodAutoscaler (HPA)**, which automatically adjusts the replica count between defined minimum and maximum limits based on utilization metrics.
    
- **In-Place Updates:** For narrow, non-disruptive changes:
    
    - **`kubectl apply`:** Recommended for configuration-as-code, it compares your local file to the live resource and applies only the changes, without overwriting automated fields.
        
    - **`kubectl edit`:** Opens the live resource manifest in a text editor for more extensive, immediate changes.
        
    - **`kubectl patch`:** Used for specific in-place updates using JSON or strategic merge patches.
        
- **Disruptive Updates:** To change immutable fields or forcefully fix a resource, use **`kubectl replace --force`**, which deletes and immediately re-creates the resource.