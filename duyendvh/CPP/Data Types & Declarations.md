

These keywords define the basic types of data a variable can hold and how its declaration is treated.

| **Keyword**                                               | **Purpose**                                                                                                                        | **Example / Notes**                                                                                                                                                                                          |
| --------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **bool**                                                  | Boolean type (holds `true` or `false`).                                                                                            | `bool is_valid = true;`                                                                                                                                                                                      |
| **char**, **char8_t** (C++20), **char16_t**, **char32_t** | Types for storing single characters or small integers. `charX_t` are fixed-size Unicode types.                                     | `'a'`, `u8'x'`, `u'y'`, `U'z'`<br>U+0041 'A'      → 41          (1 byte)<br>U+00E9 'é'      → C3 A9       (2 bytes)<br>U+4F60 '你'     → E4 BD A0    (3 bytes)<br>U+1F600 😀      → F0 9F 98 80 (4 bytes)<br> |
| **int**, **long**, **short**                              | Primary types for storing integers of various guaranteed minimum sizes.                                                            | `long counter = 10L;`                                                                                                                                                                                        |
| **float**, **double**                                     | Types for storing single- and double-precision floating-point numbers.                                                             | `double pi = 3.14159;`<br>`float pi = 3.14f`                                                                                                                                                                 |
| **signed**, **unsigned**                                  | Specifies if an integer type should be signed (can hold negative values) or unsigned (only non-negative values).                   | `unsigned int u;`                                                                                                                                                                                            |
| **void**                                                  | Indicates an absence of type (e.g., function return value) or a generic pointer.                                                   | `void* ptr;`                                                                                                                                                                                                 |
| **wchar_t**                                               | Wide character type (size is platform-dependent).                                                                                  | `wchar_t wc = L'A';`                                                                                                                                                                                         |
| **auto**                                                  | Directs the compiler to deduce the variable's type from its initializer.<br>auto x = 1 / 2;   // ❌ 0<br>auto y = 1.0 / 2; // ✅ 0.5 | `auto num = 5;` (deduced as `int`)                                                                                                                                                                           |
| **typedef**                                               | Creates an alias (new name) for an existing type.                                                                                  | `typedef int score;`                                                                                                                                                                                         |
| **using**                                                 | Used for creating type aliases (modern alternative to `typedef`) and for namespace declarations.                                   | `using T = int;`                                                                                                                                                                                             |

---

## Storage & Type Qualifiers

These modify how and where a variable's memory is managed and accessed.

