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

`Shape* s1 = new Circle(5);`
`const Shape& s2 = Rectangle(4, 6);`
or
`Rectangle r(4, 6);
`Shape& s2 = r;`

Not allowed:
Shape& s2 = Rectangle(4, 6);
Most compilers will produce an error like:

`cannot bind non-const lvalue reference to an rvalue`

If your compiler _accepts_ it (older or non-strict mode), then:

- the temporary `Rectangle` is destroyed **at the end of the full expression**
    
- `s2` becomes a **dangling reference**
    
- using it = **undefined behavior**



---

# ✅ **2. How memory is allocated in each case**

## ✔ Case A: Using pointer (dynamic allocation → heap)

`Shape* s1 = new Circle(5);`

Memory layout:

- `new Circle(5)` allocates a **Circle object on the heap**
    
- `s1` is a pointer variable stored on the **stack**
    
- `s1` holds the address of the Circle object
    

📌 Diagram:

![[Screenshot 2025-12-28 at 13.18.40.png]]
---

## ✔ Case B: Using object directly (automatic allocation → stack)

`Circle c1(5);     // allocated on stack  
`Shape& s1 = c1;   // reference to stack object`

📌 Diagram:

![[Screenshot 2025-12-28 at 13.19.40.png]]
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


## ✅ 1. Non-virtual function: static binding (compile-time)
```

```
Why?
Because hello() is not virtual, so C++ binds the function at compile time based on the type of the pointer, not the object.

✔ Pointer type = A*
→ Call A::hello

## ✅ 2. Virtual function: dynamic binding (runtime)
```
class A {
public:
    virtual void hello() { std::cout << "A::hello\n"; }
};

class B : public A {
public:
    void hello() override { std::cout << "B::hello\n"; }
};

int main() {
    A* p = new B();
    p->hello();  // ✔ calls B::hello
}
```
Why?
`virtual` tells C++ to check the actual object type at runtime.
Compiler uses a vtable (virtual function table).
✔ Object type = B
→ Call B::hello

## ✅ 3. How virtual dispatch works (vtable)
Memory layout:
```
A* p = new B();

		 HEAP
-----------------------
| B object            |
| vptr → vtable (B)   | → [ B::hello ]
-----------------------

		 STACK
-----------------------
| p (A*)  -----------→ |
-----------------------
```

When you call:
`p->hello();`
Steps:

- Look at p (points to B object)

- Look at vptr inside the B object

- Jump to B’s vtable

- Call B::hello

This is called `dynamic dispatch`.

## ✅ 4. Without virtual ⇒ no vtable
If the function is NOT virtual:

- No vtable

- Compiler uses static binding

- Call is decided at compile time based on pointer/reference type

## ✅ 5. What if object is not a pointer? → static binding
```B b;
A a = b;     // ❗ slicing
a.hello();   // calls A::hello
```
This is object slicing:

a becomes a standalone A object

B data is discarded

No polymorphism

Even if hello() is virtual, slicing prevents polymorphic behavior.

## 🔥 Summary Table: Which function is called?
Code	Virtual?	Pointer type	Object type	Function called
A* p = new B(); p->f();	❌ No	A	B	A::f
A* p = new B(); p->f();	✔ Yes	A	B	B::f
A a = B(); a.f();	✔ Yes	A	A (sliced)	A::f
B b; A& ref = b; ref.f();	✔ Yes	A	B	B::f
￼
⭐ 6. Full example with print statements
cpp
￼Copy code
class Shape {
public:
    void type() { std::cout << "Shape::type\n"; }
    virtual void draw() { std::cout << "Shape::draw\n"; }
};

class Circle : public Shape {
public:
    void type() { std::cout << "Circle::type\n"; }
    void draw() override { std::cout << "Circle::draw\n"; }
};

int main() {
    Shape* s = new Circle();

    s->type();   // Shape::type  (non-virtual)
    s->draw();   // Circle::draw (virtual)
}
🧠 Key Rules to Remember
✔ Non-virtual function ⇒ static binding (compile time)
Based on pointer/reference type.

✔ Virtual function ⇒ dynamic binding (runtime)
Based on actual object type.

✔ Abstract class uses virtual functions
(usually = 0 pure virtual).

✔ Object slicing kills polymorphism
(never pass base objects by value).




## 2️⃣ Formal C++ value categories (important)

Modern C++ (C++11+) splits values into:

![[Screenshot 2025-12-28 at 13.05.51.png]]
But for practical use, focus on:

|Category|Example|Has identity?|Movable?|
|---|---|---|---|
|lvalue|`x`|✅|❌|
|xvalue|`std::move(x)`|✅|✅|
|prvalue|`42`, `x+1`|❌|✅|

---

## 3️⃣ lvalue & virtual dispatch (important insight)

💡 **Virtual dispatch depends on the object’s dynamic type, not whether it’s an lvalue or rvalue.**

### Example

`Shape s; s.draw();       // non-polymorphic, object type = Shape`

But:

`Circle c; Shape& s = c;   // s is an lvalue reference 
`s.draw();       // Circle::draw`

Even though:

- `s` is an **lvalue**
    
- Binding happens via **reference**
    

➡️ **Virtual dispatch still works**

---

## 4️⃣ lvalue vs rvalue affects _overload resolution_, not dispatch

### Overloading on value category

`class Shape {
`public:     
`virtual void info() &  { std::cout << "lvalue\n"; }     
`virtual void info() && { std::cout << "rvalue\n"; } };`

`Shape s; s.info();        // lvalue version  
`std::move(s).info();      // rvalue version`

📌 This is called **ref-qualified member functions**

---

## 5️⃣ Behind the scenes: what changes?

|Aspect|lvalue call|rvalue call|
|---|---|---|
|Object lifetime|Stable|Possibly temporary|
|Addressable|Yes|Maybe|
|vtable lookup|Same|Same|
|Which overload chosen|`&`|`&&`|

⚠️ The **vtable pointer does NOT change**  
Only overload selection does.

---

## 6️⃣ lvalue, rvalue & references

### Binding rules

`void f(Shape&);      // lvalue only 
`void g(Shape&&);     // rvalue only`
`void h(const Shape&); // both

`Shape s;  f(s);          // OK 
`f(Shape{});    // ❌  
`g(s);          // ❌ 
`g(Shape{});    // OK`

---

## 7️⃣ lvalue & object slicing (critical)

`Circle c; 
`Shape s = c;   // ❌ slicing`

Here:

- `s` is a **new Shape object**
    
- `Circle` part is destroyed
    
- Virtual dispatch no longer works
    

`s.draw(); // Shape::draw`

📌 **lvalue reference avoids slicing**

`Shape& s = c;  // OK`

---

## 8️⃣ Why `std::move` matters

`Shape s; 
`process(s);           // lvalue 
`process(std::move(s)); // xvalue`

`std::move`:

- does **NOT** move anything
    
- just **casts lvalue → xvalue**
    

Allows:

- move constructors
    
- rvalue-qualified overloads
    

---

## 9️⃣ Real-world design guideline

### Bad

`void setData(Shape s); // copies + slices`

### Good

`void setData(const Shape& s);   // lvalue 
`void setData(Shape&& s);        // rvalue`

---

## 🔟 One-sentence mental model

> **lvalue = named, stable object**  
> **rvalue = temporary, consumable object**  
> **Virtual dispatch cares about dynamic type, not value category**