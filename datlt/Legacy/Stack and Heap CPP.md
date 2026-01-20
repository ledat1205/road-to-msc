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

**The call stack**
The stack has advantages and disadvantages:

- Allocating memory on the stack is comparatively fast.
- Memory allocated on the stack stays in scope as long as it is on the stack. It is destroyed when it is popped off the stack.
- All memory allocated on the stack is known at compile time. Consequently, this memory can be accessed directly through a variable.
- Because the stack is relatively small, it is generally not a good idea to do anything that eats up lots of stack space. This includes allocating or copying large arrays or other memory-intensive structures.
