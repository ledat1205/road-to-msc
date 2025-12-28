# ✅ **1. Why we MUST use a pointer (or reference) when dealing with abstract classes?**

Because:

### 🔹 An abstract class **cannot be instantiated**.

This is illegal:

`Shape s;   // ❌ ERROR: Shape is abstract`

And this is also illegal:

`Shape s1 = Circle(5);  // ❌ still tries to construct a Shape object`

C++ will first try to create a **Shape object**, then assign it → **not allowed**.

So you must store Circle in:

- a **pointer to Shape**
    
- or a **reference to Shape**
    

✔ Allowed:

`Shape* s1 = new Circle(5); Shape& s2 = Rectangle(4, 6);`

---

# ✅ **2. How memory is allocated in each case**

## ✔ Case A: Using pointer (dynamic allocation → heap)

`Shape* s1 = new Circle(5);`

Memory layout:

- `new Circle(5)` allocates a **Circle object on the heap**
    
- `s1` is a pointer variable stored on the **stack**
    
- `s1` holds the address of the Circle object
    

📌 Diagram:

`STACK              HEAP ------             ------------------- s1  --->   (ptr)   | Circle object   |                    | vtable pointer  |                    | radius = 5      |                    -------------------`

---

## ✔ Case B: Using object directly (automatic allocation → stack)

`Circle c1(5);     // allocated on stack  Shape& s1 = c1;   // reference to stack object`

📌 Diagram:

`STACK ------------------------- c1  | Circle object     |     | vptr              |     | radius = 5        | ------------------------- s1  -> reference to c1`

✔ No heap involved  
✔ Automatic cleanup  
✔ Faster

But…  
You cannot store different derived objects in the same array or the same variable unless you use a pointer/reference to the base class.

---

## ❗ Why you cannot do this?

`Shape s1 = Circle(5);`

### Because of **object slicing**.

C++ will:

1. Create a **Shape** object (not Circle)
    
2. Copy the Shape-part of Circle into it
    
3. Slice away all Circle-specific data
    

Since Shape is abstract → **compile error**.

Even if Shape were _not_ abstract, you would lose the derived class behavior.

# 🔥 Summary Table

| Code                       | Allowed? | Memory       | Notes                           |
| -------------------------- | -------- | ------------ | ------------------------------- |
| `Shape s;`                 | ❌        | stack        | Shape is abstract               |
| `Shape s = Circle();`      | ❌        | stack        | Object slicing + abstract class |
| `Circle c;`                | ✔        | stack        | Normal object                   |
| `Shape& ref = c;`          | ✔        | stack        | Polymorphism works              |
| `Shape* p = new Circle();` | ✔        | heap + stack | Polymorphism works              |
| `delete p;`                | ✔        | heap freed   | Must delete manually            |

---

# 🧠 Which should you use?

|Use case|Recommended|
|---|---|
|Short-lived objects|Stack object + reference|
|Need polymorphism dynamic|Pointer or smart pointer|
|Avoid manual `delete`|`std::unique_ptr<Shape>`|
|Store many heterogeneous shapes|Vector of `unique_ptr<Shape>`|

---

# ⭐ Best Modern C++ Version (No manual `new`)

Use smart pointers:

`#include <memory>  int main() {     std::unique_ptr<Shape> s1 = std::make_unique<Circle>(5);     std::unique_ptr<Shape> s2 = std::make_unique<Rectangle>(4, 6);      s1->draw();     s2->draw(); }`

- No memory leaks
    
- No manual `delete`
    
- Still polymorphic




### ✅ Use **non-virtual** when:

- Behavior **must not change** in derived classes
    
- You want **compile-time binding**
    
- It’s an **implementation detail**, not part of a polymorphic interface
    
- High-performance or low-level code (hot paths)
    

Example:

`class Shape { public:     void normalize() { /* invariant logic */ } };`

---

### ✅ Use **virtual** when:

- You want **runtime polymorphism**
    
- Behavior **depends on the dynamic type**
    
- The base class represents an **interface / contract**
    
- You expect subclassing
    

Example:

`class Shape { public:     virtual void draw() = 0; // interface };`

📌 **Rule of thumb**

> If a function is meant to be overridden → **make it virtual**  
> If it must never be overridden → **keep it non-virtual**

---

## 2️⃣ How it works behind the scenes

### 🔹 Non-virtual call (`type()`)

`s->type();`

### What the compiler does

- The **static type** of `s` is `Shape*`
    
- `type()` is **non-virtual**
    
- The call target is known **at compile time**
    

The compiler emits something like:

`call Shape::type`

No lookup. No indirection.  
This is called **static dispatch**.

---

### 🔹 Virtual call (`draw()`)

`s->draw();`

### What actually happens

Each object of a class with virtual functions contains:

- a hidden pointer: **vptr**
    
- pointing to a **vtable**
    

#### Memory layout (conceptual)

`Circle object in memory: +--------------------+ | vptr --------------+-----> Circle vtable | Shape subobject    | +--------------------+  Circle vtable: [0] Circle::draw`

At runtime:

`load vptr from object load function pointer from vtable call through that pointer`

This is **dynamic dispatch**.

---

### Why `draw()` calls `Circle::draw`

Because:

- `s` points to a `Circle` object
    
- That object’s `vptr` points to **Circle’s vtable**
    
- `draw()` entry resolves to `Circle::draw`
    

---

## 3️⃣ Key difference summarized

|Aspect|Non-virtual|Virtual|
|---|---|---|
|Binding time|Compile time|Runtime|
|Based on|Static type|Dynamic type|
|Uses vtable|❌ No|✅ Yes|
|Performance|Slightly faster|Slight overhead|
|Overridable|❌ No|✅ Yes|
|Enables polymorphism|❌|✅|