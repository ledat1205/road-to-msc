1. Introduction
2. Literals
3. Types: Enumerations, Pointer, References, and Conversions
4. Types: Type Deduction with `auto`, and `decltype`
5. Values: Initialization, Conversion, `const`, and `constexpr`
6. Move Semantics
7. Perfect Forwarding
8. Memory
9. Functions
10. Classes: Attributes and Constructors
11. Classes: Initialization, Destructors, and Member Functions
12. Classes: `default`, and `delete`, Operator Overloading, explicit, Access Rights, Friends, and Structures
13. Inheritance: Abstract Base Classes, Access Rights, Constructors, Base Class Initializers
14. Inheritance: Destructor, Virtuality, `override`, `final`, and Multiple Inheritance
15. Templates: Functions and Classes
16. Templates: Parameters and Arguments
17. Template Specialization
18. Type Traits
19. Smart Pointers
20. STL: General Ideas (Containers, Algorithms, Iterators, Callables, range-based for-loop)
21. STL: Common Interface of the Containers
22. STL: Sequence Containers
23. STL: Associative Containers
24. STL: Algorithms
25. Strings and String Views
26. Regular Expressions
27. In- and Output
28. Threads: Creation, Data Sharing, Mutexes, and Locks
29. Threads: Thread-Local Data, Thread-Safe Initialization, Condition Variables
30. Tasks: Futures and Promises

### 1. Introduction

The introduction to C++ typically covers the basics of the language, its history, compilation model, and core syntax like classes, functions, and control structures. In industry, C++ is often used for performance-critical applications such as game engines, financial trading systems, or embedded software. A practical, complex example might involve setting up a multi-threaded application framework for a real-time data processing pipeline, like in a high-frequency trading (HFT) system where low-latency is key.

**Example: Multi-Threaded Market Data Handler**

In an HFT firm, you might build a basic framework to ingest market quotes from multiple exchanges, process them in parallel, and log anomalies. This introduces core C++ elements like includes, namespaces, classes, threads, and exception handling.

C++

```
#include <iostream>
#include <vector>
#include <thread>
#include <mutex>
#include <queue>
#include <condition_variable>
#include <chrono>
#include <stdexcept>

// Namespace for organization
namespace MarketData {

class Quote {
public:
    std::string symbol;
    double price;
    long timestamp;
    Quote(std::string sym, double pr, long ts) : symbol(sym), price(pr), timestamp(ts) {}
};

class DataProcessor {
private:
    std::queue<Quote> quoteQueue;
    std::mutex mtx;
    std::condition_variable cv;
    bool running = true;
    std::vector<std::thread> workers;

    void processWorker() {
        while (true) {
            std::unique_lock<std::mutex> lock(mtx);
            cv.wait(lock, [this] { return !quoteQueue.empty() || !running; });
            if (!running && quoteQueue.empty()) break;
            if (!quoteQueue.empty()) {
                Quote q = quoteQueue.front();
                quoteQueue.pop();
                lock.unlock();
                // Simulate complex processing: Check for price anomalies
                if (q.price < 0) {
                    throw std::runtime_error("Invalid price for " + q.symbol);
                }
                std::cout << "Processed: " << q.symbol << " at " << q.price << " ts: " << q.timestamp << std::endl;
            }
        }
    }

public:
    DataProcessor(int numThreads) {
        for (int i = 0; i < numThreads; ++i) {
            workers.emplace_back(&DataProcessor::processWorker, this);
        }
    }

    ~DataProcessor() {
        {
            std::lock_guard<std::mutex> lock(mtx);
            running = false;
        }
        cv.notify_all();
        for (auto& t : workers) t.join();
    }

    void ingestQuote(const Quote& q) {
        std::lock_guard<std::mutex> lock(mtx);
        quoteQueue.push(q);
        cv.notify_one();
    }
};

} // namespace MarketData

int main() {
    try {
        MarketData::DataProcessor processor(4); // 4 worker threads
        // Simulate ingesting quotes from exchanges
        for (int i = 0; i < 100; ++i) {
            processor.ingestQuote(MarketData::Quote("AAPL", 150.0 + i % 10, std::chrono::system_clock::now().time_since_epoch().count()));
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
        return 1;
    }
    return 0;
}
```

This example demonstrates a threaded queue for handling real-time data, mimicking industry use in finance where quotes arrive asynchronously. It introduces classes for data encapsulation, mutexes for thread safety, and RAII (Resource Acquisition Is Initialization) in the destructor for clean shutdown.

### 2. Literals

Literals in C++ are fixed values like integers, strings, or user-defined types. They can be customized with suffixes (e.g., 1u for unsigned) or raw string literals. In industry, literals are used in configuration parsing, scientific simulations, or embedded systems for constants like physical units.

