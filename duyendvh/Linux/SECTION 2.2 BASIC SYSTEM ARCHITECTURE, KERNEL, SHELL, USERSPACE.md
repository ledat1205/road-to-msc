![[Screenshot 2025-11-16 at 17.07.47.png]]
![[Screenshot 2025-11-16 at 17.08.01.png]]
![[Screenshot 2025-11-16 at 17.09.19.png]]
![[Screenshot 2025-11-16 at 17.10.12.png]]
![[Screenshot 2025-11-16 at 17.10.40.png]]
![[Screenshot 2025-11-16 at 17.10.53.png]]
![[Screenshot 2025-11-16 at 17.11.13.png]]
![[Screenshot 2025-11-16 at 17.12.58.png]]
![[Screenshot 2025-11-16 at 17.13.21.png]]
![[Screenshot 2025-11-16 at 17.13.51.png]]
![[Screenshot 2025-11-16 at 17.14.39.png]]
![[Screenshot 2025-11-16 at 17.15.03.png]]
## Detailed Explanation of Kernel Memory Management

We've discussed how the Linux Kernel manages the system's memory (RAM and swap space). Here is a detailed summary of the core concepts, mechanisms, and components involved.

---

## 1. Virtual Memory and Address Space (Allocation)

The foundation of modern memory management is **Virtual Memory**.

- **Concept:** The Kernel gives every running program, called a **process**, its own isolated and continuous view of memory, known as the **Virtual Address Space**. This space is logically separated from the physical RAM.
    
- **Analogy:** Think of the Kernel giving every house (process) a street address map that _always_ starts at number 1, regardless of where the physical house is built in the city (RAM).
    
- **Mechanism:**
    
    - **Pages:** The virtual address space is divided into fixed-size units called **pages** (usually $4$KB).
        
    - **Page Frames:** Physical RAM is divided into corresponding slots called **page frames**.
        
- **Benefit:** This isolation provides **security** and **stability**. One program cannot directly read or write to the memory space of another, preventing accidental crashes or malicious attacks.
    

---

## 2. The Page Table and Translation

The mechanism that translates the virtual addresses into physical addresses is the **Page Table**.

- **Page Table:** A data structure maintained by the Kernel for **each process**. It holds the critical mapping information.
    
- **Page Table Entry (PTE):** Each entry in the Page Table corresponds to a virtual page and contains:
    
    - **The Physical Address:** The location in RAM (the Page Frame Number, or PFN).
        
    - **The `Present Bit`:** The most critical bit.
        
        - If `1` (set), the page is **in RAM**.
            
        - If `0` (cleared), the page is **not in RAM** (and is either on disk or needs to be loaded).
            
    - **Status Bits:** Flags like the `Dirty Bit` (was the page modified?) and the `Accessed Bit` (was the page recently used?).
        

---

## 3. Page Fault Handling

A Page Fault is the Kernel's main method for dynamically managing memory access.

- **The Trigger:** A process tries to access a virtual address whose corresponding **PTE has the `Present Bit` set to `0`**. The hardware (Memory Management Unit or MMU) cannot complete the translation and interrupts the CPU, passing control to the Kernel.
    
- **The Kernel's Action:**
    
    1. **Stops** the faulting process.
        
    2. **Identifies** the reason for the fault (e.g., the page is on disk, or it's a new page that needs to be created).
        
    3. If the page is on the swap file, the Kernel reads the **Swap Address** information stored in the PTE (which replaced the PFN when it was swapped out).
        
    4. **Loads** the required data from the disk (swap file or executable file) into an available RAM frame.
        
    5. **Updates the PTE:** Sets the `Present Bit` back to `1` and inserts the new PFN (the physical RAM address).
        
    6. **Resumes** the process.
        

---

## 4. Swapping and Inactive Pages

**Swapping** is the active process the Kernel uses to ensure there is always free physical memory available.

- **Inactive Page:** This is a memory page that contains **valid data** belonging to a running process (like data from a minimized application) but **has not been accessed recently**.
    
- **The Swap-Out Process:** When RAM runs low, the Kernel decides to move inactive pages to disk:
    
    1. **Write Data:** The Kernel copies the entire data content of the inactive page to a free slot in the designated **swap space** (a partition or file on the hard drive).
        
    2. **Update PTE:** The Kernel then updates the process's PTE for that page:
        
        - Clears the **`Present Bit`** (sets it to `0`).
            
        - Writes the **Swap Address** (location on disk) into the PFN field.
            
    3. **Free RAM:** The physical RAM frame is now marked as **free** and can be reused by an active process.
        
- **Thrashing:** This occurs when the system spends more time **swapping pages in and out** (disk I/O) than doing actual work, indicating the system is critically low on physical memory.