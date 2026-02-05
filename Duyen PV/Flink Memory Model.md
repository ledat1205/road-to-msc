![[Screenshot 2025-10-15 at 00.30.57.png]]

## 1. Total Flink Memory

This is the memory Flink actually uses for your job. It consists of both **Heap** and **Off-Heap** components.

### **JVM Heap Memory**

- **Framework Heap:** Reserved for Flink's internal data structures and operations that run on the heap. Usually very small.
    
- **Task Heap:** This is where your actual user code runs. If you create objects in your `Map` or `FlatMap` functions, they live here.
    

### **Off-Heap Memory**

- **Managed Memory:** Managed by Flink’s own memory manager. It is used for batch operations (sorting, hash tables) and, most importantly, for the **RocksDB State Backend**.
    
- **Direct Memory:**
    
    - **Framework Off-heap:** Small amount for Flink framework's native overhead.
        
    - **Task Off-heap:** Reserved for user code that uses native/direct memory.
        
    - **Network Memory:** Crucial for data exchange between TaskManagers (shuffles). If you have a complex graph with many "all-to-all" edges, you may need to increase this.
        

---

## 2. Total Process Memory

This is the "Outer Box." It is what the operating system or container (like Kubernetes/YARN) sees as the total memory consumption of the TaskManager process.

- **Total Flink Memory:** (Everything listed above).
    
- **JVM Metaspace:** Memory for JVM class metadata.
    
- **JVM Overhead:** A "safety buffer" for native overhead, compile cache, and code cache. It's usually calculated as a fraction of the total process memory.


![[Pasted image 20260205232733.png]]

