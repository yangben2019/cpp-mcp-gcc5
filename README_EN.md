# cpp-mcp-gcc5

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![C++](https://img.shields.io/badge/C%2B%2B-14%20%7C%2017-blue.svg)
![GCC](https://img.shields.io/badge/GCC-5.4%2B-orange.svg)

> A compatibility fork of [cpp-mcp](https://github.com/hkr04/cpp-mcp) that builds on legacy toolchains such as GCC 5/6/7.

**[简体中文](README.md) | [English](README_EN.md)**

---

## Overview

[cpp-mcp](https://github.com/hkr04/cpp-mcp) is a C++ implementation of the
[Model Context Protocol](https://modelcontextprotocol.io) (MCP) for building MCP servers and clients.
MCP is an open protocol that lets AI applications (such as Claude) **discover and invoke tools and
resources exposed by your program** in a uniform way.

This repository is a **compatibility fork** of cpp-mcp with a single goal: **lower the compiler
requirement** so it can be built and run in environments where the toolchain cannot be upgraded.
It is functionally identical to upstream.

## Background

Upstream cpp-mcp requires **C++17** and relies on several newer language and standard-library
features, including `std::optional`, `std::filesystem`, structured bindings, and
`std::invoke_result`. In GCC, these features are only provided or fully supported starting from
**version 7 / 8**.

Many projects still under active maintenance, however, cannot upgrade their compiler, for example:

- The **GCC 5.4** shipped with Ubuntu 16.04
- Projects pinned to an old Qt version (Qt 4.8 / 5.7, etc.)
- Embedded and industrial-control environments with a fixed toolchain

This fork replaces those few constructs with equivalent compatible code, so the library compiles
directly on GCC 5.4 and above — **without changing any protocol behavior**.

## Features

- **GCC 5.4+ / C++14 compatible** — no dependency on `std::optional`, `std::filesystem`,
  structured bindings, or `std::invoke_result`
- **Minimal, reviewable changes** — only 8 files, ~20 spots, all equivalent substitutions that
  can be diffed against upstream line by line
- **100% API-compatible with upstream** — no changes required in existing calling code
- **No external dependencies** — `httplib.h`, `json.hpp`, and `base64.hpp` are bundled
- **Both server and client supported** — Streamable HTTP / SSE / stdio transports
- **Works with both C++14 and C++17** — verified to compile under either standard

## Changes from upstream

| File | Change | Reason (feature missing in GCC 5) |
|---|---|---|
| `common/base64.hpp` | renamed parameter `alphabet` to `alph` | the parameter shadowed the enum of the same name; GCC 5 name lookup is stricter |
| `include/mcp_thread_pool.h` | `std::invoke_result` → `std::result_of` | `invoke_result` is not available in GCC 5 |
| `include/mcp_tool.h` | structured binding → `std::get<>` | GCC 5 does not support structured bindings |
| `src/mcp_server.cpp` | 11 structured bindings → iterate over map `kv` | same as above |
| `src/mcp_resource.cpp` | `std::filesystem` → POSIX `stat()`; 2 structured bindings | `<filesystem>` is not available in GCC 5 |
| `include/mcp_server.h` | guard `<optional>` under `#ifdef MCP_SSL`; remove unused `auto_lock`/`get_lock()` | GCC 5 has no `<optional>`; the latter relied on C++17 guaranteed copy elision |
| `src/mcp_stdio_client.cpp` | 2 structured bindings | structured bindings |
| `src/mcp_sse_client.cpp` | 1 structured binding | same as above |

## Requirements

- **Compiler**: GCC 5.4 or later (or any compiler that supports C++14)
- **Language standard**: C++14 / C++17
- **Link library**: `pthread` (`ws2_32` on Windows)

## Build & Integration

Place this repository inside your project (e.g. `third_party/cpp-mcp`), then integrate it with
your build system.

### qmake (.pro)

```qmake
QMAKE_CXXFLAGS += -std=c++14

# Required: the build fails to compile without these two macros
DEFINES += MCP_MAX_SESSIONS=10 MCP_SESSION_TIMEOUT=30

INCLUDEPATH += third_party/cpp-mcp/include third_party/cpp-mcp/common

SOURCES += third_party/cpp-mcp/src/mcp_message.cpp \
           third_party/cpp-mcp/src/mcp_tool.cpp \
           third_party/cpp-mcp/src/mcp_resource.cpp \
           third_party/cpp-mcp/src/mcp_server.cpp
# Add these for client functionality:
#          third_party/cpp-mcp/src/mcp_stdio_client.cpp \
#          third_party/cpp-mcp/src/mcp_sse_client.cpp

LIBS += -lpthread
```

### CMake

```cmake
add_library(cpp_mcp STATIC
    third_party/cpp-mcp/src/mcp_message.cpp
    third_party/cpp-mcp/src/mcp_tool.cpp
    third_party/cpp-mcp/src/mcp_resource.cpp
    third_party/cpp-mcp/src/mcp_server.cpp)
target_include_directories(cpp_mcp PUBLIC
    third_party/cpp-mcp/include third_party/cpp-mcp/common)
target_compile_definitions(cpp_mcp PUBLIC
    MCP_MAX_SESSIONS=10 MCP_SESSION_TIMEOUT=30)
target_compile_features(cpp_mcp PUBLIC cxx_std_14)
find_package(Threads REQUIRED)
target_link_libraries(cpp_mcp PUBLIC Threads::Threads)
```

### Minimal example

```cpp
#include "mcp_server.h"
#include "mcp_tool.h"

static mcp::json hello_handler(const mcp::json& params, const std::string& /*session_id*/) {
    return mcp::json::array({{{"type", "text"}, {"text", "Hello!"}}});
}

int main() {
    mcp::server::configuration conf;
    conf.host = "0.0.0.0";
    conf.port = 8888;

    mcp::server server(conf);
    server.set_server_info("demo", "1.0.0");
    server.set_capabilities({{"tools", mcp::json::object()}});

    server.register_tool(mcp::tool_builder("hello")
        .with_description("Say hello").build(), hello_handler);

    server.start(true); // blocks; the MCP endpoint is POST /mcp
    return 0;
}
```

## Known limitations

- **C++20 caveat**: `std::result_of` was removed in C++20. If you build with C++20, revert
  `std::result_of<F(Args...)>` in `include/mcp_thread_pool.h` back to upstream's
  `std::invoke_result<F, Args...>`.
- **Verified on**: Linux + GCC 5.4 only (C++14 and C++17). Windows / MSVC / MinGW are untested.
- **Behavior**: all changes are compiler-compatibility only — no protocol behavior was modified;
  it remains identical to upstream.

## License

Licensed under the [MIT License](LICENSE).

This is a derivative work of [hkr04/cpp-mcp](https://github.com/hkr04/cpp-mcp); original
copyright belongs to The cpp-mcp authors.

## Acknowledgements

Thanks to the authors and contributors of [cpp-mcp](https://github.com/hkr04/cpp-mcp).
All core implementation comes from upstream; this fork only adds compiler-compatibility adjustments.
