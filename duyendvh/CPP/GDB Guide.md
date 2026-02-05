Here is a practical, intensive **GDB guide** — from beginner essentials → intermediate workflow → advanced/power-user techniques (2025–2026 style).

### 0. Compile your program correctly (most important step!)

```bash
# Debug build (strongly recommended)
gcc -g -O0 -fno-omit-frame-pointer -Wall program.c -o program

# Or even better (modern recommendation 2024–2026)
gcc -g3 -ggdb -O0 -fno-omit-frame-pointer -fno-inline -Wall -Wextra program.c -o program

# For C++:
g++ -g3 -ggdb -O0 -fno-omit-frame-pointer -std=c++20 ...
```

`-g3` + `-ggdb` → best debug info  
`-O0` → no optimizations that destroy debuggability  
`-fno-omit-frame-pointer` → reliable backtraces

### 1. Starting GDB (most common ways – 2025)

```bash
# Classic way
gdb ./program

# With arguments
gdb --args ./program input.txt -v --mode=fast

# Attach to running process
gdb -p 12345
gdb --pid=12345

# Debug core dump
gdb ./program core
gdb ./program core.12345

# New TUI mode (strongly recommended in 2025+)
gdb -tui ./program
    # or inside gdb:  Ctrl+x Ctrl+a   (toggle TUI)
```

Inside gdb you almost always start with:

```gdb
(gdb) start                   # run + stop at main()
# or
(gdb) run                     # just run
(gdb) run arg1 arg2 < input.txt
```

### 2. Most Important Navigation Commands (must memorize)

| Command       | Short | What it does                              | When to use                              |
|---------------|-------|--------------------------------------------|------------------------------------------|
| run           | r     | (re)start program                         | Almost every session                     |
| start         |       | run and stop at beginning of main()       | Very common                              |
| continue      | c     | continue until next breakpoint/watchpoint | After you look around                    |
| next          | n     | step over (function calls)                | Normal line-by-line                      |
| step          | s     | step into                                 | Want to enter function                   |
| finish        | fin   | run until current function returns        | Got into function, want to get out fast  |
| until         | u     | run until given line/address              | Skip boring loop iterations              |
| advance N     | adv   | continue to line N (no breakpoints needed)| Jump forward without setting temp break  |

### 3. Breakpoints – The Heart of Debugging

```gdb
break main                      # break at start of main()
b main
b myfile.cpp:42                 # file + line
b function_name
b Class::method(int)
b *0x555555554321               # raw address (very useful in reverse / stripped binaries)

# Conditional breakpoints (extremely powerful)
break func if x > 100 && y < z
cond 1 ptr != NULL && ptr->magic == 0xdeadbeef   # add condition to existing breakpoint 1

# Temporary (delete after first hit)
tbreak main

# Hardware breakpoints (when code is in ROM / flash)
hbreak *0x08001234

# Delete / disable
delete 3               # delete breakpoint #3
clear main             # remove breakpoint at main
disable 2
enable 2
```

### 4. Examining State – The Most Used Commands

```gdb
print x                     # p x
p *ptr
p ptr->field
p/x ptr                     # hex
p/t value                   # binary
p/d value                   # decimal (default for integers)
p/c ch                      # character
p/s string_ptr              # null-terminated string

# Array / buffer
p *array@20                 # show first 20 elements
p *(int(*)[10])0x7fff1234   # interpret address as 10-element int array

# Watch an expression every stop
watch global_var
watch ptr->field > 100
watch *0x7fffffffe100       # watch memory location

# Memory (very important)
x/20xb  0x7fffffffdc00      # 20 bytes in hex
x/10i   $pc                 # 10 instructions at current PC
x/s     $rsp+8              # string at stack location

# Registers (x86_64)
info registers
p $rax
p $rdi
p *(void**)$rsp             # top of stack
p *(void**)$rsp@10          # top 10 stack pointers
```

### 5. Call Stack – Understand Where You Are

```gdb
bt              # backtrace (most important command after segfault)
backtrace full
frame 3         # select frame #3
up              # move one frame up
down
info frame
info args
info locals
```

### 6. TUI / Visual Mode (strongly recommended 2025+)

```gdb
# Start in TUI
gdb -tui ./program

# Or inside gdb
Ctrl + x   Ctrl + a     toggle TUI

# Useful TUI windows
layout src              # source only
layout asm              # assembly only
layout split            # src + asm
layout regs             # registers + src
tui reg general
tui reg float

# Very useful split
layout next             # cycle layouts
focus cmd               # focus command window
focus src
```

### 7. Advanced / Power Features (2025 power-user level)

```gdb
# Catch system calls
catch syscall write
catch syscall all

# Catch C++ exceptions
catch throw
catch rethrow
catch exception

# Multi-threaded
info threads
thread 2
break myfile.cpp:45 thread 3
thread apply all bt

# Reverse debugging (needs special build / rr or native gdb reverse)
reverse-step
reverse-next
reverse-continue
set exec-direction reverse

# Python scripting (very powerful)
python print(gdb.parse_and_eval("x").address)
# or write .gdbinit with python functions

# Pretty-printers for STL / boost / custom types (very common now)
# Many distros already include them
# Or install:  sudo apt install gdb python3-gdb

# Useful .gdbinit lines (put in ~/.gdbinit)
set history save on
set history filename ~/.gdb_history
set disassembly-flavor intel
set output-radix 0d10
set print pretty on
set print object on
set pagination off      # no --More-- prompts
```

### Quick Reference Card – Top 20 Commands You Use 95% of Time

1. `start` or `run`
2. `break` / `b`
3. `next` / `n`
4. `step` / `s`
5. `continue` / `c`
6. `print` / `p`
7. `bt` / `backtrace`
8. `up` / `down`
9. `info locals`
10. `info args`
11. `x/…`
12. `watch`
13. `finish`
14. `until`
15. `layout src` / `layout regs`
16. `delete` / `clear`
17. `cond`
18. `tui reg float`
19. `thread`
20. `quit`

Practice on a small buggy program (linked list corruption, use-after-free, off-by-one, null dereference, race condition) — that's the fastest way to become fluent.

Which kind of bug do you want to debug first? (segfault, wrong value, memory corruption, multithreading, …) I can give you a concrete example workflow for it.