**Example: Custom Literals for Unit-Safe Physics Simulations**

In aerospace software, you might use user-defined literals to enforce unit safety in calculations, preventing errors like mixing meters and feet (inspired by real Mars Orbiter failures).

C++

```
#include <iostream>
#include <cmath>

// User-defined literal for meters
constexpr double operator"" _m(long double val) { return static_cast<double>(val); }

// For kilometers
constexpr double operator"" _km(long double val) { return static_cast<double>(val) * 1000.0; }

// For seconds
constexpr double operator"" _s(long double val) { return static_cast<double>(val); }

// Complex class for trajectory calculation
class Trajectory {
private:
    double distance; // in meters
    double time;     // in seconds
public:
    Trajectory(double dist, double t) : distance(dist), time(t) {
        if (time <= 0) throw std::invalid_argument("Time must be positive");
    }
    double velocity() const { return distance / time; }
    double acceleration(double finalVel) const {
        return (finalVel - velocity()) / time; // Simplified
    }
};

int main() {
    try {
        // Practical use: Calculate satellite orbit insertion
        double launchDist = 400.0_km; // 400 km altitude
        double orbitTime = 90.0 * 60.0_s; // 90 minutes
        Trajectory satTraj(launchDist, orbitTime);
        
        double escapeVel = std::sqrt(2 * 6.67430e-11 * 5.972e24 / (6371e3 + launchDist)); // Complex formula with literals
        std::cout << "Orbital velocity: " << satTraj.velocity() << " m/s" << std::endl;
        std::cout << "Acceleration to escape: " << satTraj.acceleration(escapeVel) << " m/s²" << std::endl;
        
        // Raw string literal for config or logging
        std::string config = R"(
        {
            "distance": )" + std::to_string(launchDist) + R"(,
            "time": )" + std::to_string(orbitTime) + R"(
        }
        )";
        std::cout << "Config: " << config << std::endl;
    } catch (const std::exception& e) {
        std::cerr << "Simulation error: " << e.what() << std::endl;
    }
    return 0;
}
```

This uses custom literals for units, raw strings for JSON-like configs, and integrates into a trajectory class for simulations, common in defense or space industries.

### 3. Types: Enumerations, Pointer, References, and Conversions

This covers enums (scoped/unscoped), pointers (raw/smart), references (lvalue/rvalue), and type conversions (static_cast, etc.). In industry, these are crucial for safe memory management in systems like databases or graphics engines.

**Example: Enum-Based State Machine with Smart Pointers in a Game Server**

In online gaming, a server might manage player states using enums, pointers for dynamic objects, and references for efficient passing.

C++

```
#include <iostream>
#include <memory>
#include <vector>

// Scoped enum for player states
enum class PlayerState { Idle, Active, Disconnected, Banned };

// Class using pointers and references
class Player {
public:
    std::string name;
    PlayerState state = PlayerState::Idle;
    Player(std::string n) : name(n) {}
};

class GameServer {
private:
    std::vector<std::shared_ptr<Player>> players; // Smart pointers for ownership
public:
    void addPlayer(const std::string& name) {
        players.push_back(std::make_shared<Player>(name));
    }
    
    void updateState(Player& p, PlayerState newState) { // Reference for modification
        p.state = newState;
    }
    
    void removeDisconnected() {
        players.erase(std::remove_if(players.begin(), players.end(), [](const auto& ptr) {
            return ptr->state == PlayerState::Disconnected;
        }), players.end());
    }
    
    void printStates() const {
        for (const auto& ptr : players) {
            std::cout << ptr->name << ": " << static_cast<int>(ptr->state) << std::endl; // Enum conversion
        }
    }
};

int main() {
    GameServer server;
    server.addPlayer("Alice");
    server.addPlayer("Bob");
    
    // Reference usage
    auto& alice = *server.players[0]; // Raw pointer deref to reference
    server.updateState(alice, PlayerState::Active);
    
    // Conversion: Implicit in comparisons, explicit for output
    if (alice.state == PlayerState::Active) {
        std::cout << "Alice is active." << std::endl;
    }
    
    server.updateState(*server.players[1], PlayerState::Disconnected);
    server.removeDisconnected();
    server.printStates();
    
    // Pointer conversion example: void* for legacy API simulation
    void* rawPtr = static_cast<void*>(server.players[0].get());
    std::cout << "Raw pointer address: " << rawPtr << std::endl;
    
    return 0;
}
```

This models a game server with enum states, shared_ptr for players, references for updates, and casts for compatibility, reflecting real MMO server designs.

### 4. Types: Type Deduction with auto, and decltype

auto deduces types from initializers, while decltype gets types from expressions. Useful in templates or lambdas for generic code in libraries like Boost or STL extensions.

**Example: Generic Data Analyzer in a Big Data Pipeline**

