![[Screenshot 2025-11-19 at 00.18.55.png]]
![[Screenshot 2025-11-19 at 00.19.58.png]]




|**Scenario**|**Description**|**Outcome**|
|---|---|---|
|**Improper Shutdown/Power Loss**|The most common cause. Turning off power, forcefully shutting down, or a sudden power failure interrupts ongoing writes, leaving the file system in an **inconsistent** state (the data blocks might be written, but the index is not updated).|Inconsistent metadata, orphaned files, incorrect free block counts.|
|**Hardware Failure**|**Bad sectors** on a hard drive or flash media. If critical metadata is written to a sector that suddenly fails, the file system structure becomes corrupted.|Metadata corruption (superblock, inode tables), permanent data loss in the affected area.|
|**Software/Driver Bugs**|Errors within the OS kernel, device drivers, or sometimes even complex applications can lead to incorrect data being written to the file system's structure.|Corruption of specific file system structures, sometimes localized.|
|**Improper Device Ejection**|Removing an external drive (USB, etc.) without safely ejecting it prevents the OS from flushing its write **cache** and completing pending file system operations.|Incomplete writes, leading to corruption of the last-modified files.|
|**RAM Failure**|Faulty **RAM** can cause data corruption _before_ it is even written to the disk, leading to corrupted file system metadata or file contents being written by the OS.|Random or widespread file/metadata corruption.|