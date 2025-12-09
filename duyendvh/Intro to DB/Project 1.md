Here is a breakdown of what the Buffer Pool Manager does:

### 1. Primary Function: Caching Data

The BPM serves as a **cache** for the data stored on disk.

- **Goal:** To keep frequently accessed data pages in memory (in structures called **frames**) to avoid slow I/O operations to the disk.
    
- **Mechanism:** When a component (like the query execution engine) needs a specific data page, the BPM first checks if the page is already in one of its frames (a **buffer pool hit**). If it is, the page is returned immediately.
    
- **Page vs. Frame:** A **page** is the logical unit of data (e.g., 8 KB) on disk. A **frame** is the fixed-size block of physical memory (RAM) where a page is temporarily stored.
    

### 2. Managing I/O and Eviction

When the requested page is **not** in memory (a **buffer pool miss**), the BPM must load it from disk. This requires careful management of the limited memory space:

- **Fetching:** It sends a request to the disk subsystem to read the required page into an available memory frame.
    
- **Eviction:** If all memory frames are full, the BPM must choose an existing, unpinned page to remove (evict) to make room for the new page. This is handled by a **Replacement Policy** (like ARC, LRU, or LRU-K).
    
- **Write-Back (Flushing):** If the page chosen for eviction has been modified while in memory (it is "dirty"), the BPM must first schedule a write operation to flush the modified version back to disk before overwriting the frame with the new page.
    

### 3. Concurrency Control and Integrity

The BPM ensures data integrity and supports concurrent access by multiple threads:

- **Pin Count:** It maintains a **pin count** (or reference count) for each page. Any page with a pin count greater than zero is actively being used by one or more threads and **cannot be evicted**.
    
- **Page Table:** It uses a hash map (**page table**) to track which logical page ID is currently stored in which physical frame ID, allowing for fast lookups.
    
- **Thread Safety:** The entire manager and its internal data structures must be protected by **latches (locks)** to prevent race conditions when multiple threads try to access or modify pages concurrently.


## 1. Implement the Adaptive Replacement Cache (ARC) Replacer (Task #1) 💾

The ARC Replacer is a standalone component, making it the best starting point.

- **Data Structures:** You will need to implement the logic for the four main lists that define the ARC policy:
    
    - **T1 & T2:** The MRU (most recently used, accessed once) and MFU (most frequently used, accessed multiple times) lists for pages _in_ the buffer pool.
        
    - **B1 & B2:** The "ghost" lists for recently evicted pages, which are crucial for the _adaptive_ part of the policy.
        
    - Use appropriate **STL containers** (e.g., `std::list` for ordered lists and `std::unordered_map` for fast lookups) to manage these lists and quickly find a page's location.
        
- **Core Logic:** Focus on the two primary methods:
    
    - **`RecordAccess()`:** Implements the complex state transitions described in the project specification. This method updates the page's position across the four lists and adaptively adjusts the target size for the MRU list (`p`).
        
    - **`Evict()`:** Determines which frame to evict based on the adaptive policy (prioritizing MRU or MFU eviction based on the current target size).
        
- **Thread Safety:** Since the BPM will call the replacer concurrently, you **must use a latch** (like `std::mutex`) to protect all internal data structures of the `ArcReplacer` from race conditions.
    

---

## 2. Implement the Disk Scheduler (Task #2) 💿

The Disk Scheduler handles all disk I/O asynchronously, preventing the main thread from blocking.

- **Channel Implementation:** Utilize the provided `Channel` class (a thread-safe queue) to store `DiskRequest` structs.
    
    - **Enqueue:** When the BPM needs to read or write a page, it will add a request to this queue.
        
    - **Dequeue:** The scheduler's background thread continuously polls the queue for new requests.
        
- **Background Worker Thread:** Create a persistent thread (e.g., using `std::thread`) within the `DiskScheduler` constructor.
    
    - This thread's primary loop should repeatedly:
        
        1. Wait for and dequeue a `DiskRequest`.
            
        2. Call the underlying `DiskManager`'s `ReadPage()` or `WritePage()` methods.
            
        3. Signal the requesting component (the BPM) that the I/O is complete (often done via a `std::promise/std::future` mechanism embedded in the request struct).
            
- **Cleanup:** Implement the destructor to safely shut down the background thread and ensure it finishes processing any remaining requests.
    

---

## 3. Implement the Buffer Pool Manager (BPM) (Task #3) 🧠

The BPM ties everything together and must be the final component you implement.

### A. Data Structures and Initialization

- **Page Table:** Use a thread-safe hash map (`std::unordered_map<page\_id\_t, frame\_id\_t>`) to map a page ID to the frame that currently holds it.
    
- **Free List:** Maintain a list of currently available (unoccupied) frame IDs.
    
- **Frame Headers:** Manage the necessary metadata for each frame: `pin_count_` (atomic), `is_dirty_`, and the actual `page_id_`.
    
- **Integration:** The BPM class must contain instances of your implemented **`ArcReplacer`** and **`DiskScheduler`**.
    

### B. Core BPM Logic Flow

The most complex part is managing the workflow for fetching and evicting pages, which requires strict adherence to concurrency rules.

#### **`FetchPage(page_id)`**

1. **Acquire Latch:** Lock the BPM's main latch to protect its internal data structures (page table, free list, etc.).
    
2. **Page Table Check (Hit):** Check the page table. If the page is already in a frame (a **buffer pool hit**):
    
    - Increment the frame's `pin_count_`.
        
    - Call the `ArcReplacer::RecordAccess()` method.
        
    - If the page's old pin count was 0, call `ArcReplacer::SetEvictable(false)`.
        
    - Release the latch and return the page.
        
3. **Page Table Miss (Miss):** If the page is not in memory:
    
    - **Find a Free Frame:**
        
        - First, check the **free list**. If a free frame is available, pop it.
            
        - If the free list is empty, call **`ArcReplacer::Evict()`** to find a victim frame. If `Evict()` fails (no evictable pages), return an error.
            
    - **Eviction (if necessary):** If a victim frame was found:
        
        - If the victim page's `is\_dirty\_` flag is true, schedule a write-back operation via the **`DiskScheduler`** (this is where you may need to release and re-acquire your latch, or use promises, to avoid blocking while waiting for I/O).
            
        - Update the page table to remove the victim page's entry.
            
    - **Load New Page:**
        
        - Update the frame's metadata to store the new `page\_id`.
            
        - Schedule a read operation via the **`DiskScheduler`** to load the data from disk into the frame.
            
    - **Final Update:** Increment the new page's `pin_count_`, update the page table with the new mapping, call `ArcReplacer::RecordAccess()`, release the latch, and return the page.
        

#### **`UnpinPage(page_id, is_dirty)`**

1. Decrement the frame's `pin_count_`.
    
2. If `is_dirty` is true, set the frame's `is_dirty_` flag to true.
    
3. If the `pin_count_` drops to zero, call **`ArcReplacer::SetEvictable(true)`**.