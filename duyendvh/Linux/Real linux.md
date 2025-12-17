:

- REST APIs (JSON)
    
- Tools: `curl`, `jq`, `grep`, `awk`, `sed`, `xargs`, `watch`
    

---

## 1. Inspect headers + body + timing (baseline debug)

`curl -s -D >(grep -i -E 'http|content|x-' >&2) \      -w "\nTIME_TOTAL=%{time_total}\nHTTP_CODE=%{http_code}\n" \      https://api.example.com/users \ | jq .`

### What this debugs

- HTTP status
    
- Response headers
    
- JSON formatting
    
- Latency
    

### Internals

- `curl` writes headers to a **process substitution**
    
- Headers piped to `grep`
    
- Body piped to `jq`
    
- Shell sets up **multiple pipes + fork/exec**
    

---

## 2. Find failing responses only (status ≠ 200)

`curl -s -o /tmp/body.json -w "%{http_code}" https://api.example.com/users \ | grep -v '^200$' && cat /tmp/body.json`

### Use case

- Only dump body when API fails
    

### Logic

- `grep -v` checks exit code
    
- `&&` triggers body print **only on failure**
    

---

## 3. Extract error fields deeply nested in JSON

`curl -s https://api.example.com/users \ | jq -r '.errors[]? | "\(.code): \(.message)"'`

### Debugs

- Structured API errors
    
- Missing fields (`?` avoids crashes)
    

---

## 4. Compare two API responses (prod vs staging)

`diff \  <(curl -s https://staging.api.example.com/users | jq -S .) \  <(curl -s https://prod.api.example.com/users    | jq -S .)`

### Debugs

- Schema or data drift
    

### Internals

- Shell creates **temporary FIFOs**
    
- Two `curl` processes run in parallel
    
- `diff` reads both streams
    

---

## 5. Find slow endpoints from logs

`grep '/api/' access.log \ | awk '$NF > 1000 { print $7, $NF }' \ | sort -k2 -nr \ | head`

### Debugs

- Slow API calls
    

### Explanation

- `$NF` = response time
    
- Sort numerically, reverse
    

---

## 6. Validate API schema quickly

`curl -s https://api.example.com/users \ | jq -e '.data[].id and .data[].email' \ || echo "Schema validation failed"`

### Debugs

- Missing required fields
    

### Internals

- `jq -e` exits non-zero on failure
    
- `||` triggers error message
    

---

## 7. Reproduce one failing request from logs

`grep '500' api.log \ | awk '{print $NF}' \ | head -1 \ | xargs -I{} curl -v {}`

### Debugs

- Exact request causing server error
    

---

## 8. Monitor API behavior in real time

`watch -n 2 \ 'curl -s -w " HTTP=%{http_code} TIME=%{time_total}\n" \  https://api.example.com/health'`

### Debugs

- Flaky endpoints
    
- Latency spikes
    

---

## 9. Trace redirect loops

`curl -Ls -o /dev/null -w "%{url_effective}\n" https://api.example.com`

### Debugs

- Infinite redirects
    
- Wrong Location headers
    

---

## 10. Detect partial / malformed JSON

`curl -s https://api.example.com/users \ | jq . \ || echo "Invalid JSON from API"`

### Debugs

- Truncated responses
    
- Encoding issues
    

---

## 11. Find API fields changing over time

`while true; do   curl -s https://api.example.com/users | jq -S .   sleep 5 done | uniq -c`

### Debugs

- Non-deterministic fields
    
- Caching issues
    

---

## 12. End-to-end failure trap (production-safe)

`set -o pipefail  curl -s https://api.example.com/users \ | jq '.data[] | select(.status!="ACTIVE")' \ | tee bad_users.json \ | jq -e '.' \ || alert "Inactive users detected"`

### Why this matters

- `pipefail` ensures **any failure breaks the chain**
    
- `tee` captures evidence
    

---

## Mental model for API debugging

- `curl` → transport layer
    
- `jq` → schema & content validation
    
- `grep/awk/sed` → pattern & metrics
    
- `&& ||` → control flow
    
- `tee` → evidence capture
    

---

If you want next:

- **HTTP/2 debugging**
    
- **TLS / cert issues**
    
- **API auth (OAuth / JWT) inspection**
    
- **strace curl to see syscalls**
    

Just say which one 👌

note each of the keyword

Below is a **concise “notes” / glossary** of the **keywords, commands, and operators** that appeared in the API-debugging Linux command examples. Think of this as a **cheat sheet with internal meaning**.

---

## Core commands (what they do)

### `curl`

- Makes HTTP requests (API calls)
    
- Talks directly to the network stack
    
- Exit code ≠ HTTP status (important!)
    

Common flags:

- `-s` → silent (no progress bar)
    
- `-v` → verbose (headers + TLS info)
    
- `-L` → follow redirects
    
- `-D` → dump headers
    
- `-o` → write output to file
    
- `-w` → print metrics (time, status)
    

---

### `jq`

- JSON parser / validator
    
- Used for **API response inspection**
    

Common flags:

- `.` → pretty-print JSON
    
