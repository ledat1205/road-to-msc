### Explanation of the Diagram and Linux Architecture
![[Pasted image 20260131190156.png]]

The diagram titled "Linux Performance Observability Tools," created by Brendan Gregg in 2021, serves as a visual map to help understand how various monitoring tools interact with the layered architecture of a Linux system. It illustrates the flow from hardware at the bottom to user-space applications at the top, showing where specific tools can observe performance metrics, events, or issues. This layout highlights Linux's modular design, where each layer handles specific responsibilities, and tools "hook" into these layers for observability without significant overhead.

#### Key Layers of Linux Architecture Depicted in the Diagram:

- **Hardware Layer**: At the base, this includes physical components like CPUs (for processing), DRAM (memory), disks (storage), and network ports (for connectivity). Controllers (e.g., I/O Controller, Network Controller) manage data flow to/from these devices, connected via an I/O Bridge. This layer represents the raw physical resources that the operating system abstracts.
- **Device Drivers Layer**: These are kernel modules that interface directly with hardware. For example, block device drivers handle storage I/O, net device drivers manage network traffic, and CPU-related drivers deal with processing. Tools here monitor low-level hardware interactions, like perf for CPU performance or iostat for disk stats.
- **Kernel Subsystems Layer**: This is the core of the Linux kernel, handling resource management:
    - **File Systems and Volume Manager**: Manage data storage (e.g., ext4, btrfs, NFS, XFS, ZFS). VFS (Virtual File System) provides a unified interface for file operations.
    - **Block Device**: Handles block-level I/O for storage.
    - **Net Device, IP, TCP/UDP, Sockets**: Network stack for packet handling, connections, and communication.
    - **Scheduler**: Manages process execution and CPU allocation.
    - **Virtual Memory**: Handles memory allocation, paging, and swapping.
- **System Call Interface**: The boundary where user-space programs request kernel services (e.g., open a file, send data over network).
- **System Libraries**: Provide higher-level APIs for applications (e.g., libc for system calls).
- **Applications Layer**: User programs that run on top, interacting with the system via libraries and syscalls.

The diagram arrows point tools to the components they observe, emphasizing observability across the stack. For instance, tools like top and mpstat target CPUs, while tcpdump focuses on network protocols. It also shows the evolution of tools: traditional ones (e.g., sar from /proc) on the right, advanced eBPF-based ones (e.g., from BCC or bpftrace) on the left for dynamic tracing. This structure helps diagnose issues by starting at high-level symptoms (e.g., high load) and drilling down to specific layers (e.g., I/O bottlenecks in block devices).

Understanding this architecture reveals Linux's efficiency: it's monolithic (kernel handles everything) but modular, with user/kernel separation for security and stability. Performance problems often arise from contention in shared resources (e.g., CPU saturation, memory pressure), and the tools enable targeted analysis.

### Detailed Dive into Monitoring Commands

