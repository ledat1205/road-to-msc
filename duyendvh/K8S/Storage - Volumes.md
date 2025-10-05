1. **Introduction to Volumes**:
    
    - Volumes provide a way for containers in a pod to access and share data via the filesystem.
    - Purposes include:
        - Populating configuration files (e.g., ConfigMap, Secret).
        - Providing temporary scratch space.
        - Sharing filesystems between containers or pods.
        - Durable storage persisting beyond pod restarts.
        - Passing pod metadata to containers.
        - Providing read-only access to data from other container images.
2. **Why Volumes Are Important**:
    
    - **Data Persistence**: Containers have ephemeral storage, so volumes ensure data survives container crashes or restarts.
    - **Shared Storage**: Volumes allow multiple containers in a pod to share files.
3. **How Volumes Work**:
    
    - Volumes are directories accessible to containers in a pod, defined in .spec.volumes and mounted via .spec.containers[*].volumeMounts.
    - Ephemeral volumes are tied to a pod’s lifetime; persistent volumes persist beyond it.
    - Data is preserved across container restarts but may be deleted for ephemeral volumes when a pod is removed.
    - Volumes cannot mount within other volumes or contain hard links to other volumes.
4. **Types of Volumes**: The document lists various volume types, including their status (e.g., deprecated, removed) and configuration examples where applicable. Below is a summary of each volume type mentioned:
    
    - **awsElasticBlockStore (deprecated)**:
        - Redirected to ebs.csi.aws.com CSI driver in Kubernetes 1.34.
        - Deprecated in v1.19, removed in v1.27.
        - Recommendation: Use the AWS EBS CSI driver.
    - **azureDisk (deprecated)**:
        - Redirected to disk.csi.azure.com CSI driver in Kubernetes 1.34.
        - Deprecated in v1.19, removed in v1.27.
        - Recommendation: Use the Azure Disk CSI driver.
    - **azureFile (deprecated)**:
        - Redirected to file.csi.azure.com CSI driver in Kubernetes 1.34.
        - Deprecated in v1.21, removed in v1.30.
        - Recommendation: Use the Azure File CSI driver.
    - **cephfs (removed)**:
        - Removed in Kubernetes 1.31, deprecated in v1.28.
        - No longer supported.
    - **cinder (deprecated)**:
        - Redirected to cinder.csi.openstack.org CSI driver in Kubernetes 1.34.
        - Deprecated in v1.11, removed in v1.26.
        - Recommendation: Use the OpenStack Cinder CSI driver.