- `-r` → raw output (no quotes)
    
- `-S` → sort keys (stable diff)
    
- `-e` → exit non-zero if filter fails
    
- `?` → optional field (avoid crash)
    

---

### `grep`

- Line-based pattern filter
    

Common flags:

- `-i` → ignore case
    
- `-v` → invert match
    
- `-E` → extended regex
    
- `-l` → print filenames only
    

---

### `awk`

- Field-based text processor
    
- Works well with logs
    

Keywords:

- `$NF` → last field
    
- `{print $7}` → print column 7
    
- Condition before `{}` filters lines
    

---

### `sed`

- Stream editor
    
- Used to modify text inline
    

Example uses:

- Replace fields
    
- Remove headers
    
- Normalize output
    

---

### `diff`

- Compares two inputs line by line
    
- Used to compare API responses
    

---

### `sort`

- Sorts lines
    
- `-n` → numeric
    
- `-r` → reverse
    

---

### `uniq`

- Collapses duplicate lines
    
- `-c` → count occurrences
    

---

### `head`

- Shows first N lines
    
- Used to limit output
    

---

### `tee`

- Splits output:
    
    - to terminal
        
    - and to a file
        
- Keeps pipeline flowing
    

---

### `watch`

- Runs a command repeatedly
    
- Useful for real-time API monitoring
    

---

### `xargs`

- Turns stdin into command arguments
    
- Often used to replay failing API calls
    

---

## Shell operators (control flow)

### `|`

**Pipe**

- Sends stdout of left → stdin of right
    
- Implemented with `pipe()` + `dup2()`
    

---

### `;`

**Sequential execution**

- Always run next command
    
- Exit code ignored
    

`cmd1 ; cmd2`

---

### `&&`

**Conditional success**

- Run next only if exit code == 0
    

`cmd1 && cmd2`

---

### `||`

**Conditional failure**

- Run next only if exit code ≠ 0
    

`cmd1 || cmd2`

---

### `&`

**Background execution**

- Shell does not wait
    
- Job control kicks in
    

---

### `<( … )`

**Process substitution**

- Treats command output as a file
    
- Uses FIFO / `/proc`
    

`diff <(cmd1) <(cmd2)`

---

## Redirections (file descriptors)

### `>`

- Redirect stdout to file (overwrite)
    

### `>>`

- Append stdout to file
    

### `<`

- Redirect stdin from file
    

### `2>`

- Redirect stderr
    

### `&>`

- Redirect stdout + stderr
    

---

## Environment & shell behavior

### `export`

- Makes variable available to child processes
    

---

### `set -o pipefail`

- Pipeline fails if **any command fails**
    
- Critical for reliable API debugging
    

---

## Exit codes (critical concept)

- `0` → success
    
- `!= 0` → failure
    
- Shell uses these for:
    
    - `&&`
        
    - `||`
        
    - `if`
        
    - pipelines
        

---

## Internal system calls (what Linux uses)

|Concept|System call|
|---|---|
|Process creation|`fork()`|
|Run program|`execve()`|
|Pipe|`pipe()`|
|Redirect FD|`dup2()`|
|Wait|`waitpid()`|
|Open file|`open()`|

---

## Mental summary (one-liners)

- `curl` → **talk to API**
    
- `jq` → **understand JSON**
    
- `grep/awk` → **filter & extract**
    
- `|` → **connect processes**
    
- `&& ||` → **control flow**
    
- `tee` → **capture evidence**
    
- `pipefail` → **trust your pipeline**
    

---

If you want, I can:

- turn this into a **one-page printable cheat sheet**
    
- annotate **one real API debug command line-by-line**
    
- map each keyword to **exact syscalls**
    

how to show how the request called by curl will be redirected to what endpoint (if any)

To see **whether a `curl` request is redirected and exactly to which endpoint(s)**, there are a few levels of detail you can choose from—simple to very detailed.

---

## 1. Show redirect chain (most common)

`curl -L -v https://api.example.com`

### What you’ll see

- Each request/response pair
    
- `> GET …` → request sent
    
- `< HTTP/1.1 301/302/307/308`
    
- `< Location: https://new-endpoint.example.com`
    

This shows the **full redirect chain**.

---

## 2. Show only redirect targets (clean & focused)

`curl -Ls -o /dev/null -w "%{url_effective}\n" https://api.example.com`

### Meaning

- `-L` → follow redirects
    
- `-s` → silent
    
- `-o /dev/null` → discard body
    
- `%{url_effective}` → final endpoint reached
    

Useful when you only care about **where it ends up**.

---

## 3. Show each redirect hop (advanced but clean)

`curl -s -o /dev/null -D - https://api.example.com \ | grep -i '^location:'`

### What this does

- Does **not** follow redirects
    
- Prints every `Location` header
    
- Shows intermediate endpoints
    

---

## 4. Show redirect chain + HTTP codes (recommended)

`curl -Ls -o /dev/null \      -w "%{http_code} -> %{url_effective}\n" \      https://api.example.com`

Example output:

`200 -> https://final.api.example.com/v1/users`

---

## 5. Trace redirects without following them

