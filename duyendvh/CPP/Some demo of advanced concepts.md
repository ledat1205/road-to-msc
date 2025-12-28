# 🔥 **Rule of 5 – What is it?**

If a class **manages a resource**, and you define **ANY ONE** of these:

1. destructor
    
2. copy constructor
    
3. copy assignment
    
4. move constructor
    
5. move assignment
    

👉 **You probably need to define ALL FIVE**

---

# 📌 Rule of 5 Functions

|Function|Purpose|
|---|---|
|Destructor|release resource|
|Copy constructor|deep copy|
|Copy assignment|deep copy|
|Move constructor|steal resource|
|Move assignment|steal resource|

---

# ❌ Bad Example (Rule of 5 violation)

`class Buffer { public:     int* data;      Buffer(int n) : data(new int[n]) {}     ~Buffer() { delete[] data; } };`

❌ Problem:

`Buffer b1(10); Buffer b2 = b1;  // shallow copy → double delete!`

# ✅ Correct Example (Rule of 5 implemented)
```
// ===============================================
// 1. Rule of 5 + RAII
// ===============================================
#include <algorithm>
#include <cstddef>

class Buffer {
    size_t size_;
    int*   data_;

public:
    explicit Buffer(size_t n)
        : size_(n), data_(new int[n]{}) {}

    ~Buffer() { delete[] data_; }

    // Copy constructor
    Buffer(const Buffer& other)
        : size_(other.size_), data_(new int[other.size_]) {
        std::copy(other.data_, other.data_ + size_, data_);
    }

    // Copy assignment
    Buffer& operator=(const Buffer& other) {
        if (this != &other) {
            delete[] data_;
            size_ = other.size_;
            data_ = new int[size_];
            std::copy(other.data_, other.data_ + size_, data_);
        }
        return *this;
    }

    // Move constructor
    Buffer(Buffer&& other) noexcept
        : size_(other.size_), data_(other.data_) {
        other.size_ = 0;
        other.data_ = nullptr;
    }

    // Move assignment
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            size_       = other.size_;
            data_       = other.data_;
            other.size_ = 0;
            other.data_ = nullptr;
        }
        return *this;
    }
};
```

C++

```
// ===============================================
// 2. Polymorphism + virtual destructor
// ===============================================
#include <iostream>

class Animal {
public:
    virtual ~Animal() = default;
    virtual void speak() const = 0;
};

class Dog : public Animal {
public:
    void speak() const override {
        std::cout << "woof\n";
    }
};
```

C++

```
// ===============================================
// 3. Interface Segregation + Dependency Injection
// ===============================================
#include <iostream>
#include <string_view>

struct ILogger {
    virtual ~ILogger() = default;
    virtual void log(std::string_view msg) = 0;
};

class ConsoleLogger : public ILogger {
public:
    void log(std::string_view msg) override {
        std::cout << msg;
    }
};

class Service {
    ILogger& logger;

public:
    explicit Service(ILogger& l) : logger(l) {}

    void run() {
        logger.log("running...\n");
    }
};
```

C++

```
// ===============================================
// 4. PIMPL idiom (reduces compile time + ABI stability)
// ===============================================
#include <memory>
#include <iostream>

// widget.hpp
class Widget {
    struct Impl;
    std::unique_ptr<Impl> p;

public:
    Widget();
    ~Widget();
    void draw();
};

// widget.cpp
struct Widget::Impl {
    void draw() {
        std::cout << "drawing widget\n";
    }
};

Widget::Widget() : p(std::make_unique<Impl>()) {}
Widget::~Widget() = default;

void Widget::draw() {
    p->draw();
}
```

C++

```
// ===============================================
// 5. CRTP – static polymorphism (no virtual overhead)
// ===============================================
#include <iostream>

template<typename Derived>
class ShapeCRTP {
public:
    void draw() {
        static_cast<Derived*>(this)->draw_impl();
    }
};

class Square : public ShapeCRTP<Square> {
public:
    void draw_impl() {
        std::cout << "square\n";
    }
};
```

C++

