### 3.1 The most important idea: Most devices look like files

In Linux almost everything is a file — including devices.

You open, read, write, and close them almost the same way as normal files.

Examples everyone knows:

Bash

```
cat /dev/null          # reads nothing forever
echo "test" > /dev/null   # writes → nothing happens (thrown away)

dd if=/dev/zero of=zeros.bin bs=1M count=100   # creates 100 MB file full of zeros
```

### 3.2 Two main kinds of device files

|Type|First letter in ls -l|Can seek / jump around?|Examples|Typical use case|
|---|---|---|---|---|
|**Block**|**b**|Yes|/dev/sda, /dev/sda1, /dev/nvme0n1|Hard disks, SSDs, USB sticks|
|**Character**|**c**|No (stream only)|/dev/null, /dev/random, /dev/tty|Keyboard, mouse, serial port, printer|

Quick memory aid:

- **Block** → you can jump to any place (like chapters in a book)
- **Character** → only forward/backward stream (like live radio)

### 3.3 Where are device files? → /dev/

Bash

```
ls -l /dev/sd*     # hard disks & USB drives
ls -l /dev/sr*     # CD/DVD/Blu-ray drives
ls -l /dev/tty*    # terminals & virtual consoles
ls -l /dev/input/  # mice, keyboards, touchpads
```

### 3.4 Modern way: We don't create device files by hand anymore

Before ~2005–2008 people used to run:

Bash

```
mknod /dev/sda1 b 8 1
```

Today almost nobody does this.

Two modern systems do it automatically:

1. **devtmpfs** Kernel itself creates basic /dev entries very early during boot
2. **udev** (the userspace daemon udevd)
    - listens to kernel events ("a new USB stick appeared!")
    - reads device info
    - creates nice names + many symlinks
    - runs programs if needed (mount, notify desktop, etc.)

### 3.5 The most useful symlinks udev creates

You almost never use /dev/sda directly nowadays.

Instead you use stable names:

Bash

```
/dev/disk/by-id/     # contains serial number → very stable
/dev/disk/by-uuid/   # filesystem UUID → most common in /etc/fstab
/dev/disk/by-label/  # filesystem label
/dev/disk/by-partuuid/
/dev/disk/by-partlabel/
```

Example (very common today):

Bash

```
UUID=abcd-1234-...   /     ext4   defaults   0 1
```

### 3.6 Quick cheat-sheet — most common device names in 2024/2025

|Device type|Typical names today|Older names (still possible)|How to list them nicely|
|---|---|---|---|
|NVMe SSD|/dev/nvme0n1, /dev/nvme1n1|—|lsblk -o NAME,MODEL,SIZE,TYPE|
|SATA / USB hard disk|/dev/sda, /dev/sdb, …|/dev/hda, /dev/hdb (very old)|lsblk or lsscsi|
|USB flash drive|/dev/sdc, /dev/sdd, …|—|lsblk|
|Optical drive (CD/DVD)|/dev/sr0, /dev/sr1|/dev/cdrom (symlink)|ls /dev/sr*|
|Generic SCSI (burning)|/dev/sg0, /dev/sg1, …|—|lsscsi -g|
|Serial port (USB-serial)|/dev/ttyUSB0, /dev/ttyACM0|/dev/ttyS0 (real COM port)|`dmesg|

### 3.7 Very useful commands to understand your devices

|Command|What it shows|
|---|---|
|lsblk -f|disks + partitions + filesystems + mount points|
|lsblk -o +MODEL,SERIAL|even more info|
|lsscsi|SCSI view (good for USB disks too)|
|lsscsi -g|+ generic SCSI devices (/dev/sg*)|
|udevadm info -a -p $(udevadm info -q path -n /dev/sda)|deep hardware attributes|
|udevadm monitor|watch live "device added/removed" events|
|cat /proc/devices|major numbers → driver mapping|
|`dmesg|grep -i -E 'sd|

### Summary — the shortest possible version

1. Devices are files in **/dev**
2. Two types: **block** (disks) vs **character** (everything stream-like)
3. Modern systems use **devtmpfs + udev** → you don't create devices manually
4. Use stable names: **UUID**, **by-id**, **by-partuuid** (not /dev/sda!)
5. Useful commands: **lsblk**, **lsscsi**, **udevadm**, **dmesg**
![[Screenshot 2025-11-19 at 00.18.55.png]]
![[Screenshot 2025-11-19 at 00.19.58.png]]
![[Screenshot 2025-11-19 at 00.24.35.png]]

![[Screenshot 2025-11-19 at 00.23.06.png]]
![[Screenshot 2025-11-19 at 00.27.19.png]]
![[Screenshot 2025-11-19 at 00.27.36.png]]

Need to backup before formatting if fs has data.
![[Screenshot 2025-11-19 at 00.28.15.png]]
![[Screenshot 2025-11-19 at 00.28.42.png]]

![[Screenshot 2025-11-19 at 00.29.00.png]]
![[Screenshot 2025-11-19 at 00.29.45.png]]

## When File System Errors Happen?

File system errors typically occur when a process that is modifying the file system's metadata is **interrupted** before it can complete its full transaction and write the final "clean" state to the disk.

The most common scenarios that cause file system errors include:

| **Scenario**                     | **Description**                                                                                                                                                                                                                                 | **Outcome**                                                                               |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **Improper Shutdown/Power Loss** | The most common cause. Turning off power, forcefully shutting down, or a sudden power failure interrupts ongoing writes, leaving the file system in an **inconsistent** state (the data blocks might be written, but the index is not updated). | Inconsistent metadata, orphaned files, incorrect free block counts.                       |
| **Hardware Failure**             | **Bad sectors** on a hard drive or flash media. If critical metadata is written to a sector that suddenly fails, the file system structure becomes corrupted.                                                                                   | Metadata corruption (superblock, inode tables), permanent data loss in the affected area. |
| **Software/Driver Bugs**         | Errors within the OS kernel, device drivers, or sometimes even complex applications can lead to incorrect data being written to the file system's structure.                                                                                    | Corruption of specific file system structures, sometimes localized.                       |
| **Improper Device Ejection**     | Removing an external drive (USB, etc.) without safely ejecting it prevents the OS from flushing its write **cache** and completing pending file system operations.                                                                              | Incomplete writes, leading to corruption of the last-modified files.                      |
| **RAM Failure**                  | Faulty **RAM** can cause data corruption _before_ it is even written to the disk, leading to corrupted file system metadata or file contents being written by the OS.                                                                           | Random or widespread file/metadata corruption.                                            |
![[Screenshot 2025-11-19 at 00.32.07.png]]

![[Screenshot 2025-11-19 at 00.31.11.png]]![[Screenshot 2025-11-19 at 00.33.07.png]]
![[Screenshot 2025-11-19 at 00.33.23.png]]
![[Screenshot 2025-11-19 at 00.33.52.png]]
