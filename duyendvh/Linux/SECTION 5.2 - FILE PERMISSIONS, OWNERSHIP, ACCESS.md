![[Screenshot 2025-11-16 at 19.43.58.png]]

![[Screenshot 2025-11-16 at 19.44.26.png]]

![[Screenshot 2025-11-16 at 19.44.12.png]]

![[Screenshot 2025-11-16 at 19.44.48.png]]

![[Screenshot 2025-11-16 at 19.45.16.png]]

![[Screenshot 2025-11-16 at 19.45.31.png]]

![[Screenshot 2025-11-16 at 19.47.08.png]]

![[Screenshot 2025-11-16 at 19.47.39.png]]

![[Screenshot 2025-11-16 at 19.48.08.png]]

![[Screenshot 2025-11-16 at 19.49.46.png]]

In the context of k8s config (e.g: druid historical):
- **`securityContext: runAsUser: 1000`:** This forces the main Historical process to execute as user ID 1000.
    
- **`securityContext: fsGroup: 1000`:** This ensures that all persistent volumes (like the `datadir` PVC) mounted to the Pod are owned by, or group-writable by, GID 1000.
    
- **`initContainer` Command:** The command `chown 1000:1000 /tiki/druid_var` explicitly sets the ownership of the segment cache directory to the numeric user ID 1000 and group ID 1000, ensuring the Historical process has the necessary **read/write** access to its own data directory.
![[Screenshot 2025-11-18 at 23.44.55.png]]
![[Screenshot 2025-11-18 at 23.45.32.png]]