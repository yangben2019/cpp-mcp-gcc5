# cpp-mcp-gcc5

> A port of [hkr04/cpp-mcp](https://github.com/hkr04/cpp-mcp) that compiles on **GCC 5/6/7 (C++14)** and other legacy toolchains.

**🌐 Language / 语言:** [中文](README.md) ｜ [English](README_EN.md)

---

## Why this exists

Upstream cpp-mcp requires **C++17** and uses `std::optional`, `std::filesystem`,
structured bindings, and `std::invoke_result` — features that GCC only provides
(or fully supports) starting from version 7/8. But plenty of production projects
simply cannot upgrade their toolchain:

- The **GCC 5.4** shipped with Ubuntu 16.04
- Projects pinned to old Qt versions (Qt 4.8 / 5.7, etc.)
- Embedded / industrial environments with a locked-down old compiler

This project replaces those few constructs with **equivalent legacy code** so that
cpp-mcp compiles and runs in these environments, **without changing any protocol
behavior**.

## Features

- **GCC 5.4+ / C++14 compatible** — no dependency on `std::optional`,
  `std::filesystem`, structured bindings, or `std::invoke_result`
- **Minimal, auditable changes** — only 8 files, ~20 spots, all equivalent
  substitutions you can diff against upstream line by line
- **100% API-compatible with upstream** — zero changes needed in calling code
- **Zero external dependencies** — `httplib.h`, `json.hpp`, `base64.hpp` are all bundled
- **Both server and client supported** — Streamable HTTP / SSE / stdio transports
- **Works under both C++14 and C++17** — verified to compile with either

## Changes vs. upstream

| File | Change | Reason (what GCC 5 lacks) |
|---|---|---|
| `common/base64.hpp` | renamed parameter `alphabet` to `alph` | the parameter shadowed the enum type of the same name; GCC 5 name lookup is stricter |
| `include/mcp_thread_pool.h` | `std::invoke_result` → `std::result_of` | GCC 5 has no `invoke_result` |
| `include/mcp_tool.h` | structured binding → `std::get<>` | GCC 5 does not support structured bindings |
| `src/mcp_server.cpp` | 11 structured bindings → iterate over map `kv` | same as above |
| `src/mcp_resource.cpp` | `std::filesystem` → POSIX `stat()`; 2 structured bindings | GCC 5 has no `<filesystem>` |
| `include/mcp_server.h` | guard `<optional>` under `#ifdef MCP_SSL`; remove unused `auto_lock`/`get_lock()` | GCC 5 has no `<optional>`; the latter relied on C++17 guaranteed copy elision |
| `src/mcp_stdio_client.cpp` | 2 structured bindings | structured bindings |
| `src/mcp_sse_client.cpp` | 1 structured binding | same as above |

## Getting started

Drop this repository into your project as a subdirectory (e.g. `third_party/cpp-mcp`).

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
# Add these two for client functionality:
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

    server.start(true); // blocks; MCP endpoint is POST /mcp
    return 0;
}
```

## Known limitations

- **C++20 caveat**: `std::result_of` was removed in C++20. If you build with C++20,
  revert `std::result_of<F(Args...)>` in `include/mcp_thread_pool.h` back to upstream's
  `std::invoke_result<F, Args...>`.
- **Tested only on** Linux + GCC 5.4 (C++14 / C++17). Windows / MSVC / MinGW are untested.
- **Behavior**: all changes are compiler-compatibility only — **no protocol behavior
  was modified**; it stays identical to upstream.

## License

[MIT](LICENSE). This is a derivative work of
[hkr04/cpp-mcp](https://github.com/hkr04/cpp-mcp); original copyright belongs to
The cpp-mcp authors.