In data analytics firms, you might use auto/decltype for flexible processing of heterogeneous datasets.

C++

```
#include <iostream>
#include <vector>
#include <type_traits>

// Template function with auto and decltype
template <typename Container>
auto average(const Container& c) {
    using value_type = decltype(*c.begin()); // Deduce element type
    value_type sum = 0;
    for (const auto& elem : c) { // auto for loop variable
        sum += elem;
    }
    return sum / static_cast<value_type>(c.size());
}

template <typename T, typename U>
auto multiply(T a, U b) -> decltype(a * b) { // Trailing return type
    return a * b;
}

class DataPipeline {
public:
    template <typename InputIt>
    void process(InputIt begin, InputIt end) {
        auto dist = end - begin; // auto deduces ptrdiff_t
        std::cout << "Processing " << dist << " items." << std::endl;
        
        if constexpr (std::is_arithmetic_v<decltype(*begin)>) { // decltype in constexpr
            std::cout << "Average: " << average(std::vector<decltype(*begin)>(begin, end)) << std::endl;
        }
    }
};

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5};
    DataPipeline pipeline;
    pipeline.process(data.begin(), data.end());
    
    auto result = multiply(3.5, 2); // Deduce double
    std::cout << "Multiplied: " << result << std::endl;
    
    decltype(result) another = 4.0; // Reuse type
    std::cout << "Another: " << another << std::endl;
    
    return 0;
}
```

This uses auto for simplicity in loops/templates and decltype for type-safe generics, akin to processing streams in cloud data pipelines like Apache Spark wrappers.

### 5. Values: Initialization, Conversion, const, and constexpr

Covers uniform initialization {}, implicit/explicit conversions, const for immutability, and constexpr for compile-time computation. Vital for bug-free code in safety-critical systems like automotive software.

**Example: Constexpr Config Parser in Embedded Automotive ECU**

In car electronics, constexpr ensures compile-time checks, const prevents mutations, and uniform init avoids narrowing.

C++

```
#include <iostream>
#include <array>
#include <string>

// Constexpr function for compile-time validation
constexpr int computeMaxSpeed(int base, int boost) {
    return base + boost;
}

class EngineConfig {
private:
    const std::string model; // Immutable after init
    std::array<int, 3> params; // Uniform init
    
public:
    EngineConfig(std::string m, int p1, int p2, int p3) 
        : model(std::move(m)), params{p1, p2, p3} {} // Move for efficiency
    
    constexpr static int maxRPM = computeMaxSpeed(6000, 1000); // Compile-time const
    
    int getParam(int idx) const { // Const method
        return params[idx];
    }
    
    // Conversion operator
    operator std::string() const {
        return model + " with params: " + std::to_string(params[0]);
    }
};

int main() {
    constexpr int safeSpeed = EngineConfig::maxRPM; // Compile-time
    std::cout << "Max RPM: " << safeSpeed << std::endl;
    
    EngineConfig config("V8", 500, 600, 700); // Uniform init
    std::cout << "Param 0: " << config.getParam(0) << std::endl;
    
    std::string desc = config; // Implicit conversion
    std::cout << "Description: " << desc << std::endl;
    
    // Attempt mutation: config.model = "V6"; // Error: const
    
    return 0;
}
```

This simulates ECU config with constexpr for constants, const for safety, and conversions for logging, preventing runtime errors in vehicles.

### 6. Move Semantics

Move semantics (std::move, rvalue refs) optimize resource transfer, avoiding copies in containers or large objects. Common in performance-sensitive apps like video encoders.

**Example: Efficient Resource Manager in a Video Streaming Service**

In media servers, move semantics handle large buffers without copying.

C++

```
#include <iostream>
#include <vector>
#include <memory>

class VideoBuffer {
private:
    std::unique_ptr<char[]> data;
    size_t size;
public:
    VideoBuffer(size_t s) : data(new char[s]), size(s) {
        std::cout << "Allocated " << s << " bytes." << std::endl;
    }
    
    // Move constructor
    VideoBuffer(VideoBuffer&& other) noexcept : data(std::move(other.data)), size(other.size) {
        other.size = 0;
        std::cout << "Moved buffer." << std::endl;
    }
    
    // Move assignment
    VideoBuffer& operator=(VideoBuffer&& other) noexcept {
        if (this != &other) {
            data = std::move(other.data);
            size = other.size;
            other.size = 0;
            std::cout << "Move assigned." << std::endl;
        }
        return *this;
    }
    
    ~VideoBuffer() {
        if (size > 0) std::cout << "Deallocated " << size << " bytes." << std::endl;
    }
};

class StreamManager {
private:
    std::vector<VideoBuffer> buffers;
public:
    void addBuffer(VideoBuffer buf) {
        buffers.push_back(std::move(buf)); // Move into vector
    }
};

int main() {
    VideoBuffer buf1(1024 * 1024); // 1MB
    VideoBuffer buf2(2 * 1024 * 1024); // 2MB
    
    StreamManager manager;
    manager.addBuffer(std::move(buf1)); // Explicit move
    manager.addBuffer(VideoBuffer(3 * 1024 * 1024)); // Temporary: implicit move
    
    // buf1 is now empty after move
    return 0;
}
```