- **configMap**:
    
    - Injects configuration data into pods.
    - Mounted as files from a ConfigMap.
    - **Example**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: configmap-pod
        spec:
          containers:
            - name: test
              image: busybox:1.28
              command: ['sh', '-c', 'echo "The app is running!" && tail -f /dev/null']
              volumeMounts:
                - name: config-vol
                  mountPath: /etc/config
          volumes:
            - name: config-vol
              configMap:
                name: log-config
                items:
                  - key: log_level
                    path: log_level.conf
        ```
        
        - Mounts the log-config ConfigMap’s log_level key as /etc/config/log_level.conf.
        
    
- **downwardAPI**:
    
    - Exposes pod metadata as read-only files.
    - No specific example provided, but referenced for further reading.
    
- **emptyDir**:
    
    - Provides a temporary, empty directory created when a pod is assigned to a node.
    - Deleted permanently when the pod is removed.
    - Can use disk or memory (tmpfs) as the storage medium.
    - **Example (default medium)**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: test-pd
        spec:
          containers:
          - image: registry.k8s.io/test-webserver
            name: test-container
            volumeMounts:
            - mountPath: /cache
              name: cache-volume
          volumes:
          - name: cache-volume
            emptyDir:
              sizeLimit: 500Mi
        ```
        
    - **Example (memory-backed)**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: test-pd
        spec:
          containers:
          - image: registry.k8s.io/test-webserver
            name: test-container
            volumeMounts:
            - mountPath: /cache
              name: cache-volume
          volumes:
          - name: cache-volume
            emptyDir:
              sizeLimit: 500Mi
              medium: Memory
        ```
        
    
- **fc (fibre channel)**:
    
    - Mounts an existing fibre channel block storage volume.
    - Supports single or multiple WWNs for multi-path connections.
    - No specific example provided.
    
- **hostPath**:
    
    - Mounts a file or directory from the host node’s filesystem.
    - Types include DirectoryOrCreate, Directory, FileOrCreate, File, Socket, CharDevice, BlockDevice.
    - **Example (Linux)**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: hostpath-example-linux
        spec:
          os: { name: linux }
          nodeSelector:
            kubernetes.io/os: linux
          containers:
          - name: example-container
            image: registry.k8s.io/test-webserver
            volumeMounts:
            - mountPath: /foo
              name: example-volume
              readOnly: true
          volumes:
          - name: example-volume
            hostPath:
              path: /data/foo
              type: Directory
        ```
        
    - **Example (Windows)**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: hostpath-example-windows
        spec:
          os: { name: windows }
          nodeSelector:
            kubernetes.io/os: windows
          containers:
          - name: example-container
            image: microsoft/windowsservercore:1709
            volumeMounts:
            - name: example-volume
              mountPath: "C:\\foo"
              readOnly: true
          volumes:
          - name: example-volume
            hostPath:
              path: "C:\\Data\\foo"
              type: Directory
        ```
        
    - **Example (FileOrCreate)**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: test-webserver
        spec:
          os: { name: linux }
          nodeSelector:
            kubernetes.io/os: linux
          containers:
          - name: test-webserver
            image: registry.k8s.io/test-webserver:latest
            volumeMounts:
            - mountPath: /var/local/aaa
              name: mydir
            - mountPath: /var/local/aaa/1.txt
              name: myfile
          volumes:
          - name: mydir
            hostPath:
              path: /var/local/aaa
              type: DirectoryOrCreate
          - name: myfile
            hostPath:
              path: /var/local/aaa/1.txt
              type: FileOrCreate
        ```
        
    
- **image**:
    
    - Feature state: Kubernetes v1.33 [beta].
    - Mounts an OCI object (container image or artifact) as a volume.
    - **Example**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: image-volume
        spec:
          containers:
          - name: shell
            command: ["sleep", "infinity"]
            image: debian
            volumeMounts:
            - name: volume
              mountPath: /volume
          volumes:
          - name: volume
            image:
              reference: quay.io/crio/artifact:v2
              pullPolicy: IfNotPresent
        ```
        
    
- **iscsi**:
    
    - Mounts an existing iSCSI volume.
    - Supports read-only mounts by multiple consumers but only single-writer read-write mounts.
    - No specific example provided.
    
- **local**:
    
    - Represents a mounted local storage device (disk, partition, directory).
    - Used as statically created PersistentVolumes, not dynamically provisioned.
    - **Example**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: PersistentVolume
        metadata:
          name: example-pv
        spec:
          capacity:
            storage: 100Gi
          volumeMode: Filesystem
          accessModes:
          - ReadWriteOnce
          persistentVolumeReclaimPolicy: Delete
          storageClassName: local-storage
          local:
            path: /mnt/disks/ssd1
          nodeAffinity:
            required:
              nodeSelectorTerms:
              - matchExpressions:
                - key: kubernetes.io/hostname
                  operator: In
                  values:
                  - example-node
        ```
        
    
