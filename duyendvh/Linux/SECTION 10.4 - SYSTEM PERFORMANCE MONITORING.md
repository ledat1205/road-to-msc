# **1️⃣ What is Swap?**

- Swap is a **space on your disk (or a swap file/partition)** that Linux/Unix uses as an **overflow area for RAM**.
    
- When your **physical RAM is full**, the system can **move inactive pages of memory** from RAM to swap, freeing RAM for active processes.
    
- Swap is **slower than RAM** because it resides on disk.
    

---

# **2️⃣ Why Swap Exists**

1. **Memory overflow prevention**
    
    - Prevents processes from being killed immediately when RAM runs out.
        
2. **Hibernation**
    
    - Linux can store the content of RAM in swap during hibernation.
        
3. **System stability**
    
    - Keeps system responsive under heavy load.
        

---

# **3️⃣ How Swap Works**

- **Pages in memory:** Linux divides memory into “pages” (usually 4KB each).
    
- **Inactive pages** (not used recently) can be **paged out** to swap.
    
- When a program accesses a paged-out page, a **page fault** occurs → Linux reads it back into RAM.
    

---

# **4️⃣ How to Check Swap Usage**

1. **`free` command**
    

`free -h`

Example output:

              `total        used        free      shared  buff/cache   available Mem:           7.8G        3.2G        1.0G        512M        3.6G        3.8G Swap:          2.0G        512M        1.5G`

- `total` → total swap available
    
- `used` → how much swap is currently used
    
- `free` → remaining swap
    

2. **`swapon -s` or `cat /proc/swaps`**
    

`cat /proc/swaps Filename                                Type        Size    Used    Priority /swapfile                               file        2097148 524288  -2`

3. **`vmstat`**
    

`vmstat 1`

- Columns `si` (swap in) and `so` (swap out) show ongoing swap activity.
    

---

# **5️⃣ Swap Usage Best Practices**

- **Minimal swap usage** is ideal — heavy swap usage may indicate RAM shortage.
    
- **Swap size** recommendations:
    
    - ≤2GB RAM → swap = 2×RAM
        
    - 4–16GB RAM → swap = RAM or 1×RAM
        
    - > 32GB RAM → swap = 4–8GB (mostly for hibernation)
        
- **Swapiness setting** (`/proc/sys/vm/swappiness`) controls how aggressively Linux uses swap:
    

`cat /proc/sys/vm/swappiness  # default 60 # lower value → use RAM more, swap less # higher value → swap earlier`

---

# **6️⃣ When Swap Usage is Bad**

- Swap usage is not necessarily bad if **RAM is fully used but active processes are running smoothly**.
    
- Swap becomes a problem when **heavy swapping (thrashing) slows down the system**, i.e., constant page-in/page-out.
    

---

# **7️⃣ Summary**

- Swap = disk space acting as “backup RAM”
    
- Used when RAM is full or for hibernation
    
- Check with `free -h` or `swapon -s`
    
- Control usage with `swappiness`
    
- High swap with slow system = problem; low swap = healthy system
    

---
![[Screenshot 2025-11-30 at 17.19.24.png]]


![[Screenshot 2025-11-30 at 17.16.25.png]]
![[Screenshot 2025-11-30 at 17.16.42.png]]
![[Screenshot 2025-11-30 at 17.16.57.png]]
![[Screenshot 2025-11-30 at 17.17.22.png]]
![[Screenshot 2025-11-30 at 17.17.38.png]]
![[Screenshot 2025-11-30 at 17.17.57.png]]