```
// ===============================================
// 6. Type Erasure (simple std::function-like callable)
// ===============================================
#include <memory>

class AnyCallable {
    struct Base {
        virtual void call() = 0;
        virtual ~Base() = default;
    };

    template<typename F>
    struct Model : Base {
        F f;
        explicit Model(F&& func) : f(std::forward<F>(func)) {}
        void call() override { f(); }
    };

    std::unique_ptr<Base> self;

public:
    template<typename F>
    explicit AnyCallable(F&& f)
        : self(std::make_unique<Model<std::decay_t<F>>>(std::forward<F>(f))) {}

    void operator()() {
        self->call();
    }
};
```

C++

```
// ===============================================
// 7. Strategy Pattern
// ===============================================
#include <vector>
#include <algorithm>
#include <memory>

struct SortStrategy {
    virtual ~SortStrategy() = default;
    virtual void sort(std::vector<int>& v) = 0;
};

class QuickSort : public SortStrategy {
public:
    void sort(std::vector<int>& v) override {
        std::sort(v.begin(), v.end());  // using std::sort as a stand-in
    }
};

class Context {
    std::unique_ptr<SortStrategy> strategy;

public:
    explicit Context(std::unique_ptr<SortStrategy> s)
        : strategy(std::move(s)) {}

    void execute(std::vector<int>& v) {
        strategy->sort(v);
    }
};
```

C++

```
// ===============================================
// 8. Observer Pattern
// ===============================================
#include <vector>
#include <iostream>

class Observer {
public:
    virtual ~Observer() = default;
    virtual void onNotify(int value) = 0;
};

class Subject {
    std::vector<Observer*> observers;

public:
    void attach(Observer* o) {
        observers.push_back(o);
    }

    void notify(int value) {
        for (Observer* o : observers) {
            o->onNotify(value);
        }
    }
};

class Display : public Observer {
public:
    void onNotify(int value) override {
        std::cout << "value: " << value << "\n";
    }
};
```

C++

```
// ===============================================
// 9. Multiple Inheritance + Diamond Problem Fix (virtual base)
// ===============================================
class A {
public:
    int x = 1;
};

class B : virtual public A {};
class C : virtual public A {};
class D : public B, public C {};  // only one A subobject
```

C++

```
// ===============================================
// 10. Exception-safe Factory with Smart Pointers
// ===============================================
#include <memory>

std::unique_ptr<Animal> makeAnimal() {
    return std::make_unique<Dog>();
}
```

C++

```
// ===============================================
// 11. Abstract Factory
// ===============================================
#include <memory>

class Button {
public:
    virtual ~Button() = default;
};

class WinButton : public Button {};
class MacButton : public Button {};

class GUIFactory {
public:
    virtual ~GUIFactory() = default;
    virtual std::unique_ptr<Button> createButton() = 0;
};

class WinFactory : public GUIFactory {
public:
    std::unique_ptr<Button> createButton() override {
        return std::make_unique<WinButton>();
    }
};
```

C++

```
// ===============================================
// 12. Singleton (Meyers' – thread-safe, no explicit locking)
// ===============================================
class Config {
public:
    static Config& instance() {
        static Config inst;
        return inst;
    }

private:
    Config() = default;
};
```

C++

```
// ===============================================
// 13. Curiously Recurring Template Singleton
// ===============================================
template<typename T>
class SingletonCRTP {
public:
    static T& instance() {
        static T inst;
        return inst;
    }

protected:
    SingletonCRTP() = default;
};
```

C++

```
// ===============================================
// 14. Visitor Pattern
// ===============================================
struct Circle;
struct Rect;

struct Visitor {
    virtual void visit(Circle& c) = 0;
    virtual void visit(Rect& r)   = 0;
    virtual ~Visitor() = default;
};

struct Shape2 {
    virtual ~Shape2() = default;
    virtual void accept(Visitor& v) = 0;
};

struct Circle : Shape2 {
    void accept(Visitor& v) override { v.visit(*this); }
};

struct Rect : Shape2 {
    void accept(Visitor& v) override { v.visit(*this); }
};
```

C++

```
// ===============================================
// 15. OOP + Templates = Type-safe Message Bus
// ===============================================
#include <functional>

struct BaseMsg {
    virtual ~BaseMsg() = default;
};

template<typename T>
class Bus {
    std::function<void(const T&)> handler;

public:
    void subscribe(std::function<void(const T&)> h) {
        handler = std::move(h);
    }

    void publish(const T& msg) {
        if (handler) handler(msg);
    }
};

struct PriceUpdate : BaseMsg {
    int price;
};
```