- **nfs**:
    
    - Mounts an existing NFS share.
    - Supports multiple writers.
    - **Example**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: test-pd
        spec:
          containers:
          - image: registry.k8s.io/test-webserver
            name: test-container
            volumeMounts:
            - mountPath: /my-nfs-data
              name: test-volume
          volumes:
          - name: test-volume
            nfs:
              server: my-nfs-server.example.com
              path: /my-nfs-volume
              readOnly: true
        ```
        
    
- **persistentVolumeClaim**:
    
    - Mounts a PersistentVolume into a pod.
    - Allows users to claim durable storage without knowing cloud-specific details.
    - No specific example provided, but referenced for further reading.
    
- **projected**:
    
    - Maps multiple volume sources into the same directory.
    - No specific example provided, but referenced for further reading.
    
- **secret**:
    
    - Mounts sensitive data (e.g., passwords) as files from a Secret.
    - Backed by tmpfs (RAM-backed filesystem).
    - No specific example provided, but referenced for further reading.
    

5. **Using subPath**:
    
    - Allows sharing a single volume for multiple purposes within a pod.
    - **Example (LAMP stack)**:
        
        yaml
        
        ```
        apiVersion: v1
        kind: Pod
        metadata:
          name: my-lamp-site
        spec:
          containers:
          - name: mysql
            image: mysql
            env:
            - name: MYSQL_ROOT_PASSWORD
              value: "rootpasswd"
            volumeMounts:
            - mountPath: /var/lib/mysql
              name: site-data
              subPath: mysql
          - name: php
            image: php:7.0-apache
            volumeMounts:
            - mountPath: /var/www/html
              name: site-data
              subPath: html
          volumes:
          - name: site-data
            persistentVolumeClaim:
              claimName: my-lamp-site-data
        ```
        
    - **Note**: Not recommended for production use.
    - **subPathExpr**:
        
        - Constructs sub-paths using environment variables (Kubernetes v1.17 [stable]).
        - **Example**:
            
            yaml
            
            ```
            apiVersion: v1
            kind: Pod
            metadata:
              name: pod1
            spec:
              containers:
              - name: container1
                env:
                - name: POD_NAME
                  valueFrom:
                    fieldRef:
                      apiVersion: v1
                      fieldPath: metadata.name
                image: busybox:1.28
                command: [ "sh", "-c", "while [ true ]; do echo 'Hello'; sleep 10; done | tee -a /logs/hello.txt" ]
                volumeMounts:
                - name: workdir1
                  mountPath: /logs
                  subPathExpr: $(POD_NAME)
              restartPolicy: Never
              volumes:
              - name: workdir1
                hostPath:
                  path: /var/log/pods
            ```
            
        
    
6. **Resources**:
    
    - Discusses storage medium for emptyDir (depends on kubelet root dir, typically /var/lib/kubelet).
    - No limits on emptyDir or hostPath space consumption, and no isolation between containers/pods.
    
7. **Out-of-Tree Volume Plugins**:
    
    - **Container Storage Interface (CSI)**:
        
        - Standard interface for exposing storage systems.
        - Fields include driver, volumeHandle, readOnly, fsType, volumeAttributes, and secret references.
        - Supports raw block volumes (v1.18 [stable]) and ephemeral volumes (v1.25 [stable]).
        - Windows CSI proxy for privileged operations on Windows nodes (v1.22 [stable]).
        - **CSIMigration**: Redirects in-tree plugin operations to CSI drivers.
        
    
8. **Mount Propagation**:
    
    - Controls how mounts are propagated:
        
        - None: No propagation (default, equivalent to rprivate).
        - HostToContainer: Receives mounts from the host (equivalent to rslave).
        - Bidirectional: Propagates mounts to/from the host and other containers (equivalent to rshared).
        
    
9. **Read-Only Mounts**:
    - Set via .spec.containers[].volumeMounts[].readOnly: true.
    - Does not make the volume itself read-only; only the specific mount is read-only.
    - **Recursive Read-Only Mounts** (v1.33 [stable]):
        - Enabled by RecursiveReadOnlyMounts feature gate.
        - Requires Linux kernel v5.12+, CRI/OCI runtime support, and specific configuration.
        - **Example**:
            
            yaml
            
            ```
            apiVersion: v1
            kind: Pod
            metadata:
              name: rro
            spec:
              volumes:
                - name: mnt
                  hostPath:
                    path: /mnt
              containers:
                - name: busybox
                  image: busybox
                  args: ["sleep", "infinity"]
                  volumeMounts:
                    - name: mnt
                      mountPath: /mnt-rro
                      readOnly: true
                      mountPropagation: None
                      recursiveReadOnly: Enabled
                    - name: mnt
                      mountPath: /mnt-ro
                      readOnly: true
                    - name: mnt
                      mountPath: /mnt-rw
            ```