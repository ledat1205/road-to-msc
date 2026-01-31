This cheat sheet is designed for SREs, DevOps engineers, and Backend developers who need to diagnose issues across the entire stack—from high-level API JSON responses down to kernel-level system calls.

---

## 1. The Network & API Layer (`curl`)

Use this for connectivity, latency, and protocol-level debugging.

|**Command / Flag**|**Purpose**|**Example / Use Case**|
|---|---|---|
|`curl -Iv`|**Header & TLS**|Check cert expiry, `Server` headers, and HTTP version.|
|`curl -L --max-redirs 5`|**Redirect Trace**|Stop infinite loops; see every `Location` hop.|
|`curl -w "@format.txt"`|**Latency Breakdown**|Use a file to define metrics like `time_namelookup`, `time_connect`.|
|`curl --resolve host:port:address`|**DNS Pinning**|Test a specific server IP without changing `/etc/hosts`.|
|`curl --trace-ascii -`|**Full Dump**|See every byte sent/received, including hidden control characters.|

**Pro-Tip:** Use `-D -` to dump headers to stdout along with the body.

---

## 2. The Data Processing Layer (`jq`, `awk`, `sed`)

Use this to transform "noise" into actionable "signals."

### `jq` (JSON Power User)

- **`.data[] | select(.id == "123")`**: Find a specific object in an array.
    
- **`-e 'if .error then error("API Error") else . end'`**: Use `jq` as a logic gate for scripts.
    
- **`path(..)`**: List every possible JSON path (great for learning massive nested schemas).
    
- **`@base64d`**: Decode Base64 strings found inside JSON fields directly in the pipe.
    

### `awk` (Columnar Genius)

- **`awk '{print $1}'`**: Extract the first column (IP Address in most logs).
    
- **`awk '/error/ {c++} END {print c}'`**: Count occurrences of "error" without `grep`.
    
- **`awk '$9 >= 500'`**: Filter logs where the 9th column (Status Code) is an error.
    

---

## 3. The System & Process Layer (`lsof`, `strace`, `ps`)

Use this when the application "hangs" or "disappears."

### `lsof` (The "Everything is a File" Tool)

- **`lsof -i :8080`**: Who is listening on this port?
    
- **`lsof -p <PID>`**: List all files, sockets, and libs loaded by a process.
    
- **`lsof +L1`**: Find deleted files still consuming disk space.
    

### `strace` (The Kernel Spy)

- **`strace -p <PID> -e trace=network`**: See every `sendto` and `recvfrom` call in real-time.
    
- **`strace -c -p <PID>`**: Summary of which system calls are taking the most time (Profile).
    
- **`strace -e open,connect`**: Only watch file opens and network connections.
    

---

## 4. The "War Room" Pipeline (Combined Power)

The ultimate "One-Liner" for monitoring a failing production endpoint.

Bash

```
# Set pipefail so errors don't hide
set -o pipefail; 

# 1. Repeatedly hit endpoint
watch -n 1 -d ' \
  curl -s -L -D >(grep -i "x-cache" >&2) \
  -w "HTTP=%{http_code} TIME=%{time_total} IP=%{remote_ip}\n" \
  "https://api.example.com/health" \
  | tee -a debug.log \
  | jq -e ".status == \"UP\"" \
  || echo "CRITICAL: Service Down" \
'
```

---

## 5. Summary Cheat Sheet Table