Below, I group the tools by subsystem (based on the diagram's mappings) for clarity. For each, I cover: purpose, example command(s), sample output, how to read/interpret it, and when to use it. These draw from standard Linux utilities, perf-tools, and BCC/eBPF collections. Many are basic (quick snapshots), intermediate (detailed stats), or advanced (tracing). Install missing ones via package managers (e.g., apt install sysstat for sar, iostat; BCC via apt install bpfcc-tools).

#### CPU and Scheduler Tools

These monitor processing, scheduling, and interrupts (e.g., top, mpstat, turbostat).

- **top**
    - **Purpose**: Interactive real-time view of processes, CPU, and memory usage.
    - **Example Command**: top (interactive; press 'q' to quit).
    - **Sample Output**:
        
        text
        
        ```
        top - 07:09:00 up 1 day, 12:34,  1 user,  load average: 0.50, 0.60, 0.70
        Tasks: 200 total,   2 running, 198 sleeping,   0 stopped,   0 zombie
        %Cpu(s): 10.0 us,  5.0 sy,  0.0 ni, 80.0 id,  5.0 wa,  0.0 hi,  0.0 si,  0.0 st
        MiB Mem :  8192.0 total,  2000.0 free,  4000.0 used,  2192.0 buff/cache
        MiB Swap:  2048.0 total,  2048.0 free,     0.0 used.  5000.0 avail Mem 
        
        PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
        1234 user     20   0 1234567  89012  34567 R  15.0   1.0   0:10.00 myapp
        ```
        
    - **How to Read/Interpret**: Load averages (over 1/5/15 min) > CPU count = overload. %Cpu(s): 'us' = user time (high = app CPU use); 'sy' = kernel time; 'id' = idle (low = busy); 'wa' = I/O wait (high = disk bottleneck). Process list: Sort by %CPU/%MEM to spot hogs; S = state (R=running, D=disk sleep).
    - **When to Use**: Quick check for high CPU processes during slowdowns; interactive troubleshooting.
- **mpstat**
    - **Purpose**: Per-CPU usage stats for multi-processor systems.
    - **Example Command**: mpstat -P ALL 1 3 (per-CPU, every 1s, 3 samples).
    - **Sample Output**:
        
        text
        
        ```
        Linux 5.15.0 (hostname) 	01/31/2026 	_x86_64_	(4 CPU)
        
        07:09:00 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
        07:09:01 PM  all    5.00    0.00    2.00    1.00    0.00    0.00    0.00    0.00    0.00   92.00
        07:09:01 PM    0    10.0    0.00    3.00    2.00    0.00    0.00    0.00    0.00    0.00   85.00
        ```
        
    - **How to Read/Interpret**: %usr high = user apps busy; %sys high = kernel overhead; %iowait high = waiting for I/O; low %idle = saturation. Per-CPU rows show imbalances (e.g., one CPU at 0% idle).
    - **When to Use**: Identify CPU hotspots or imbalances in multi-core systems.
- **turbostat**
    - **Purpose**: CPU frequency, power, and temperature stats (Intel-specific).
    - **Example Command**: turbostat sleep 5 (observe over 5s).
    - **Sample Output**:
        
        text
        
        ```
        turbostat version 20.03.20 - Len Brown <lenb@kernel.org>
        CPUID(7): No-SGX
        cpu0: MSR_RAPL_PKG_POWER_LIMITS: 0x2400000012000000 locked
        PKG_LIMIT1: 45.0000 Watts (0.000000 sec)
        ```
        
    - **How to Read/Interpret**: Shows C-states (idle modes), frequency (GHz), temp (°C). High temp/frequency = power issues; low C-states = busy CPU.
    - **When to Use**: Power/thermal analysis on servers; check turbo boost efficiency.

#### Memory Tools

Monitor allocation, swapping (e.g., vmstat, free, slabtop).

- **vmstat**
    - **Purpose**: System-wide memory, processes, I/O, and CPU stats.
    - **Example Command**: vmstat 1 3 (every 1s, 3 samples).
    - **Sample Output**:
        
        text
        
        ```
        procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
         r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
         1  0      0 2000000 100000 3000000   0    0     0     0  100  200  5  2 90  3  0
        ```
        
    - **How to Read/Interpret**: r > CPUs = runnable queue; swpd >0 = swap used (pressure); si/so >0 = swapping in/out (bad); bi/bo = blocks I/O (high = disk activity); us/sy/id/wa = CPU breakdown.
    - **When to Use**: Detect memory pressure or swapping during high load.
- **free**
    - **Purpose**: Quick memory and swap usage snapshot.
    - **Example Command**: free -h (human-readable).
    - **Sample Output**:
        
        text
        
        ```
        total        used        free      shared  buff/cache   available
        Mem:           8Gi       4Gi       2Gi       100Mi       2Gi       3Gi
        Swap:          2Gi         0B       2Gi
        ```
        
    - **How to Read/Interpret**: 'available' = usable memory (better than 'free'); buff/cache = reclaimable; swap used >0 = overflow (add more RAM).
    - **When to Use**: Initial check for low memory; combine with vmstat for trends.
- **slabtop**
    - **Purpose**: Kernel slab allocator stats (memory caches).
    - **Example Command**: slabtop (interactive).
    - **Sample Output**:
        
        text
        
        ```
        Active / Total Objects (% used)    : 123456 / 234567 (52.6%)
        Active / Total Slabs (% used)      : 1234 / 2345 (52.6%)
        ```
        
    - **How to Read/Interpret**: High active objects = kernel memory use; look for large caches (e.g., inode_cache) indicating leaks.
    - **When to Use**: Debug kernel memory issues, like in containers.

#### Disk/Storage Tools

For I/O, file systems (e.g., iostat, biotop, biosnoop).

- **iostat**
    - **Purpose**: Device I/O stats.
    - **Example Command**: iostat -x 1 3 (extended, every 1s, 3 samples).
    - **Sample Output**:
        
        text
        
        ```
        avg-cpu:  %user   %nice %system %iowait  %steal   %idle
                  5.00    0.00    2.00   10.00    0.00   83.00
        
        Device            r/s     w/s     rkB/s    wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz svctm  %util
        sda             10.00   20.00   100.00   200.00     0.00     0.00   0.00   0.00    5.00   10.00   0.50   10.00   10.00  3.33  100.00
        ```
        
    - **How to Read/Interpret**: r/s w/s = reads/writes per sec; %util near 100% = saturated; r_await/w_await high = latency; aqu-sz >1 = queueing.
    - **When to Use**: Spot disk bottlenecks during slow file ops.
- **biotop (from BCC)**
    - **Purpose**: Top-like for block I/O, showing processes causing disk activity.
    - **Example Command**: biotop (runs until Ctrl-C).
    - **Sample Output**:
        
        text
        
        ```
        PID    COMM             DISK NAME         I/O        Kbytes   AVGms
        1234   dd               sda  READ         100        1024     0.50
        ```
        
    - **How to Read/Interpret**: High I/O or Kbytes = busy processes; AVGms high = slow I/O.
    - **When to Use**: Trace which apps cause high disk load.
- **biosnoop (from BCC)**
    - **Purpose**: Traces individual block I/O events with latency.
    - **Example Command**: biosnoop (real-time).
    - **Sample Output**:
        
        text
        
        ```
        TIME(s)     COMM           PID    DISK    T SECTOR    BYTES   LAT(ms)
        0.000000    dd             1234   sda     R 123456    4096     0.50
        ```
        
    - **How to Read/Interpret**: LAT(ms) high = slow requests; T = type (R=read, W=write).
    - **When to Use**: Debug intermittent I/O latency spikes.

#### Network Tools

For packets, connections (e.g., ss, tcpdump, nicstat).

- **ss**
    - **Purpose**: Socket stats (faster than netstat).
    - **Example Command**: ss -tamp (TCP, all, memory, processes).
    - **Sample Output**:
        
        text
        
        ```
        State      Recv-Q Send-Q Local Address:Port Peer Address:Port Process
        ESTAB      0      0      127.0.0.1:80      127.0.0.1:12345 users:(("nginx",pid=1234,fd=3))
        ```
        
    - **How to Read/Interpret**: State = connection status; Recv-Q/Send-Q high = backlogs; Process = owner.
    - **When to Use**: Check active connections during network issues.
- **tcpdump**
    - **Purpose**: Capture and analyze network packets.
    - **Example Command**: tcpdump -i eth0 port 80 (HTTP on eth0).
    - **Sample Output**:
        
        text
        
        ```
        07:09:00.123456 IP host1.12345 > host2.http: Flags [S], seq 1234567890, win 12345, length 0
        ```
        
    - **How to Read/Interpret**: Flags [S] = SYN (handshake); length = data size; high packets = traffic volume.
    - **When to Use**: Debug protocol issues or packet loss.
- **nicstat**
    - **Purpose**: Network interface stats.
    - **Example Command**: nicstat 1 3.
    - **Sample Output**:
        
        text
        
        ```
        Time      Int   rKB/s   wKB/s   rPk/s   wPk/s    rAvs    wAvs     %Util    Sat
        07:09:00  eth0  100.00  200.00  1000.0  2000.0   100.0   100.0     50.00   0.00
        ```
        
    - **How to Read/Interpret**: High %Util = bandwidth saturation; Sat >0 = errors.
    - **When to Use**: Monitor throughput during high network load.

#### Tracing and General Tools

For syscalls, events (e.g., strace, perf, bpftrace, opensnoop).

- **strace**
    - **Purpose**: Trace system calls and signals for a process.
    - **Example Command**: strace -p 1234 (attach to PID).
    - **Sample Output**:
        
        text
        
        ```
        openat(AT_FDCWD, "/file.txt", O_RDONLY) = 3
        read(3, "data\n", 4096)                  = 5
        ```
        
    - **How to Read/Interpret**: Shows calls (e.g., openat) and returns (=3 = success); errors = -1 with errno.
    - **When to Use**: Debug app failures or slow syscalls.
- **perf**
    - **Purpose**: Profiling and tracing via perf_events.
    - **Example Command**: perf stat sleep 5 (stats over 5s).
    - **Sample Output**:
        
        text
        
        ```
        Performance counter stats for 'sleep 5':
        
                0.001 msec task-clock                #  0.000 CPUs utilized          
                     1      context-switches          #    1.000 K/sec                  
                     0      cpu-migrations            #    0.000 /sec                   
                   100      page-faults               #  100.000 K/sec                  
        
             5.000 seconds time elapsed
        ```
        
    - **How to Read/Interpret**: High context-switches = scheduling overhead; page-faults high = memory issues.
    - **When to Use**: CPU profiling, generating flame graphs for hotspots.
- **opensnoop (from BCC)**
    - **Purpose**: Traces file opens system-wide.
    - **Example Command**: opensnoop (real-time).
    - **Sample Output**:
        
        text
        
        ```
        PID    COMM               FD ERR PATH
        1234   bash                3   0 /etc/passwd
        ```
        
    - **How to Read/Interpret**: FD = descriptor; ERR >0 = failure; PATH = file.
    - **When to Use**: Track file access during config issues.
- **execsnoop (from BCC)**
    - **Purpose**: Traces new process executions.
    - **Example Command**: execsnoop.
    - **Sample Output**:
        
        text
        
        ```
        PCOMM            PID    PPID   RET ARGS
        bash             1234   5678     0 /bin/ls -l
        ```
        
    - **How to Read/Interpret**: RET =0 success; ARGS = command.
    - **When to Use**: Monitor script/process spawns in loops.
- **sar**
    - **Purpose**: Historical system activity reports (CPU, memory, I/O, network).
    - **Example Command**: sar -u 1 3 (CPU every 1s, 3 samples).
    - **Sample Output**:
        
        text
        
        ```
        07:09:00 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
        07:09:01 PM     all      5.00      0.00      2.00      1.00      0.00     92.00
        ```
        
    - **How to Read/Interpret**: Similar to mpstat; use -r for memory, -d for devices. Low %idle = busy; historical via logs.
    - **When to Use**: Trend analysis over days (enable in sysstat).

These tools provide a comprehensive toolkit. Start with basic ones (top, free) for overviews, then use intermediate (sar, pidstat) for details, and advanced (BCC) for tracing. For custom needs, bpftrace allows scripting. If a specific tool's output varies by system, test on your environment.