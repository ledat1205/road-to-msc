
## 1. Finding "Port Already in Use" Errors

The most common use case is debugging a service (like Nginx or a Go app) that fails to start because its port is taken.

- **Command:** `lsof -i :8080`
    
- **What it tells you:** It shows the Command (process name) and the PID (Process ID) currently occupying port 8080.
    
- **Next Step:** You can then run `kill -9 <PID>` to clear the port.
    

## 2. Debugging "No Space Left on Device" (Ghost Files)

Sometimes `df` shows a disk is 100% full, but `du` doesn't show any large files. This usually happens when a process (like a logger) is still holding a deleted file open. The space isn't reclaimed until the process closes the file.

- **Command:** `lsof +L1`
    
- **What it tells you:** It lists all open files with a link count of less than 1.
    
- **Debug Insight:** Find the PID of the process holding the deleted file and restart that service to instantly free up disk space.
    

## 3. Identifying Memory Leaks or Resource Exhaustion

If an application is crashing with "Too many open files," it has hit its `ulimit`.

- **Command:** `lsof -p <PID>`
    
- **What it tells you:** Every single file, library (.so), and network connection that specific process has open.
    
- **Debug Insight:** If you see thousands of entries for the same log file or connection, your code likely has a leak where it's opening files/sockets but never calling `.close()`.
    

## 4. Tracking Down Configuration and Log Locations

If you've inherited a messy server and don't know where a specific process is writing its logs or reading its config:

- **Command:** `lsof -p <PID> | grep -E 'log|conf|yaml'`
    
- **What it tells you:** The exact absolute path of the files the process is actively interacting with.
![[Screenshot 2025-11-19 at 00.51.15.png]]

![[Screenshot 2025-11-19 at 00.51.57.png]]
![[Screenshot 2025-11-19 at 00.53.05.png]]
killall: kill 10 chrome windows (one gmail, one google search,...)