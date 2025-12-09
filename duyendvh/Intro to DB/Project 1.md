### 1. `Evict()`

- **Purpose:** The core function that selects and returns a victim frame for eviction.
    
- **Logic:** Implements the actual ARC policy to choose a frame ID based on the internal T1 and T2 lists.
    
    - It first checks if any frame is currently marked as evictable.
        
    - It then applies the adaptive rules (comparing the MRU target size to the T1 list size) to decide whether to evict from the **T1 (Recency)** list or the **T2 (Frequency)** list.
        
    - If a victim is found, it is removed from the internal tracking lists, and its ID is returned.
        
- **Return Type:** Returns the `frame_id_t` of the victim frame, or `std::nullopt` if no evictable frames are available.
    

### 2. `RecordAccess(frame_id_t frame_id, page_id_t page_id)`

- **Purpose:** Notifies the replacer that a specific page in a frame has been accessed by a thread.
    
- **Logic:** This is where the complex state management of ARC happens:
    
    - It checks which of the four lists (T1, T2, B1, B2) the page ID belongs to.
        
    - If the page is currently **resident** (in T1 or T2), it moves the page to the front of **T2** (marking it as frequently used).
        
    - If the page is a **ghost hit** (in B1 or B2), it triggers the **adaptive size adjustment** of the T1 target size (`p`), proving the eviction decision needs tuning. The page is then moved into the T2 resident list.
        
    - If the page is a **miss** (not in any list), it is added to the front of **T1**.
        

### 3. `SetEvictable(frame_id_t frame_id, bool set_evictable)`

- **Purpose:** Allows the Buffer Pool Manager to explicitly control whether a frame can be chosen for eviction.
    
- **Logic:** This function is called by the BPM when a page's pin count changes:
    
    - When a page's pin count drops to zero, the BPM calls `SetEvictable(frame_id, **true**)` to make it available for eviction.
        
    - When the BPM pins an unpinned page (pin count goes from 0 to 1), it calls `SetEvictable(frame_id, **false**)` to protect it from eviction.
        
    - It also updates the replacer's internal count of available evictable frames.
        

### 4. `Size()`

- **Purpose:** Returns the current number of frames that are available to be evicted.
    
- **Logic:** Simply returns the count of frames currently marked as evictable. The BPM uses this to quickly check if an eviction is possible before committing to finding a victim.