| **Keyword**           | **Purpose**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | **Example / Notes**                                                                                                                                                                                                                         |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **const**             | Declares a variable as **read-only** (cannot be modified after initialization).                                                                                                                                                                                                                                                                                                                                                                                                                                                              | `const int size = 5;`<br>void print(const std::vector<int>& v) {<br>    // v.push_back(1); ❌ error<br>    std::cout << v.size(); // ✅ allowed<br>}<br>int x = 10;<br>const int& r = x;<br>x = 20;        // allowed<br>// r = 30;     ❌<br> |
| **constexpr**         | Specifies that a value or function can be evaluated at **compile time**.                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | `constexpr int sq(int x) { return x*x; }`                                                                                                                                                                                                   |
| **consteval** (C++20) | Specifies a function must be evaluated at **compile time** (immediately).                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Enforces compile-time execution.<br>`constexpr` = _can_  <br>`consteval` = _must_                                                                                                                                                           |
| **constinit** (C++20) | Guarantees a variable has **static initialization** (at compile time).                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Prevents issues with initialization order.                                                                                                                                                                                                  |
| **volatile**          | ### What `volatile` DOES<br><br>- Prevents certain compiler optimizations<br>    <br>- Forces memory reads/writes<br>    <br><br>### What it DOES NOT do ❌<br><br>- ❌ Does NOT make code thread-safe<br>    <br>- ❌ Does NOT replace atomics or mutexes<br>    <br><br>### Modern rule<br><br>> Use `std::atomic`, not `volatile`, for multithreading                                                                                                                                                                                        | Used in multi-threaded or embedded programming.<br>volatile int* status = reinterpret_cast<int*>(0x40000000);<br><br>while ((*status & 1) == 0) {<br>    // compiler must reload value every iteration<br>}<br>                             |
| **extern**            | Declares a variable or function is defined in another translation unit (file).<br>`extern` = “trust me, it exists somewhere else”                                                                                                                                                                                                                                                                                                                                                                                                            | Used for specifying **external linkage**.<br>// header.h<br>extern int globalCounter;<br><br>// source.cpp<br>int globalCounter = 0;<br>                                                                                                    |
| **static**            | ### 1️⃣ Static inside function (persistent state)<br><br>`int counter() {     static int count = 0;     return ++count; }`<br><br>- Initialized once<br>    <br>- Lives for program lifetime<br>    <br>### 2️⃣ Static at file scope (internal linkage)<br><br>`static int helper() { return 42; }`<br><br>- Visible **only in this file**<br>    <br>- Prevents symbol collision<br>    <br><br>---<br><br>### 3️⃣ Static class member<br><br>`struct A {     static int value; };  int A::value = 10;`<br><br>Shared across all instances. | Used for specifying **internal linkage**.                                                                                                                                                                                                   |
| **thread_local**      | Specifies that a variable will be distinct for each thread.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Used for thread-safe global variables.                                                                                                                                                                                                      |
| **mutable**           | Allows a `const` object to modify a specific member variable.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Used in `const` member functions of a class.                                                                                                                                                                                                |
| **register**          | **Deprecated in C++17.** Hint to the compiler to store the variable in a CPU register (usually ignored).                                                                                                                                                                                                                                                                                                                                                                                                                                     |                                                                                                                                                                                                                                             |

---

## Control Flow & Program Structure

Keywords used to manage the logical flow and organization of a program.

|**Keyword**|**Purpose**|**Syntax/Context**|
|---|---|---|
|**if**, **else**|Conditional execution.|`if (cond) { ... } else { ... }`|
|**for**, **while**, **do**|Loop constructs.|`for(;;)`, `while(cond)`, `do { ... } while(cond)`|
|**break**|Exits the innermost loop or `switch` statement immediately.|Inside a loop/switch.|
|**continue**|Skips the remainder of the innermost loop's body and proceeds to the next iteration.|Inside a loop.|
|**switch**, **case**, **default**|Multi-way branching based on a variable's value.|`switch(var) { case 1: ...; default: ... }`|
|**return**|Exits a function and returns a value (if the function is not `void`).|`return result;`|
|**goto**|Unconditional jump to a labeled statement (highly discouraged).|`goto label_name;`|
|**asm**|Embeds assembly language instructions directly into the C++ code.|Platform-dependent.|

---

## Object-Oriented & User-Defined Types

These define custom data structures, their members, and access control.

| **Keyword**                            | **Purpose**                                                                                                                                      | **Context**                                 |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------- |
| **class**, **struct**                  | Defines a custom user-defined type (aggregate data structure). **`class`** defaults to **private** members; **`struct`** defaults to **public**. | `class MyClass { ... };`                    |
| **union**                              | Defines a type where all members share the same memory location.                                                                                 | Only one member can be used at a time.      |
| **enum**                               | Declares an enumeration type (a set of named constant integer values).                                                                           | `enum Color { RED, BLUE };`                 |
| **public**, **protected**, **private** | **Access specifiers** that control the visibility of class/struct members.                                                                       | Used inside `class` or `struct` definition. |
| **virtual**                            | Used for functions in a base class to allow derived classes to override the implementation (**Polymorphism**).                                   | `virtual void func();`                      |
| **friend**                             | Grants a non-member function or another class access to a class's private and protected members.                                                 | Used inside the class definition.           |
| **this**                               | A pointer to the object on which a member function is currently executing.                                                                       | Used inside member functions.               |
| **explicit**                           | Prevents a constructor from being used for implicit conversions.                                                                                 | Used before a single-argument constructor.  |
| **namespace**                          | Provides a scope to organize code and prevent name collisions.                                                                                   | `namespace MyLib { ... }`                   |
| **operator**                           | Used to **overload** an operator (e.g., `+`, `<<`) for a user-defined type.                                                                      | `MyClass operator+(const MyClass& other);`  |

---

## Casting, Type ID, & Memory

Keywords used for converting between types, runtime type information, and memory operations.

| **Keyword**          | **Purpose**                                                                                                                    | **Context / Notes**                                       |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------- |
| **static_cast**      | Converts between related types (e.g., base $\leftrightarrow$ derived class, `int` $\leftrightarrow$ `float`). Safest C++ cast. | `static_cast<int>(f);`                                    |
| **const_cast**       | Used only to add or remove `const` or `volatile` qualifiers from a type.                                                       | **Use with caution.**                                     |
| **reinterpret_cast** | Converts between unrelated types (e.g., a pointer to an integer). Most dangerous C++ cast.                                     | Used for low-level, bitwise conversions.                  |
| **dynamic_cast**     | Performs checked conversion of polymorphic types during **run time**.                                                          | Used primarily for safe downcasting in class hierarchies. |
| **typeid**           | Returns an object of type `std::type_info` that describes the object's type at **run time**.                                   | Used in conjunction with `dynamic_cast`.                  |
| **new**              | Allocates memory dynamically (on the heap) and returns a pointer to the allocated object(s).                                   | `int* p = new int[10];`                                   |
| **delete**           | Deallocates dynamically allocated memory.                                                                                      | `delete p;`                                               |
| **nullptr**          | A literal value that represents a null pointer (safer than using the integer `0` or `NULL`).                                   | `int* p = nullptr;`                                       |
| **sizeof**           | Returns the size in bytes of a variable or a type.                                                                             | `sizeof(int)` or `sizeof(my_var)`                         |
| **alignas**          | Specifies the desired memory alignment for a variable or user-defined type.                                                    | `alignas(16) int x;`                                      |
| **alignof**          | Queries the alignment requirement of a type.                                                                                   | `alignof(int)`                                            |
| **decltype**         | Determines the type of an expression at **compile time**.                                                                      | `decltype(x + y) z;`                                      |

---

## Template & Concepts (C++20)

Keywords central to generic programming.

|**Keyword**|**Purpose**|**Context**|
|---|---|---|
|**template**|Declares a parameterized type (a function or class template).|`template <typename T>`|
|**typename**|Used within templates to specify that a dependent name is a type.|Used in template parameter lists and definitions.|
|**concept** (C++20)|Defines a set of constraints on template type parameters.|`template <MyConcept T>`|
|**requires** (C++20)|Used to specify constraints on template parameters using concepts or other compile-time logic.|`requires (T t) { ... }`|

---

## Exceptions & Coroutines (C++20)

Keywords for error handling and asynchronous programming.

|**Keyword**|**Purpose**|**Context**|
|---|---|---|
|**try**|Encloses a block of code where exceptions might be thrown.|`try { ... }`|
|**catch**|Defines the code block that handles exceptions of a specific type.|`catch (Exception e) { ... }`|
|**throw**|Explicitly raises an exception.|`throw MyError();`|
|**noexcept**|Specifies that a function is guaranteed not to throw an exception.|`void func() noexcept;`|
|**static_assert**|Performs a compile-time assertion (checks a condition at compile time).|`static_assert(sizeof(int) >= 4, "...");`|
|**co_await** (C++20)|Suspends execution until the result of an asynchronous operation is available.|Used in coroutine functions.|
|**co_return** (C++20)|Completes a coroutine's execution, optionally returning a result.|Used in coroutine functions.|
|**co_yield** (C++20)|Suspends execution and returns a value to the caller (used for generators/streams).|Used in coroutine functions.|

---

## Alternative Operator Tokens

These keywords are text alternatives for C++ operators, often used to improve code readability in environments where certain symbols are not easily accessible (though rarely used in modern code).

|**Keyword**|**Operator Equivalent**|**Type**|
|---|---|---|
|**and**|`&&`|Logical AND|
|**or**|`||
|**not**|`!`|Logical NOT|
|**bitand**|`&`|Bitwise AND|
|**bitor**|`|`|
|**compl**|`~`|Bitwise NOT|
|**xor**|`^`|Bitwise XOR|
|**and_eq**|`&=`|Bitwise AND assignment|
|**or_eq**|`|=`|
|**not_eq**|`!=`|Relational NOT equal|
|**xor_eq**|`^=`|Bitwise XOR assignment|

## Declaration vs. Definition (The Core Distinction)

This is a fundamental concept in C++. A **Definition** is always a **Declaration**, but a **Declaration** is not always a **Definition**.

| **Term**             | **Technical Meaning**                                                         | **Contains**                | **Examples**                                                                                              |
| -------------------- | ----------------------------------------------------------------------------- | --------------------------- | --------------------------------------------------------------------------------------------------------- |
| **Declaration**      | Tells the compiler the **name** and **type** of an identifier.                | **Only Type Info.**         | `int x;` (Variable)<br><br>  <br><br>`int add(int x, int y);` (Function Prototype)                        |
| **Definition**       | Implements the function or allocates storage (instantiates) for the variable. | **Implementation/Storage.** | `int x;` (Variable, allocates storage)<br><br>  <br><br>`int add() { ... }` (Function, provides the body) |
| **Pure Declaration** | A declaration that is **not** a definition (e.g., a function prototype).      | **No storage/body.**        | `extern int x;`<br><br>  <br><br>`void func();`                                                           |

> 🔑 **Key Insight:**
> 
> - The **compiler** only needs a **Declaration** (prototype) to validate syntax.
>     
> - The **linker** needs the single, corresponding **Definition** (function body/variable storage) to build the final executable.


## Accessing Namespaced Identifiers

| **Method**                 | **Syntax**              | **Description**                                                                                  | **Best Practice**                                                                                                                                                    |
| -------------------------- | ----------------------- | ------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Explicit Qualification** | `namespace::identifier` | Uses the **Scope Resolution Operator (`::`)** to explicitly tell the compiler which name to use. | **Preferred.** Safe, clear, and avoids polluting the current scope.                                                                                                  |
| **Using Declaration**      | `using std::cout;`      | Introduces a single identifier from a namespace into the current scope.                          | Okay for a few names, but use sparingly.                                                                                                                             |
| **Using Directive**        | `using namespace std;`  | Introduces **all** names from a namespace into the current scope.                                | **Discouraged.** Often leads right back to naming collisions if multiple libraries or user code use the same identifier. **Avoid in header files and global scope.** |

## Example 1: Compiler Error (Same File Scope)

This happens when you try to define the same identifier twice in the same scope, or when two identifiers from different namespaces are pulled into the same scope, creating **ambiguity**.

|**Scenario**|**Code**|**Result**|
|---|---|---|
|**Direct Redefinition (ODR Violation)**|`cpp int count = 10; int count = 5; // Error`|**Compiler Error:** `redefinition of 'count'` (Violates ODR Part 1). The compiler stops immediately.|
|**Ambiguity via `using` Directive**|`cpp namespace A { void log(); } namespace B { void log(); } using namespace A; using namespace B; void test() { log(); // Error }`|**Compiler Error:** `call to 'log' is ambiguous`. The compiler cannot decide which `log` function to call, as both are now visible in the current scope.|

---

## Example 2: Linker Error (Multiple Files)

This happens when two different source files (translation units) define the same non-local global identifier. The compiler sees one definition per file and is happy, but the **linker** sees two definitions for the same symbol when combining the files.

|**File**|**Code**|
|---|---|
|**`helper.cpp`**|`cpp // Definition 1: Global function void initialize_settings() { /* ... */ }`|
|**`main.cpp`**|`cpp // Definition 2: Global function void initialize_settings() { /* ... */ } int main() { // Linker tries to resolve this name: initialize_settings(); }`|

> ❌ **Result:** **Linker Error:** `multiple definition of 'initialize_settings'` (Violates ODR Part 2). The program compiles successfully but fails at the linking stage because the linker cannot include two separate function bodies for the same global name in the final executable.

### How Namespaces Fix the Linker Error

If you wrap the functions in namespaces, the collision is resolved because the names are no longer identical in the global scope:

|**File**|**Code (Fixed)**|**Linker Sees**|
|---|---|---|
|**`helper.cpp`**|`cpp namespace Helper { void initialize_settings() { /* ... */ } }`|`Helper::initialize_settings`|
|**`main.cpp`**|`cpp namespace Main { void initialize_settings() { /* ... */ } } int main() { Helper::initialize_settings(); Main::initialize_settings(); }`|

# C++ Data Structures Cheat Sheet

This cheat sheet covers the most commonly used methods for key data structures in the C++ Standard Library, including both pre-C++11 ("old") and modern features (C++11 and later, up to C++23 where applicable). I've focused on containers from `<vector>`, `<list>`, `<deque>`, `<array>`, `<forward_list>`, `<stack>`, `<queue>`, `<set>`, `<map>`, `<unordered_set>`, `<unordered_map>`, `<string>`, and `<bitset>`. Methods are grouped by category (e.g., constructors, access, modifiers, iterators, capacity). 

Rarely used or deprecated methods are omitted for brevity. Always include the relevant headers and use `std::` namespace. For modern features, assume C++11+ unless noted.

## Arrays (Built-in, Fixed-Size)
Built-in arrays are "old-school" C-style. For modern fixed-size, use `std::array` (below).

- **Declaration**: `T arr[N];` (fixed size N, T is type)
- **Common Operations**:
  - Access: `arr[i]` (0-based indexing, no bounds checking)
  - Size: Use `sizeof(arr)/sizeof(arr[0])` (manual, error-prone)
  - Iteration: Manual loop or `std::begin(arr)`, `std::end(arr)` (C++11+)
  - No dynamic resize, insert, or erase.

## std::vector (Dynamic Array) - `<vector>`
Resizable array, contiguous storage. Most used container.

| Category     | Method                                                                             | Description                       | Notes                  |
| ------------ | ---------------------------------------------------------------------------------- | --------------------------------- | ---------------------- |
| Constructors | `vector()`                                                                         | Empty vector                      |                        |
|              | `vector(size_type n, const T& val = T())`                                          | n elements with value val         |                        |
|              | `vector(InputIt first, InputIt last)`                                              | From range [first, last)          |                        |
|              | `vector(initializer_list<T> il)`                                                   | From initializer list (C++11)     |                        |
| Access       | `at(size_type pos)`                                                                | Element at pos (bounds-checked)   | Throws if out-of-range |
|              | `operator[](size_type pos)`                                                        | Element at pos (no bounds check)  |                        |
|              | `front()` / `back()`                                                               | First/last element                |                        |
|              | `data()`                                                                           | Pointer to underlying array       |                        |
| Modifiers    | `push_back(const T& val)` / `push_back(T&& val)`                                   | Add to end                        | Move overload (C++11)  |
|              | `pop_back()`                                                                       | Remove last                       |                        |
|              | `insert(const_iterator pos, const T& val)` / `insert(const_iterator pos, T&& val)` | Insert at pos                     | Move (C++11)           |
|              | `emplace_back(Args&&... args)`                                                     | Construct in-place at end (C++11) |                        |
|              | `emplace(const_iterator pos, Args&&... args)`                                      | Construct in-place at pos (C++11) |                        |
|              | `erase(const_iterator pos)` / `erase(const_iterator first, const_iterator last)`   | Remove at pos or range            |                        |
|              | `clear()`                                                                          | Remove all                        |                        |
|              | `resize(size_type n, T val = T())`                                                 | Resize to n (add val if growing)  |                        |
|              | `swap(vector& other)`                                                              | Swap contents                     |                        |
|              | `assign(InputIt first, InputIt last)` / `assign(initializer_list<T> il)`           | Replace contents                  | il (C++11)             |
| Iterators    | `begin()` / `end()` / `cbegin()` / `cend()`                                        | Iterators (const versions C++11)  |                        |
|              | `rbegin()` / `rend()`                                                              | Reverse iterators                 |                        |
| Capacity     | `size()`                                                                           | Number of elements                |                        |
|              | `capacity()`                                                                       | Allocated storage                 |                        |
|              | `empty()`                                                                          | True if size==0                   |                        |
|              | `reserve(size_type n)`                                                             | Reserve space for n               |                        |
|              | `shrink_to_fit()`                                                                  | Reduce capacity to size (C++11)   |                        |
| Other        | `max_size()`                                                                       | Theoretical max size              |                        |

## std::array (Fixed-Size Array) - `<array>` (C++11+)
Modern wrapper for fixed-size arrays.

| Category | Method | Description | Notes |
|----------|--------|-------------|-------|
| Constructors | `array()` | Default-initialized | |
| Access | `at(size_type pos)` | Bounds-checked access | |
| | `operator[](size_type pos)` | No bounds check | |
| | `front()` / `back()` | First/last | |
| | `data()` | Pointer to data | |
| Modifiers | `fill(const T& val)` | Fill with val | |
| | `swap(array& other)` | Swap | |
| Iterators | `begin()` / `end()` / `cbegin()` / `cend()` | Iterators | |
| | `rbegin()` / `rend()` | Reverse | |
| Capacity | `size()` | Fixed size | Compile-time constant |
| | `empty()` | True if size==0 | |
| Other | `max_size()` | Same as size() | |

## std::list (Doubly-Linked List) - `<list>`
Bidirectional, non-contiguous.

| Category | Method | Description | Notes |
|----------|--------|-------------|-------|
| Constructors | Similar to vector | | |
| Access | `front()` / `back()` | First/last | No random access |
| Modifiers | `push_front(const T& val)` / `push_front(T&&)` | Add to front | Move (C++11) |
| | `push_back(...)` / `pop_back()` / `pop_front()` | As vector, plus front | |
| | `insert(const_iterator pos, const T& val)` | Insert at pos | |
| | `emplace_front(Args&&...)` / `emplace_back(...)` / `emplace(pos, ...)` | In-place (C++11) | |
| | `erase(const_iterator pos)` / `erase(first, last)` | Remove | |
| | `clear()` / `resize(n, val)` / `assign(...)` / `swap(...)` | As vector | |
| | `remove(const T& val)` / `remove_if(Pred pred)` | Remove matching | pred (unary predicate) |
| | `unique()` / `unique(BinaryPred pred)` | Remove consecutive duplicates | |
| | `merge(list& other)` / `merge(other, Comp comp)` | Merge sorted lists | |
| | `sort()` / `sort(Comp comp)` | Sort | |
| | `reverse()` | Reverse order | |
| | `splice(const_iterator pos, list& other)` | Transfer from other | Variants for ranges/single |
| Iterators | `begin()` / `end()` etc. | Bidirectional iterators | |
| Capacity | `size()` / `empty()` / `max_size()` | As vector | No capacity/reserve |

## std::forward_list (Singly-Linked List) - `<forward_list>` (C++11+)
Forward-only, efficient for insertions.

| Category | Method | Description | Notes |
|----------|--------|-------------|-------|
| Constructors | Similar to list, no back | | |
| Access | `front()` | First element | No back/random |
| Modifiers | `push_front(...)` / `pop_front()` | Front only | |
| | `insert_after(const_iterator pos, const T& val)` | Insert after pos | |
| | `emplace_front(Args&&...)` / `emplace_after(pos, ...)` | In-place (C++11) | |
| | `erase_after(const_iterator pos)` / `erase_after(first, last)` | Erase after pos | |
| | `clear()` / `resize(n, val)` / `assign(...)` / `swap(...)` | As list | |
| | `remove(val)` / `remove_if(pred)` / `unique()` / `merge(...)` / `sort(...)` / `reverse()` / `splice_after(...)` | Similar to list | |
| Iterators | `before_begin()` / `begin()` / `end()` | Forward iterators | before_begin for inserts |
| Capacity | `empty()` / `max_size()` | No size() (O(n) to compute) | |

## std::deque (Double-Ended Queue) - `<deque>`
Efficient push/pop at both ends.

| Category | Method | Description | Notes |
|----------|--------|-------------|-------|
| Constructors | Similar to vector | | |
| Access | `at(pos)` / `operator[]` / `front()` / `back()` | As vector | |
| Modifiers | `push_front(...)` / `push_back(...)` / `pop_front()` / `pop_back()` | Both ends | |
| | `insert(pos, val)` / `emplace(pos, ...)` / `erase(pos)` / `clear()` / `resize(...)` / `assign(...)` / `swap(...)` | As vector | |
| | `emplace_front(...)` / `emplace_back(...)` | In-place both ends (C++11) | |
| | `shrink_to_fit()` | (C++11) | |
| Iterators | `begin()` / `end()` etc. | Random access iterators | |
| Capacity | `size()` / `empty()` / `max_size()` / `reserve(n)` | As vector | No capacity, but reserve |

## std::stack (LIFO Adapter) - `<stack>`
Adapter over container (default deque).

| Category | Method | Description | Notes |
|----------|--------|-------------|-------|
| Constructors | `stack()` / `stack(const Container& c)` | From container | |
| Access | `top()` | Top element | |
| Modifiers | `push(const T& val)` / `push(T&& val)` | Add to top | Move (C++11) |
| | `emplace(Args&&... args)` | In-place (C++11) | |
| | `pop()` | Remove top | |
| | `swap(stack& other)` | Swap | |
| Capacity | `size()` / `empty()` | | |

## std::queue (FIFO Adapter) - `<queue>`
Adapter over container (default deque).

| Category | Method | Description | Notes |
|----------|--------|-------------|-------|
| Constructors | Similar to stack | | |
| Access | `front()` / `back()` | Front/back | |
| Modifiers | `push(...)` / `emplace(...)` | Add to back | As stack |
| | `pop()` | Remove front | |
| | `swap(...)` | | |
| Capacity | `size()` / `empty()` | | |

## std::priority_queue (Priority Adapter) - `<queue>`
Heap-based, max-heap by default.

| Category | Method | Description | Notes |
|----------|--------|-------------|-------|
| Constructors | `priority_queue()` / `priority_queue(Comp comp)` / from range | Custom comparator | |
| Access | `top()` | Highest priority | |
| Modifiers | `push(...)` / `emplace(...)` | Insert (heapify) | |
| | `pop()` | Remove top | |
| | `swap(...)` | | |
| Capacity | `size()` / `empty()` | | |

## std::set / std::multiset (Sorted Unique/Multi Set) - `<set>`
Ordered, balanced tree (usually red-black).

| Category | Method | Description | Notes |
|----------|--------|-------------|-------|
| Constructors | `set()` / from range / with Comp | Custom comparator | multiset allows duplicates |
| Access | `find(const Key& k)` | Iterator to k or end() | |
| | `count(const Key& k)` | Number of k (1 or 0 in set) | |
| | `lower_bound(k)` / `upper_bound(k)` / `equal_range(k)` | Bounds for range | |
| Modifiers | `insert(const T& val)` / `insert(T&& val)` | Insert, returns pair<it,bool> | multiset always inserts |
| | `emplace(Args&&... args)` / `emplace_hint(hint, ...)` | In-place (C++11) | hint is iterator |
| | `erase(const_iterator pos)` / `erase(const Key& k)` / `erase(first, last)` | Remove | Returns size erased (C++11) |
| | `clear()` / `swap(...)` | | |
| Iterators | `begin()` / `end()` etc. | Sorted order | |
| Capacity | `size()` / `empty()` / `max_size()` | | |
| Other | `key_comp()` / `value_comp()` | Comparators | |

## std::map / std::multimap (Sorted Key-Value) - `<map>`
Ordered associative array.

| Category | Method | Description | Notes |
|----------|--------|-------------|-------|
| Constructors | Similar to set | Key is first, value second | multimap allows duplicate keys |
| Access | `operator[](const Key& k)` | Access/insert with default value | map only (not multi) |
| | `at(const Key& k)` | Bounds-checked (C++11) | Throws if missing |
| | `find(k)` / `count(k)` / `lower_bound(k)` etc. | As set | |
| Modifiers | `insert(const value_type& val)` / `insert(T&&)` | value_type is pair<Key, T> | Returns pair<it,bool> |
| | `emplace(Args&&... args)` / `emplace_hint(hint, ...)` | In-place for pair (C++11) | |
| | `try_emplace(k, Args&&... args)` | Insert if missing (C++17) | |
| | `erase(...)` / `clear()` / `swap(...)` | As set | |
| | `insert_or_assign(k, obj)` | Insert or assign (C++17) | |
| Iterators | As set | | |
| Capacity | As set | | |

## std::unordered_set / std::unordered_multiset (Hash Set) - `<unordered_set>` (C++11+)
Unordered, hash-based.

| Category | Method | Description | Notes |
|----------|--------|-------------|-------|
| Constructors | `unordered_set()` / with Hash, Eq / bucket_count | Custom hash/equal | unordered_multiset for duplicates |
| Access | `find(k)` / `count(k)` / `equal_range(k)` | As set, but unordered | |
| Modifiers | `insert(val)` / `emplace(...)` / `emplace_hint(hint, ...)` | As set | |
| | `erase(pos)` / `erase(k)` / `clear()` / `swap(...)` | | |
| Iterators | `begin()` / `end()` etc. | Unordered traversal | |
| Capacity | `size()` / `empty()` / `max_size()` | | |
| Hash Policy | `load_factor()` / `max_load_factor(f)` | Current/max load | |
| | `rehash(n)` / `reserve(n)` | Set buckets / preallocate | reserve (C++11) |
| | `bucket_count()` / `bucket(k)` | Bucket info | |

## std::unordered_map / std::unordered_multimap (Hash Map) - `<unordered_map>` (C++11+)
Unordered key-value.

| Category | Method | Description | Notes |
|----------|--------|-------------|-------|
| Constructors | Similar to unordered_set | | unordered_multimap for dup keys |
| Access | `operator[](k)` / `at(k)` | As map | |
| | `find(k)` / `count(k)` / `equal_range(k)` | | |
| Modifiers | `insert(val)` / `emplace(...)` / `try_emplace(k, ...)` / `insert_or_assign(k, obj)` | As map | |
| | `erase(...)` / `clear()` / `swap(...)` | | |
| Iterators | As unordered_set | | |
| Capacity | As unordered_set | | |
| Hash Policy | As unordered_set | | |

## std::string

| Category     | Method                                                                 | Description                       | Notes                                |
| ------------ | ---------------------------------------------------------------------- | --------------------------------- | ------------------------------------ |
| Constructors | `string()` / `string(const char* s)` / from range / n chars            |                                   |                                      |
| Access       | `at(pos)` / `operator[]` / `front()` / `back()` / `data()` / `c_str()` | c_str() null-terminated           | data() const char* (C++11 non-const) |
| Modifiers    | `append(const string& str)` / `append(const char* s)`                  | Concat                            | + overloads                          |
|              | `push_back(char c)` / `pop_back()`                                     |                                   |                                      |
|              | `insert(pos, str)` / `erase(pos, len)` / `replace(pos, len, str)`      |                                   |                                      |
|              | `clear()` / `resize(n, c)` / `assign(str)` / `swap(...)`               |                                   |                                      |
|              | `reserve(n)` / `shrink_to_fit()`                                       | As vector                         |                                      |
| Search       | `find(str, pos=0)` / `rfind(...)` / `find_first_of(...)` etc.          | Returns size_t, npos if not found |                                      |
| Substring    | `substr(pos=0, len=npos)`                                              |                                   |                                      |
| Compare      | `compare(str)` / operators ==, < etc.                                  |                                   |                                      |
| Iterators    | As vector                                                              |                                   |                                      |
| Capacity     | `size()` / `length()` / `empty()` / `capacity()` / `max_size()`        | length() == size()                |                                      |
| Other        | `operator+=` / global `+`                                              | Concat                            |                                      |

## std::bitset (Fixed-Size Bit Array)

| Category     | Method                                                       | Description              | Notes              |
| ------------ | ------------------------------------------------------------ | ------------------------ | ------------------ |
| Constructors | `bitset<N>()` / `bitset<N>(unsigned long val)` / from string | N template param         |                    |
| Access       | `operator[](size_t pos)` / `test(pos)`                       | Bit at pos (test throws) |                    |
|              | `count()`                                                    | Set bits                 |                    |
|              | `any()` / `all()` / `none()`                                 | Bits set? (all C++11)    |                    |
| Modifiers    | `set(pos, val=1)` / `reset(pos)` / `flip(pos)`               | Set/reset/flip bit       | All bits if no pos |
|              | `operator&= /                                                | = / ^= / ~`              | Bitwise ops        |
|              | `operator<<= / >>=`                                          | Shift                    |                    |
| Other        | `to_string()` / `to_ulong()` / `to_ullong()`                 | Convert (ullong C++11)   |                    |
|              | `size()`                                                     | N                        |                    |