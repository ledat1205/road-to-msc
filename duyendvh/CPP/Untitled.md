//===============================================
// 1. Rule of 5 + RAII
//===============================================
class Buffer {
    size_t size_;
    int* data_;
public:
    Buffer(size_t n) : size_(n), data_(new int[n]{}) {}

    ~Buffer() { delete[] data_; }

    Buffer(const Buffer& other)            // copy ctor
        : size_(other.size_), data_(new int[other.size_]) {
        std::copy(other.data_, other.data_ + size_, data_);
    }

    Buffer& operator=(const Buffer& other) { // copy assign
        if (this != &other) {
            delete[] data_;
            size_ = other.size_;
            data_ = new int[size_];
            std::copy(other.data_, other.data_ + size_, data_);
        }
        return *this;
    }

    Buffer(Buffer&& o) noexcept             // move ctor
        : size_(o.size_), data_(o.data_) {
        o.data_ = nullptr;
        o.size_ = 0;
    }

    Buffer& operator=(Buffer&& o) noexcept { // move assign
        if (this != &o) {
            delete[] data_;
            size_ = o.size_;
            data_ = o.data_;
            o.data_ = nullptr;
            o.size_ = 0;
        }
        return *this;
    }
};

//===============================================
// 2. Polymorphism + virtual destructor
//===============================================
class Animal {
public:
    virtual ~Animal() = default;
    virtual void speak() const = 0;
};

class Dog : public Animal {
public:
    void speak() const override { std::cout << "woof\n"; }
};

//===============================================
// 3. Interface Segregation + Dependency Injection
//===============================================
struct ILogger {
    virtual ~ILogger() = default;
    virtual void log(std::string_view) = 0;
};

class ConsoleLogger : public ILogger {
public:
    void log(std::string_view s) override { std::cout << s; }
};

class Service {
    ILogger& logger;
public:
    Service(ILogger& l) : logger(l) {}
    void run() { logger.log("running...\n"); }
};

//===============================================
// 4. PIMPL to reduce compile time + ABI stability
//===============================================
class Widget {
    struct Impl;
    std::unique_ptr<Impl> p;
public:
    Widget();
    ~Widget();
    void draw();
};

// impl.cpp
struct Widget::Impl {
    void draw() { std::cout << "drawing widget\n"; }
};

Widget::Widget() : p(std::make_unique<Impl>()) {}
Widget::~Widget() = default;
void Widget::draw() { p->draw(); }

//===============================================
// 5. CRTP = static polymorphism (no virtual cost)
//===============================================
template<typename Derived>
class ShapeCRTP {
public:
    void draw() {
        static_cast<Derived*>(this)->draw_impl();
    }
};

class Square : public ShapeCRTP<Square> {
public:
    void draw_impl() { std::cout << "square\n"; }
};

//===============================================
// 6. Type Erasure (std::function-like) 
//===============================================
class AnyCallable {
    struct Base { virtual void call() = 0; virtual ~Base() = default; };
    template<typename F>
    struct Model : Base {
        F f;
        Model(F&& func) : f(std::forward<F>(func)) {}
        void call() override { f(); }
    };
    std::unique_ptr<Base> self;
public:
    template<typename F>
    AnyCallable(F&& f) : self(std::make_unique<Model<F>>(std::forward<F>(f))) {}

    void operator()() { self->call(); }
};

//===============================================
// 7. Strategy Pattern
//===============================================
struct SortStrategy {
    virtual ~SortStrategy() = default;
    virtual void sort(std::vector<int>&) = 0;
};

class QuickSort : public SortStrategy {
public:
    void sort(std::vector<int>& v) override { std::sort(v.begin(), v.end()); }
};

class Context {
    std::unique_ptr<SortStrategy> st;
public:
    Context(std::unique_ptr<SortStrategy> s) : st(std::move(s)) {}
    void exec(std::vector<int>& v) { st->sort(v); }
};

//===============================================
// 8. Observer Pattern
//===============================================
class Observer {
public:
    virtual void onNotify(int v) = 0;
    virtual ~Observer() = default;
};

class Subject {
    std::vector<Observer*> obs;
public:
    void attach(Observer* o) { obs.push_back(o); }
    void notify(int v) { for (auto* o : obs) o->onNotify(v); }
};

class Display : public Observer {
public:
    void onNotify(int v) override { std::cout << "value: " << v << "\n"; }
};

//===============================================
// 9. Multi-inheritance + diamond fix using virtual base
//===============================================
class A {
public: int x = 1;
};

class B : virtual public A {};
class C : virtual public A {};
class D : public B, public C {};  // single A subobject

//===============================================
// 10. Exception-safe factory using smart pointers
//===============================================
std::unique_ptr<Animal> makeAnimal() {
    return std::make_unique<Dog>();
}

//===============================================
// 11. Abstract Factory
//===============================================
class Button { public: virtual ~Button() = default; };
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

//===============================================
// 12. Singleton (Meyer's — thread-safe)
//===============================================
class Config {
public:
    static Config& instance() {
        static Config inst;
        return inst;
    }
private:
    Config() = default;
};

//===============================================
// 13. Curiously Recurring Template Singleton
//===============================================
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

//===============================================
// 14. Visitor Pattern
//===============================================
struct Circle; struct Rect;

struct Visitor {
    virtual void visit(Circle&) = 0;
    virtual void visit(Rect&) = 0;
};

struct Shape2 {
    virtual ~Shape2() = default;
    virtual void accept(Visitor&) = 0;
};

struct Circle : Shape2 {
    void accept(Visitor& v) override { v.visit(*this); }
};
struct Rect : Shape2 {
    void accept(Visitor& v) override { v.visit(*this); }
};

//===============================================
// 15. OOP + Templates = type-safe message bus
//===============================================
struct BaseMsg { virtual ~BaseMsg() = default; };

template<typename T>
class Bus {
    std::function<void(const T&)> handler;
public:
    void subscribe(std::function<void(const T&)> h) { handler = std::move(h); }
    void publish(const T& msg) { if (handler) handler(msg); }
};

struct PriceUpdate : BaseMsg { int price; };
