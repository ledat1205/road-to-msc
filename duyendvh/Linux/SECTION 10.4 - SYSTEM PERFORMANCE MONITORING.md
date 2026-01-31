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

![[Screenshot 2025-11-30 at 20.18.51.png]]![[Screenshot 2025-11-30 at 23.21.18.png]]
![[Screenshot 2025-11-30 at 20.20.45.png]]

![[Screenshot 2025-11-30 at 20.21.21.png]]
![[Screenshot 2025-11-30 at 20.21.44.png]]

![[Screenshot 2025-11-30 at 17.19.24.png]]
![[Screenshot 2025-11-30 at 17.19.38.png]]
![[Screenshot 2025-11-30 at 17.19.54.png]]
![[Screenshot 2025-11-30 at 17.20.15.png]]

![[Screenshot 2025-11-30 at 17.16.25.png]]
![[Screenshot 2025-11-30 at 17.16.42.png]]
![[Screenshot 2025-11-30 at 17.16.57.png]]
![[Screenshot 2025-11-30 at 17.17.22.png]]
![[Screenshot 2025-11-30 at 17.17.38.png]]
![[Screenshot 2025-11-30 at 17.17.57.png]]

![[Pasted image 20260131180901.png]]
## 1. Reading the 8 Bars (The CPU Dashboard)

The bars numbered **0 through 7** represent the individual logical cores of your processor.

- **Color Meaning**: Each bar is composed of different colored segments that tell you _how_ that core is being used:
    
    - **Blue**: Low-priority processes (nice).
        
    - **Green**: Normal user-level processes (your apps like Chrome or VS Code).
        
    - **Red**: System/Kernel processes.
        
    - **Yellow**: Virtualized processes (guest OS).
        
- **Usage Percentage**: The number on the right (e.g., **59.1%**, **78.0%**) is the total utilization of that specific core.
    
- **The "Balanced" Load**: In your second screenshot, the cores are fairly evenly loaded (between 47% and 78%), which is healthy for a machine running heavy applications like Java, Chrome, and VS Code.
    

---

## 2. Memory and Swap Bars

Below the CPU bars, you have two critical resource meters:

- **Mem (RAM)**: In your screenshot, it shows **13.4G/16.0G**.
    
    - This means you are using about **84%** of your total physical RAM.
        
    - The white vertical bars represent used memory, while the darker area is free or cached.
        
- **Swp (Swap)**: It shows **2.19G/3.00G**.
    
    - **What this tells you**: Since your RAM is nearly full (13.4G), the system is moving less-active data to your disk (Swap) to make room. A high Swap usage with high RAM usage often indicates your machine is reaching its performance limit.
        

---

## 3. System Statistics Summary

Next to the bars, you see high-level system metrics:

- **Tasks**: **511** total processes with **2888** threads. This shows how busy the OS is managing different pieces of code.
    
- **Load Average**: **4.23, 2.90, 2.35**. These numbers represent the system load over the last 1, 5, and 15 minutes. On an 8-core machine, a load of 4.23 is well within capacity (anything under 8.00 means the CPU is not "waiting" for work).
    
- **Uptime**: **5 days, 17:43:07**. This is how long your system has been running since the last reboot.
    

---

## 4. Reading the Process Columns (The Table)

Using your specific screenshot as a reference, here is how to interpret the rows:

|**Column**|**Example from your Screenshot**|**Meaning**|
|---|---|---|
|**VIRT**|**1781G**|**Virtual Memory**: Often misleadingly high on macOS. It includes shared libraries and doesn't represent actual RAM usage.|
|**RES**|**217M**|**Resident Memory**: The _actual_ physical RAM this process is using. This is the most important memory number.|
|**CPU%**|**3.6**|The percentage of one CPU core used by this task.|
|**MEM%**|**1.7**|The percentage of total physical RAM (16GB) used by this task.|
|**S (State)**|**S** (Sleeping) / **R** (Running)|**S** means the process is idle; **R** means it is currently being calculated by a CPU core.|

---

## 5. Practical Debugging Steps

Based on your current view, here is how you can use `htop` to troubleshoot:

- **Find the "Hog"**: Press **F6** (or click "Setup") to sort. You can sort by **CPU%** to see what's using your processor or **RES** to see what's eating your RAM.
    
- **Filter**: Press **F4** and type "Chrome" to hide everything except Google Chrome processes.
    
- **Kill a Hang**: If an app is frozen, highlight it and press **F9** to send a "Kill" signal.
    
- **Tree View**: Press **F5** to see which processes "own" others (e.g., seeing which Chrome "Helper" belongs to which Tab).

![[Screenshot 2026-01-31 at 18.36.37.png]]

- **`TYPE`**:
    
    - **`DIR`**: A directory.
        
    - **`REG`**: A regular file.
        
    - **`CHR`**: A "Character Special" file (hardware/terminal device).
        
- **`DEVICE`**: The hardware ID of your disk (1,13 in this case).
    
- **`SIZE/OFF`**: For the terminal (`ttys042`), it shows the **Offset** (0t3366573), which essentially tracks how much data has passed through that terminal session.
    
- **`NODE`**: The **Inode number** on the disk. This is the unique physical address of the file.