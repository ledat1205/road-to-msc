## Variable Initialization (Initial-ization)

**Initialization** is the process of specifying an **initial value** for a variable at the moment it is defined, combining two steps into one.

### The 5 Common Forms of Initialization

The C++ standard includes several forms for initialization. **List-initialization** is the modern, preferred approach.

|**Form**|**Syntax/Example**|**Description**|**Modern?**|
|---|---|---|---|
|**Default-initialization**|`int a;`|**No initializer provided.** Often leaves the variable with an **indeterminate ("garbage") value** (unless it's a class type or global variable). **Avoid.**|No|
|**Copy-initialization**|`int b = 5;`|Initial value provided after the **`=`** sign. Inherited from the C language.|Legacy|
|**Direct-initialization**|`int c(6);`|Initial value provided in **parentheses `()`**.|Legacy|
|**Direct-list-initialization**|`int d {7};`|Initial value in **braces `{}`**. This is a form of **List-Initialization** and is generally **preferred**.|**Yes**|
|**Value-initialization**|`int e {};`|**Empty braces `{}`**. In most cases for fundamental types, this performs **Zero-initialization** (sets the value to 0).|**Yes**|

### Key Benefit of List-Initialization (`{}`)

List-initialization (`{value}` or `{}`) is preferred because it **disallows narrowing conversions**.

|**Conversion Type**|**Syntax**|**Result**|**Status**|
|---|---|---|---|
|**Narrowing Conversion**|`int w { 4.5 };`|Compiler **Error** (Prevents data loss of `.5`).|**Safe**|
|**Non-List Conversion**|`int w = 4.5;`|Compiles, but `w` is silently initialized to `4`.|Risky|


## Fundamental Data Types & Declarations

These keywords define the basic types of data a variable can hold and how its declaration is treated.

|**Keyword**|**Purpose**|**Example / Notes**|
|---|---|---|
|**bool**|Boolean type (holds `true` or `false`).|`bool is_valid = true;`|
|**char**, **char8_t** (C++20), **char16_t**, **char32_t**|Types for storing single characters or small integers. `charX_t` are fixed-size Unicode types.|`'a'`, `u8'x'`, `u'y'`, `U'z'`|
|**int**, **long**, **short**|Primary types for storing integers of various guaranteed minimum sizes.|`long counter = 10L;`|
|**float**, **double**|Types for storing single- and double-precision floating-point numbers.|`double pi = 3.14159;`|
|**signed**, **unsigned**|Specifies if an integer type should be signed (can hold negative values) or unsigned (only non-negative values).|`unsigned int u;`|
|**void**|Indicates an absence of type (e.g., function return value) or a generic pointer.|`void* ptr;`|
|**wchar_t**|Wide character type (size is platform-dependent).|`wchar_t wc = L'A';`|
|**auto**|Directs the compiler to deduce the variable's type from its initializer.|`auto num = 5;` (deduced as `int`)|
|**typedef**|Creates an alias (new name) for an existing type.|`typedef int score;`|
|**using**|Used for creating type aliases (modern alternative to `typedef`) and for namespace declarations.|`using T = int;`|

---

## Storage & Type Qualifiers

These modify how and where a variable's memory is managed and accessed.

| **Keyword**           | **Purpose**                                                                                                                                | **Example / Notes**                             |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------- |
| **const**             | Declares a variable as **read-only** (cannot be modified after initialization).                                                            | `const int size = 5;`                           |
| **constexpr**         | Specifies that a value or function can be evaluated at **compile time**.                                                                   | `constexpr int sq(int x) { return x*x; }`       |
| **consteval** (C++20) | Specifies a function must be evaluated at **compile time** (immediately).                                                                  | Enforces compile-time execution.                |
| **constinit** (C++20) | Guarantees a variable has **static initialization** (at compile time).                                                                     | Prevents issues with initialization order.      |
| **volatile**          | Tells the compiler that a variable's value may be changed by external factors (e.g., hardware), preventing aggressive optimization.        | Used in multi-threaded or embedded programming. |
| **extern**            | Declares a variable or function is defined in another translation unit (file).                                                             | Used for specifying **external linkage**.       |
| **static**            | Changes storage duration and linkage. Inside a function: keeps value between calls. Outside a class: restricts visibility to current file. | Used for specifying **internal linkage**.       |
| **thread_local**      | Specifies that a variable will be distinct for each thread.                                                                                | Used for thread-safe global variables.          |
| **mutable**           | Allows a `const` object to modify a specific member variable.                                                                              | Used in `const` member functions of a class.    |
| **register**          | **Deprecated in C++17.** Hint to the compiler to store the variable in a CPU register (usually ignored).                                   |                                                 |

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

|**Keyword**|**Purpose**|**Context**|
|---|---|---|
|**class**, **struct**|Defines a custom user-defined type (aggregate data structure). **`class`** defaults to **private** members; **`struct`** defaults to **public**.|`class MyClass { ... };`|
|**union**|Defines a type where all members share the same memory location.|Only one member can be used at a time.|
|**enum**|Declares an enumeration type (a set of named constant integer values).|`enum Color { RED, BLUE };`|
|**public**, **protected**, **private**|**Access specifiers** that control the visibility of class/struct members.|Used inside `class` or `struct` definition.|
|**virtual**|Used for functions in a base class to allow derived classes to override the implementation (**Polymorphism**).|`virtual void func();`|
|**friend**|Grants a non-member function or another class access to a class's private and protected members.|Used inside the class definition.|
|**this**|A pointer to the object on which a member function is currently executing.|Used inside member functions.|
|**explicit**|Prevents a constructor from being used for implicit conversions.|Used before a single-argument constructor.|
|**namespace**|Provides a scope to organize code and prevent name collisions.|`namespace MyLib { ... }`|
|**operator**|Used to **overload** an operator (e.g., `+`, `<<`) for a user-defined type.|`MyClass operator+(const MyClass& other);`|

---

## Casting, Type ID, & Memory

Keywords used for converting between types, runtime type information, and memory operations.

|**Keyword**|**Purpose**|**Context / Notes**|
|---|---|---|
|**static_cast**|Converts between related types (e.g., base $\leftrightarrow$ derived class, `int` $\leftrightarrow$ `float`). Safest C++ cast.|`static_cast<int>(f);`|
|**const_cast**|Used only to add or remove `const` or `volatile` qualifiers from a type.|**Use with caution.**|
|**reinterpret_cast**|Converts between unrelated types (e.g., a pointer to an integer). Most dangerous C++ cast.|Used for low-level, bitwise conversions.|
|**dynamic_cast**|Performs checked conversion of polymorphic types during **run time**.|Used primarily for safe downcasting in class hierarchies.|
|**typeid**|Returns an object of type `std::type_info` that describes the object's type at **run time**.|Used in conjunction with `dynamic_cast`.|
|**new**|Allocates memory dynamically (on the heap) and returns a pointer to the allocated object(s).|`int* p = new int[10];`|
|**delete**|Deallocates dynamically allocated memory.|`delete p;`|
|**nullptr**|A literal value that represents a null pointer (safer than using the integer `0` or `NULL`).|`int* p = nullptr;`|
|**sizeof**|Returns the size in bytes of a variable or a type.|`sizeof(int)` or `sizeof(my_var)`|
|**alignas**|Specifies the desired memory alignment for a variable or user-defined type.|`alignas(16) int x;`|
|**alignof**|Queries the alignment requirement of a type.|`alignof(int)`|
|**decltype**|Determines the type of an expression at **compile time**.|`decltype(x + y) z;`|

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