`curl -I https://api.example.com`

If response is:

`HTTP/1.1 302 Found Location: https://auth.example.com/login`

You immediately know the next endpoint.

---

## 6. Show redirects with timing (debug performance)

`curl -L -w \ "DNS=%{time_namelookup} CONNECT=%{time_connect} REDIRECT=%{time_redirect} TOTAL=%{time_total}\n" \ -o /dev/null -s https://api.example.com`

Shows:

- Time spent in redirects
    
- Whether redirects are slowing the API
    

---

## 7. Detect redirect loops

`curl -L --max-redirs 5 -v https://api.example.com`

If exceeded:

`curl: (47) Maximum (5) redirects followed`

---

## 8. JSON API redirect detection (script-friendly)

`curl -s -o /dev/null -w \ "REDIRECTS=%{num_redirects} FINAL=%{url_effective}\n" \ https://api.example.com`

---

## Internals (what actually happens)

- Server responds with:
    
    - `3xx` status code
        
    - `Location` header
        
- `curl -L`:
    
    - Parses `Location`
        
    - Issues **new request**
        
    - Repeats until non-3xx
        
- Each redirect is a **new TCP/HTTP request**
    
- Credentials & headers may or may not be forwarded (security!)
    

---

## Quick decision table

|Goal|Command|
|---|---|
|See all hops|`curl -L -v URL`|
|Final endpoint only|`curl -Ls -w "%{url_effective}\n"`|
|Headers only|`curl -I URL`|
|Redirect loop debug|`curl -L --max-redirs`|

---

If you want next:

- how **Authorization headers behave across redirects**
    
- debugging **OAuth / login redirects**
    
- seeing redirects with **HTTP/2 or TLS details**
    
- tracing redirects using `strace`
    

curl -L -v 'https://api.tiki.vn/raiden/v3/widgets/top_choise' * URL rejected: Malformed input to a URL function * Closing connection curl: (3) URL rejected: Malformed input to a URL function

Ah! The error message you’re seeing:

`curl: (3) URL rejected: Malformed input to a URL function`

means that **`curl` thinks your URL is invalid**. Let’s break down why and how to fix it.

---

## 1. Common causes

### a) Quoting / shell characters

- Single quotes `'...'` are usually fine.
    
- But sometimes **non-ASCII or invisible characters** (like smart quotes or zero-width spaces) sneak in.
    
- Your URL has:
    

`https://api.tiki.vn/raiden/v3/widgets/top_choise`

Check for:

- Typo in **`top_choice`**?
    
- Extra spaces or hidden characters.
    

**Tip:** Re-type the URL manually instead of copy-paste.

---

### b) URL encoding issues

- URLs **cannot contain spaces, accented characters, or some symbols**.
    
- If any exists, they must be **percent-encoded**.
    

Example:

`curl 'https://example.com/path%20with%20space'`

---

### c) Shell interpretation

- `&` → background job
    
- `?` → globbing
    
- `;` → command separator
    

Always quote URLs or escape special characters:

`curl -L "https://api.tiki.vn/raiden/v3/widgets/top_choice"`

---

## 2. Check the URL

1. Verify **exact spelling**: `top_choise` vs `top_choice`
    
2. Use `echo` to see hidden characters:
    

`echo 'https://api.tiki.vn/raiden/v3/widgets/top_choise' | cat -A`

- `$` → line end
    
- `^M` → Windows CR
    
- `M-b` → invisible character
    

---

## 3. Minimal working command

`curl -v -L "https://api.tiki.vn/raiden/v3/widgets/top_choice"`

- Replace single quotes with double quotes
    
- Make sure the path is **exact**
    
- If the endpoint needs authentication, add headers:
    

`curl -v -L \   -H "Authorization: Bearer <TOKEN>" \   "https://api.tiki.vn/raiden/v3/widgets/top_choice"`

---

## 4. Troubleshooting tips

- Run `curl` without `-L` first, to see raw response:
    

`curl -v "https://api.tiki.vn/raiden/v3/widgets/top_choice"`

- Check for **HTTP redirects** (`3xx`) or **errors** (`4xx/5xx`)
    
- Use `--trace-ascii trace.txt` to debug fully:
    

`curl --trace-ascii trace.txt "https://api.tiki.vn/raiden/v3/widgets/top_choice"`

---

✅ **Summary**

- The `Malformed input` error = URL is **not valid ASCII URL** or contains **illegal characters**.
    
- Solutions:
    
    1. Verify spelling
        
    2. Quote the URL
        
    3. Percent-encode special characters
        
    4. Remove invisible/hidden characters
        

---

If you want, I can write a **ready-to-use `curl` command that traces this Tiki API redirect chain and shows the final endpoint**. This will help debug it fully. Do you want me to do that?

  

ChatGPT can make mistakes. Check important info. See Cookie Preferences.

# We use cookies

We use cookies to help this site function, understand service usage, and support marketing efforts. Visit Manage Cookies to change preferences anytime. View our [Cookie Policy](https://openai.com/policies/cookie-policy/) for more info.

Manage Cookies

Reject non-essential

Accept all