This avoids copying large video frames, optimizing memory in streaming platforms like Netflix backends.

### 7. Perfect Forwarding

Perfect forwarding uses forwarding references (T&&) and std::forward to preserve value categories in templates. Essential for generic factories in libraries.

**Example: Generic Logger Factory in a Distributed System**

In cloud logging services, perfect forwarding creates loggers with varied args without copies.

C++

```
#include <iostream>
#include <string>
#include <utility>

// Perfect forwarding in template
template <typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

class Logger {
private:
    std::string prefix;
public:
    template <typename... Args>
    Logger(Args&&... args) : prefix(std::forward<Args>(args)...) {} // Forward ctor args
    
    void log(const std::string& msg) {
        std::cout << prefix << ": " << msg << std::endl;
    }
};

template <typename... Args>
auto createLogger(Args&&... args) {
    return make_unique<Logger>(std::forward<Args>(args)...);
}

int main() {
    std::string tempPrefix = "Error";
    auto logger1 = createLogger("Info"); // Lvalue
    auto logger2 = createLogger(std::move(tempPrefix)); // Rvalue move
    
    logger1->log("System started.");
    logger2->log("Warning issued.");
    
    return 0;
}
```

This forwards arguments to create loggers efficiently, used in systems like Kubernetes logging.

### 8. Memory

Memory management includes new/delete, allocators, smart pointers, and alignment. Critical for leak-free code in long-running servers.

**Example: Custom Allocator for Cache in a Web Server**

In high-traffic servers like Nginx extensions, custom allocators optimize memory for caches.

C++

```
#include <iostream>
#include <memory>
#include <vector>
#include <map>

// Custom allocator for aligned memory
template <typename T>
class AlignedAllocator {
public:
    using value_type = T;
    T* allocate(std::size_t n) {
        return static_cast<T*>(aligned_alloc(alignof(T), n * sizeof(T)));
    }
    void deallocate(T* p, std::size_t) { free(p); }
};

class Cache {
private:
    std::map<std::string, std::vector<char, AlignedAllocator<char>>> entries; // Custom allocator
public:
    void insert(const std::string& key, std::vector<char> data) {
        entries[key] = std::move(data); // Move to avoid copy
    }
    
    const std::vector<char, AlignedAllocator<char>>& get(const std::string& key) {
        auto it = entries.find(key);
        if (it == entries.end()) throw std::out_of_range("Key not found");
        return it->second;
    }
};

int main() {
    Cache webCache;
    std::vector<char> pageData(4096, 'A'); // Simulate page
    webCache.insert("/index.html", std::move(pageData));
    
    try {
        const auto& retrieved = webCache.get("/index.html");
        std::cout << "Retrieved size: " << retrieved.size() << std::endl;
    } catch (const std::exception& e) {
        std::cerr << "Cache error: " << e.what() << std::endl;
    }
    
    return 0;
}
```

This uses custom allocators for SSE-aligned buffers in caches, improving performance in web servers.

### 9. Functions

Functions include lambdas, overloads, variadics, and std::function. Used for callbacks in event-driven systems.

**Example: Variadic Callback System in an IoT Device Manager**

In IoT platforms, functions handle dynamic events with lambdas and variadics.

C++

```
#include <iostream>
#include <functional>
#include <vector>
#include <string>

// Variadic template for logging
template <typename... Args>
void log(Args&&... args) {
    (std::cout << ... << std::forward<Args>(args)) << std::endl;
}

class DeviceManager {
private:
    std::vector<std::function<void(const std::string&)>> callbacks;
public:
    void registerCallback(std::function<void(const std::string&)> cb) {
        callbacks.push_back(cb);
    }
    
    void triggerEvent(const std::string& event) {
        for (const auto& cb : callbacks) {
            cb(event);
        }
    }
};

int main() {
    DeviceManager mgr;
    
    // Lambda as function
    auto lambdaCb = [](const std::string& ev) {
        log("Lambda handled: ", ev);
    };
    mgr.registerCallback(lambdaCb);
    
    // Overloaded function
    struct Overload {
        void operator()(const std::string& ev) {
            log("Overload handled: ", ev);
        }
    };
    mgr.registerCallback(Overload{});
    
    mgr.triggerEvent("Sensor Alert");
    
    // Variadic in action
    log("Event with params: ", 42, " ", 3.14, " ", "done");
    
    return 0;
}
```