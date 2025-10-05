- ConfigMaps
- Secrets
- Managing Resources for Containers
- Pod Overhead
- Assigning Pods to Nodes
- Taints and Tolerations
- Pod Topology Spread Constraints

- ### 1. ConfigMaps

- **URL**: [https://kubernetes.io/docs/concepts/configuration/configmap/](https://kubernetes.io/docs/concepts/configuration/configmap/)
- **Description**: ConfigMaps store non-confidential configuration data in key-value pairs, which can be consumed by pods or other Kubernetes resources.
- **Key Points**:
    - **Purpose**: Decouples configuration from application code, allowing portable and flexible configuration management.
    - **Use Cases**:
        - Set environment variables for containers.
        - Provide command-line arguments.
        - Mount configuration files as volumes.
    - **Creation**: ConfigMaps can be created using YAML/JSON manifests, kubectl create configmap, or from files/directories/literal values.
    - **Consumption**:
        - As environment variables (via env or envFrom in pod spec).
        - As volume mounts (mounted as files in a pod).
        - As command-line arguments (via args referencing ConfigMap data).
    - **Immutable ConfigMaps**: Supported (feature gate ImmutableEphemeralVolumes) to prevent updates, improving security and performance.
    - **Example**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: ConfigMap
        metadata:
          name: example-config
        data:
          key1: value1
          key2: value2
        ---
        apiVersion: v1
        kind: Pod
        metadata:
          name: configmap-pod
        spec:
          containers:
          - name: test-container
            image: busybox
            env:
            - name: KEY1
              valueFrom:
                configMapKeyRef:
                  name: example-config
                  key: key1
            volumeMounts:
            - name: config-volume
              mountPath: /etc/config
          volumes:
          - name: config-volume
            configMap:
              name: example-config
        ```
        
    - **Limitations**: ConfigMaps are not for sensitive data (use Secrets instead) and have a size limit (1 MiB in Kubernetes 1.34).
    - **Management**: Use kubectl or Helm (though no Helm chart example is provided) to manage ConfigMaps.

---

### 2. Secrets

- **URL**: [https://kubernetes.io/docs/concepts/configuration/secret/](https://kubernetes.io/docs/concepts/configuration/secret/)
- **Description**: Secrets store sensitive data (e.g., passwords, tokens) securely and make it available to pods.
- **Key Points**:
    - **Purpose**: Safely manage sensitive information without hardcoding it in pod specs or images.
    - **Types**:
        - Generic (Opaque): Arbitrary key-value pairs.
        - Docker registry credentials (docker-registry).
        - TLS certificates (tls).
    - **Creation**: Via kubectl create secret, YAML manifests, or from files.
    - **Consumption**:
        - As environment variables.
        - As volume mounts (backed by tmpfs for security).
        - For image pull secrets to authenticate with private registries.
    - **Security**:
        - Secrets are stored in etcd, optionally encrypted at rest (requires EncryptionConfiguration).
        - Access is controlled via RBAC.
        - Secrets are not written to disk unless mounted as tmpfs.
    - **Example**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: Secret
        metadata:
          name: my-secret
        type: Opaque
        data:
          username: YWRtaW4= # base64-encoded
          password: MWYyZDFlMmU2N2Rm # base64-encoded
        ---
        apiVersion: v1
        kind: Pod
        metadata:
          name: secret-pod
        spec:
          containers:
          - name: test-container
            image: nginx
            env:
            - name: USERNAME
              valueFrom:
                secretKeyRef:
                  name: my-secret
                  key: username
            volumeMounts:
            - name: secret-volume
              mountPath: /etc/secret
          volumes:
          - name: secret-volume
            secret:
              secretName: my-secret
        ```
        
    - **Immutable Secrets**: Supported to prevent updates, improving security.
    - **Best Practices**: Use minimal Secrets, avoid exposing in logs, and consider external secret management systems.

---

### 3. Managing Resources for Containers

- **URL**: [https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- **Description**: Explains how to specify CPU and memory requests and limits for containers to optimize resource allocation.
- **Key Points**:
    - **Requests and Limits**:
        - **Requests**: Minimum resources a container needs (used by the scheduler for pod placement).
        - **Limits**: Maximum resources a container can use (enforced by the container runtime).
    - **Resource Types**:
        - CPU: Measured in cores or millicores (e.g., 500m = 0.5 CPU).
        - Memory: Measured in bytes (e.g., 128Mi, 1Gi).
        - Extended resources (e.g., GPUs): Vendor-specific.
    - **Impact**:
        - Pods with requests are scheduled on nodes with sufficient capacity.
        - Limits prevent resource hogging; exceeding limits may lead to container termination (OOM for memory) or CPU throttling.
    - **Example**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: resource-pod
        spec:
          containers:
          - name: test-container
            image: nginx
            resources:
              requests:
                memory: "64Mi"
                cpu: "250m" # 0.25 CPU
              limits:
                memory: "128Mi"
                cpu: "500m" # 0.5 CPU
        ```
        
    - **Quality of Service (QoS)**:
        - **Guaranteed**: Requests equal limits.
        - **Burstable**: Requests less than limits.
        - **BestEffort**: No requests or limits specified.
    - **Best Practices**: Set requests conservatively, limits to prevent overuse, and monitor resource usage.

---

### 4. Pod Overhead

- **URL**: [https://kubernetes.io/docs/concepts/configuration/pod-overhead/](https://kubernetes.io/docs/concepts/configuration/pod-overhead/)
- **Description**: Describes pod overhead, which accounts for resources consumed by the pod’s runtime environment.
- **Key Points**:
    - **Purpose**: Ensures accurate resource accounting for pods, especially when using runtime extensions (e.g., Kata Containers, virtlet).
    - **Definition**: Overhead is the additional CPU and memory consumed by the pod’s runtime (e.g., container runtime, networking stack).
    - **Configuration**: Defined in the RuntimeClass resource, applied to pods via runtimeClassName.
    - **Impact**: Overhead is added to pod resource requests during scheduling, ensuring nodes have sufficient capacity.
    - **Example**:
        
        yaml
        
        ```
        apiVersion: node.k8s.io/v1
        kind: RuntimeClass
        metadata:
          name: kata
        handler: kata
        overhead:
          podFixed:
            memory: "120Mi"
            cpu: "250m"
        ---
        apiVersion: v1
        kind: Pod
        metadata:
          name: overhead-pod
        spec:
          runtimeClassName: kata
          containers:
          - name: test-container
            image: nginx
            resources:
              requests:
                memory: "64Mi"
                cpu: "250m"
              limits:
                memory: "128Mi"
                cpu: "500m"
        ```
        
    - **Feature Status**: Stable in Kubernetes v1.18.
    - **Use Case**: Critical for environments using heavyweight runtimes or virtualized containers.

---

### 5. Assigning Pods to Nodes

- **URL**: [https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- **Description**: Details mechanisms to control which nodes pods are scheduled on.
- **Key Points**:
    - **Mechanisms**:
        - **Node Selector**: Simple key-value pairs to match node labels (e.g., nodeSelector: disktype=ssd).
        - **Node Affinity/Anti-Affinity**: Advanced rules for preferring or requiring specific nodes based on labels.
        - **Taints and Tolerations**: Used to repel pods from nodes unless they have matching tolerations (covered in the next section).
    - **Node Affinity Types**:
        - requiredDuringSchedulingIgnoredDuringExecution: Pod must be scheduled on matching nodes.
        - preferredDuringSchedulingIgnoredDuringExecution: Scheduler prefers matching nodes but can fall back.
    - **Example (Node Selector)**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: node-selector-pod
        spec:
          nodeSelector:
            disktype: ssd
          containers:
          - name: nginx
            image: nginx
        ```
        
    - **Example (Node Affinity)**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: affinity-pod
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: kubernetes.io/hostname
                    operator: In
                    values:
                    - node-1
          containers:
          - name: nginx
            image: nginx
        ```
        
    - **Use Cases**: Ensure pods run on nodes with specific hardware, in specific zones, or avoid certain nodes.
    - **Limitations**: Node affinity is more flexible than node selectors but requires careful label management.

---

### 6. Taints and Tolerations

- **URL**: [https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- **Description**: Explains how to use taints and tolerations to control pod scheduling by repelling pods from nodes.
- **Key Points**:
    - **Taints**: Applied to nodes to prevent pods from scheduling unless they have matching tolerations.
    - **Tolerations**: Specified in pod specs to allow scheduling on tainted nodes.
    - **Effects**:
        - NoSchedule: Pods without tolerations cannot schedule.
        - PreferNoSchedule: Scheduler avoids tainted nodes if possible.
        - NoExecute: Evicts running pods without tolerations.
    - **Example**:
        
        yaml
        
        ```
        # Apply taint to a node
        kubectl taint nodes node1 key1=value1:NoSchedule
        ---
        apiVersion: v1
        kind: Pod
        metadata:
          name: toleration-pod
        spec:
          tolerations:
          - key: "key1"
            operator: "Equal"
            value: "value1"
            effect: "NoSchedule"
          containers:
          - name: nginx
            image: nginx
        ```
        
    - **Use Cases**:
        - Dedicate nodes to specific workloads (e.g., GPU nodes).
        - Evict pods from nodes undergoing maintenance.
        - Isolate critical workloads.
    - **Default Taints**: Nodes with node.kubernetes.io/not-ready or node.kubernetes.io/unreachable are tainted with NoExecute.

---

### 7. Pod Topology Spread Constraints

- **URL**: [https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- **Description**: Describes how to distribute pods across topology domains (e.g., nodes, zones) for high availability and load balancing.
- **Key Points**:
    - **Purpose**: Ensures even pod distribution to improve resilience and resource utilization.
    - **Topology Domains**: Defined by labels (e.g., topology.kubernetes.io/zone, kubernetes.io/hostname).
    - **Constraints**:
        - maxSkew: Maximum difference in the number of pods between any two topology domains.
        - whenUnsatisfiable: DoNotSchedule (block scheduling) or ScheduleAnyway (allow but prefer even distribution).
    - **Example**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: topology-pod
          labels:
            app: my-app
        spec:
          topologySpreadConstraints:
          - maxSkew: 1
            topologyKey: topology.kubernetes.io/zone
            whenUnsatisfiable: DoNotSchedule
            labelSelector:
              matchLabels:
                app: my-app
          containers:
          - name: nginx
            image: nginx
        ```
        
    - **Use Cases**:
        - Distribute pods across availability zones for fault tolerance.
        - Balance workload across nodes to prevent overloading.
    - **Feature Status**: Stable in Kubernetes v1.25.
    - **Considerations**: Requires proper node labeling and works best with multiple replicas (e.g., in Deployments).