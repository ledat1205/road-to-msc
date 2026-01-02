`CMakeLists.txt` is a _script_ that tells CMake:

> “What am I building, from which source files, with which dependencies, and how those targets relate to each other?”

CMake then generates build files for **Make**, **Ninja**, **Visual Studio**, **Xcode**, etc.

---

## 1. Mental model (important)

### Without CMake

`g++ main.cpp util.cpp -Iinclude -O2 -o app`

You manually specify:

- sources
    
- include paths
    
- flags
    
- output
    

### With CMake

`CMakeLists.txt  ──▶  CMake  ──▶  Makefile / Ninja / VS project  ──▶  Compiler`

You **declare intent**, CMake handles the platform-specific details.

---

## 2. Smallest valid `CMakeLists.txt`

`cmake_minimum_required(VERSION 3.20) project(MyApp LANGUAGES CXX)  add_executable(my_app main.cpp)`

### What this means

|Line|Meaning|
|---|---|
|`cmake_minimum_required`|Ensures compatible CMake version|
|`project`|Defines project name, enables C++|
|`add_executable`|Create a build _target_|

> 🔑 **Targets** are the core concept in modern CMake.

---

## 3. Targets (the heart of CMake)

A **target** is something you build:

- executable
    
- library (static/shared/header-only)
    

`add_executable(app main.cpp) add_library(math STATIC math.cpp)`

Targets hold:

- source files
    
- include paths
    
- compile flags
    
- link libraries
    

👉 You **attach properties to targets**, not globally.

---

## 4. Linking targets

`add_library(math STATIC math.cpp) add_executable(app main.cpp)  target_link_libraries(app PRIVATE math)`

### PRIVATE / PUBLIC / INTERFACE

This controls **dependency propagation**.

|Keyword|Meaning|
|---|---|
|`PRIVATE`|Only this target needs it|
|`PUBLIC`|This target + things that link to it|
|`INTERFACE`|Only consumers need it|

Example:

`target_link_libraries(app PUBLIC math)`

If `math` needs include paths or flags, `app` gets them automatically.

---

## 5. Include directories (modern way)

❌ Old (global, bad):

`include_directories(include)`

✅ Modern (target-based):

`target_include_directories(app     PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include )`

For libraries:

`target_include_directories(math     PUBLIC include )`

Now:

- `math` can find its headers
    
- anything linking to `math` also can
    

---

## 6. Compile options & standards

`set(CMAKE_CXX_STANDARD 20) set(CMAKE_CXX_STANDARD_REQUIRED ON)`

Or better, target-based:

`target_compile_features(app PRIVATE cxx_std_20)`

Compiler flags:

`target_compile_options(app PRIVATE -Wall -Wextra)`

CMake converts this properly per compiler (GCC / Clang / MSVC).

---

## 7. Variables & generators

Common variables:

`PROJECT_NAME CMAKE_SOURCE_DIR CMAKE_BINARY_DIR CMAKE_CURRENT_SOURCE_DIR CMAKE_CURRENT_BINARY_DIR`

Example:

`add_executable(app     ${CMAKE_CURRENT_SOURCE_DIR}/main.cpp )`

CMake has:

- **configure time** (CMake runs)
    
- **build time** (compiler runs)
    

This is a critical distinction.

---

## 8. Out-of-source builds (recommended)

`project/  ├── CMakeLists.txt  ├── src/  └── build/`

Commands:

`cmake -S . -B build cmake --build build`

Why?

- clean separation
    
- easy cleanup
    
- multiple build configs (Debug/Release)
    

---

## 9. Debug vs Release

`cmake -S . -B build -DCMAKE_BUILD_TYPE=Release`

CMake defines:

- `Debug`
    
- `Release`
    
- `RelWithDebInfo`
    
- `MinSizeRel`
    

Flags are compiler-specific and automatic.

---

## 10. Multi-directory projects

Top-level:

`add_subdirectory(src) add_subdirectory(lib)`

`lib/CMakeLists.txt`

`add_library(math math.cpp)`

`src/CMakeLists.txt`

`add_executable(app main.cpp) target_link_libraries(app PRIVATE math)`

Targets become globally visible once defined.

---

## 11. Header-only libraries

`add_library(utils INTERFACE)  target_include_directories(utils     INTERFACE include )`

No source files, only usage requirements.

---

## 12. Finding external libraries

`find_package(OpenSSL REQUIRED)  target_link_libraries(app PRIVATE OpenSSL::SSL)`

CMake exports **imported targets**:

- `OpenSSL::SSL`
    
- `fmt::fmt`
    
- `Boost::boost`
    

Never manually specify `.a` or `.so` files if possible.

---

## 13. Why people hate / love CMake

### Why it feels hard

- Two execution phases
    
- Strange language
    
- Old tutorials use bad practices
    

### Why it’s powerful

- Cross-platform
    
- Dependency-aware
    
- Scales to massive codebases (LLVM, KDE, Chrome)
    

---

## 14. One golden rule (modern CMake)

> **Everything is a target.**  
> **Targets own their dependencies.**  
> **Never use global flags if you can avoid it.**

If you understand this, CMake suddenly makes sense.