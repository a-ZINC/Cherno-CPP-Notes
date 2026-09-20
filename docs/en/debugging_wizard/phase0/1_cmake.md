Here is a comprehensive, well-structured set of markdown notes based on the configuration files and architecture provided. You can save this template for your future modern C++ projects.

---

# Modern C++ CMake & Presets Reference Guide

A clean, modular, and production-ready `CMake` setup featuring **CMake Presets**, **AddressSanitizer (ASan)** integration, a **lab/experiment sandbox**, and **CTest** integration.

---

## 📂 Project Directory Structure

```text
debugging-wizard/
├── CMakeLists.txt        # Top-level build configuration & global flags
├── CMakePresets.json     # Standardized build & test configurations
├── include/              # Public headers (carried via INTERFACE target)
├── lab/                  # Throwaway experiments & prototypes
│   └── CMakeLists.txt
└── tests/                # Automated unit tests
    └── CMakeLists.txt

```

---

## 🛠️ The Four Essential CMake Files

### 1. Top-Level `CMakeLists.txt`

Sets up global standards, compiler options, and defines an **`INTERFACE` target** (`dw_options`) to cleanly propagate warning levels, include directories, and sanitizer flags to all targets.

```cmake
cmake_minimum_required(VERSION 3.25)
project(DebuggingWizard VERSION 0.1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)   # Generates compile_commands.json for clangd

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE Debug CACHE STRING "Build type" FORCE)
endif()

option(DW_SANITIZE    "Build with AddressSanitizer + UBSan" OFF)
option(DW_WERROR      "Treat warnings as errors"            OFF)
option(DW_BUILD_LAB   "Build lab experiments"               ON)
option(DW_BUILD_TESTS "Build tests"                         ON)

# INTERFACE target carries include paths, warnings, and sanitizers without compiling its own code
add_library(dw_options INTERFACE)
target_include_directories(dw_options INTERFACE ${PROJECT_SOURCE_DIR}/include)
target_compile_options(dw_options INTERFACE -Wall -Wextra -Wpedantic -Wshadow)

if(DW_WERROR)
  target_compile_options(dw_options INTERFACE -Werror)
endif()

if(DW_SANITIZE)
  target_compile_options(dw_options INTERFACE
    -fsanitize=address,undefined -fno-omit-frame-pointer)
  target_link_options(dw_options INTERFACE
    -fsanitize=address,undefined)
endif()

if(DW_BUILD_LAB)
  add_subdirectory(lab)
endif()

if(DW_BUILD_TESTS)
  enable_testing()
  add_subdirectory(tests)
endif()

```

---

### 2. `CMakePresets.json`

Manages build configurations, compiler flags, and build directory isolation so different configurations (Debug, ASan, Release) can coexist.

```json
{
  "version": 6,
  "cmakeMinimumRequired": { "major": 3, "minor": 25, "patch": 0 },
  "configurePresets": [
    {
      "name": "base",
      "hidden": true,
      "binaryDir": "${sourceDir}/build/${presetName}",
      "cacheVariables": { "CMAKE_EXPORT_COMPILE_COMMANDS": "ON" }
    },
    {
      "name": "debug",
      "displayName": "Debug (plain: gdb + Valgrind)",
      "inherits": "base",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Debug" }
    },
    {
      "name": "asan",
      "displayName": "Debug + ASan/LSan/UBSan",
      "inherits": "base",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Debug", "DW_SANITIZE": "ON" }
    },
    {
      "name": "rel",
      "displayName": "RelWithDebInfo (plain: benchmarks)",
      "inherits": "base",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "RelWithDebInfo" }
    },
    {
      "name": "asan-rel",
      "displayName": "RelWithDebInfo + ASan/UBSan (benchmarks)",
      "inherits": "base",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "RelWithDebInfo", "DW_SANITIZE": "ON" }
    }
  ],
  "buildPresets": [
    { "name": "debug",    "configurePreset": "debug" },
    { "name": "asan",     "configurePreset": "asan" },
    { "name": "rel",      "configurePreset": "rel" },
    { "name": "asan-rel", "configurePreset": "asan-rel" }
  ],
  "testPresets": [
    { "name": "debug", "configurePreset": "debug", "output": { "outputOnFailure": true } },
    { "name": "asan",  "configurePreset": "asan",  "output": { "outputOnFailure": true } }
  ]
}

```

---

### 3. `lab/CMakeLists.txt`

Utilizes a custom CMake function (`dw_lab`) to rapidly spin up throwaway executables or scratchpad code without repeating boilerplate configuration.

```cmake
# Lab = throwaway experiments and deliberately broken programs.
# Code graduates to src/ only after it has earned its place.
function(dw_lab name)
  add_executable(${name} ${name}.cpp)
  target_link_libraries(${name} PRIVATE dw_options)
endfunction()

dw_lab(hello)
dw_lab(leak_bounded)
dw_lab(bugs)
dw_lab(fd_leak)
dw_lab(fd_raii)
dw_lab(bench)

```

> **Tip:** To add a new experimental file, just create `lab/myexperiment.cpp` and append `dw_lab(myexperiment)` to this file.

---

### 4. `tests/CMakeLists.txt`

Registers automated unit tests and integrates them directly with `CTest`.

```cmake
add_executable(fd_test fd_test.cpp)
target_link_libraries(fd_test PRIVATE dw_options)
add_test(NAME fd_test COMMAND fd_test)

```

---

## 🚀 Common Workflow Commands

| Action | Command |
| --- | --- |
| **Configure** | `cmake --preset debug` |
| **Compile** | `cmake --build --preset debug -j` |
| **Run Tests** | `ctest --preset debug` |
| **Run Binary** | `./build/debug/lab/hello` |

*(Note: Replace `debug` with `asan`, `rel`, or `asan-rel` depending on your target configuration).*

---

## 💡 Key Design Takeaways

* **`INTERFACE` Libraries**: `add_library(dw_options INTERFACE)` creates a virtual target holding compiler flags and include paths. Linking it via `target_link_libraries(... PRIVATE dw_options)` keeps individual target definitions extremely concise.
* **Build Directory Isolation**: Using `${sourceDir}/build/${presetName}` inside `CMakePresets.json` guarantees that your `debug`, `asan`, and `release` artifacts live in separate folders and never conflict with each other.
* **IDE IntelliSense Support**: `CMAKE_EXPORT_COMPILE_COMMANDS ON` automatically exports `compile_commands.json`, which language servers like `clangd` rely on for fast, accurate code navigation and autocompletion.

---

Would you like to explore adding custom unit testing frameworks (like GoogleTest or Catch2) or continuous integration (CI) pipelines to this reference stack?
