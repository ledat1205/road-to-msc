![[Screenshot 2025-11-19 at 00.34.11.png]]
![[Screenshot 2025-11-19 at 00.34.34.png]]
![[Screenshot 2025-11-19 at 00.34.53.png]]
![[Screenshot 2025-11-19 at 00.35.08.png]]
![[Screenshot 2025-11-19 at 00.35.25.png]]
![[Screenshot 2025-11-19 at 00.35.53.png]]

![[Screenshot 2025-11-19 at 00.36.11.png]]
![[Screenshot 2025-11-19 at 00.36.57.png]]
![[Screenshot 2025-11-19 at 00.37.50.png]]
![[Screenshot 2025-11-19 at 00.40.35.png]]
![[Screenshot 2025-11-19 at 00.41.22.png]]

![[Screenshot 2025-11-19 at 00.42.04.png]]
![[Screenshot 2025-11-19 at 00.42.41.png]]
## 💡 Low vs. High Swap Usage Indicators

Swap usage reflects how a system is managing its virtual memory relative to its physical RAM.
### 🟢 Low Swap Usage (Ideal or Proactive)

|Indicator|Meaning|Performance Implication|
|---|---|---|
|**0% Used**|The system has **more than enough physical RAM** for the current workload. All active data is in RAM.|**Excellent Performance.** Data access is fast (RAM speed).|
|**Low, Stable Use**|The Linux kernel is proactively using swap (based on **`vm.swappiness`**) to move very old, inactive data pages out of RAM, freeing up RAM for the **disk cache** (file system cache).|**Generally Good.** Optimizes RAM utilization. Performance is not impacted unless the swapped data is suddenly needed.|

### 🔴 High Swap Usage (Memory Pressure)

|Indicator|Meaning|Performance Implication|
|---|---|---|
|**High, Steadily Increasing Use**|The system is experiencing **significant memory pressure**. Applications need more memory than the physical RAM can provide.|**Poor Performance.** The system relies heavily on slow disk I/O to handle memory requests.|
|**Fluctuating Use (Thrashing)**|The system is **constantly moving data pages** in and out of swap. The required data isn't staying in RAM long enough.|**Critical Performance Killer.** The OS spends most of its time swapping (I/O) instead of executing tasks. Requires more RAM.|
|**Swap is Full**|**Exhausted Memory.** The system has run out of both RAM and swap space.|**System Crash/Freeze.** Triggers the **OOM (Out-Of-Memory) killer** to forcibly terminate processes to free resources.|

![[Screenshot 2025-11-19 at 00.44.08.png]]
-> unmount ensure 5GB of buff/ xache wont be lost