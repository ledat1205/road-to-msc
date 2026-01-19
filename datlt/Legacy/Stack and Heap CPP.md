Memory divided into a few different segments:
- code segment: where the compiled program sits in the memory. This segment is typically read-only
- bss segment (the uninitialized data segment): where zero-initialized global and static variables are stored
- data segment (the initialized data segment): where initialized global and static variables are stored 
- the heap: dynamic allocated variables are allocated from
- the call stack: function parameters, local variables, and other function-related information are stored

**The heap segment**
The heap has advantages and disadvantages:

- Allocating memory on the heap is comparatively slow.
- Allocated memory stays allocated until it is specifically deallocated (beware memory leaks) or the application ends (at which point the OS should clean it up).
- Dynamically allocated memory must be accessed through a pointer. Dereferencing a pointer is slower than accessing a variable directly.
- Because the heap is a big pool of memory, large arrays, structures, or classes can be allocated here.

