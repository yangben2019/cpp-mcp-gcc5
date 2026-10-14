# cpp-mcp-gcc5

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![C++](https://img.shields.io/badge/C%2B%2B-14%20%7C%2017-blue.svg)
![GCC](https://img.shields.io/badge/GCC-5.4%2B-orange.svg)

> [cpp-mcp](https://github.com/hkr04/cpp-mcp) 的兼容性分支，使其可在 GCC 5/6/7 等旧版工具链上编译运行。

**[简体中文](README.md) | [English](README_EN.md)**

---

## 简介

[cpp-mcp](https://github.com/hkr04/cpp-mcp) 是一个用 C++ 实现的
[Model Context Protocol](https://modelcontextprotocol.io)（MCP）框架，用于构建 MCP 服务端与客户端。
MCP 是一套开放协议，让 AI 应用（如 Claude）能够以统一的方式**发现并调用你程序中的工具与资源**。

本仓库是 cpp-mcp 的**移植分支**，目标只有一个：**降低编译器版本要求**，使其在无法升级工具链的环境中同样可以编译运行。功能与上游保持一致。

## 背景

上游 cpp-mcp 要求 **C++17**，并使用了若干较新的语言与标准库特性，包括
`std::optional`、`std::filesystem`、结构化绑定（structured bindings）与
`std::invoke_result`。在 GCC 中，这些特性直到 **7 / 8** 才被提供或完整支持。

然而，许多仍在维护的项目难以升级编译器，例如：

- Ubuntu 16.04 自带的 **GCC 5.4**
- 依赖旧版 Qt（Qt 4.8 / 5.7 等）的工程
- 嵌入式、工业控制等对工具链有硬性约束的场景

本分支将上述少数几处替换为等价的兼容写法，使其在 GCC 5.4 及以上版本中可直接编译，
且**不改变任何协议行为**。

## 特性

- **兼容 GCC 5.4+ / C++14** —— 不依赖 `std::optional`、`std::filesystem`、结构化绑定、`std::invoke_result`
- **改动精简、可审阅** —— 仅涉及 8 个文件、约 20 处修改，均为等价替换，可逐处对照上游
- **API 与上游完全一致** —— 现有调用代码无需任何改动
- **无外部依赖** —— `httplib.h`、`json.hpp`、`base64.hpp` 均已内置
- **服务端与客户端均可用** —— 支持 Streamable HTTP / SSE / stdio 三种传输
- **同时兼容 C++14 与 C++17** —— 两种标准下均通过编译

## 与上游的差异

| 文件 | 改动 | 原因（GCC 5 缺失的特性） |
|---|---|---|
| `common/base64.hpp` | 参数名 `alphabet` 重命名为 `alph` | 参数遮蔽了同名枚举类型，GCC 5 的名字查找更严格 |
| `include/mcp_thread_pool.h` | `std::invoke_result` → `std::result_of` | GCC 5 未提供 `invoke_result` |
| `include/mcp_tool.h` | 结构化绑定 → `std::get<>` | GCC 5 不支持结构化绑定 |
| `src/mcp_server.cpp` | 11 处结构化绑定 → 遍历 map 的 `kv` | 同上 |
| `src/mcp_resource.cpp` | `std::filesystem` → POSIX `stat()`；2 处结构化绑定 | GCC 5 未提供 `<filesystem>` |
| `include/mcp_server.h` | `<optional>` 改为在 `#ifdef MCP_SSL` 内包含；移除未使用的 `auto_lock`/`get_lock()` | GCC 5 未提供 `<optional>`；后者按值返回依赖 C++17 的保证拷贝消除 |
| `src/mcp_stdio_client.cpp` | 2 处结构化绑定 | 同结构化绑定 |
| `src/mcp_sse_client.cpp` | 1 处结构化绑定 | 同上 |

## 环境要求

- **编译器**：GCC 5.4 及以上（或任何支持 C++14 的编译器）
- **语言标准**：C++14 / C++17
- **链接库**：`pthread`（Windows 下为 `ws2_32`）

## 构建与集成

将本仓库放入工程目录（例如 `third_party/cpp-mcp`）后，按所用构建系统引入：

### qmake（.pro）

```qmake
QMAKE_CXXFLAGS += -std=c++14

# 必需：未定义这两个宏将导致编译失败
DEFINES += MCP_MAX_SESSIONS=10 MCP_SESSION_TIMEOUT=30

INCLUDEPATH += third_party/cpp-mcp/include third_party/cpp-mcp/common

SOURCES += third_party/cpp-mcp/src/mcp_message.cpp \
           third_party/cpp-mcp/src/mcp_tool.cpp \
           third_party/cpp-mcp/src/mcp_resource.cpp \
           third_party/cpp-mcp/src/mcp_server.cpp
# 需要客户端功能时再追加：
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

### 最小示例

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
        .with_description("打个招呼").build(), hello_handler);

    server.start(true); // 阻塞运行；MCP 端点为 POST /mcp
    return 0;
}
```

## 已知限制

- **C++20 需注意**：`std::result_of` 已于 C++20 移除。若使用 C++20 编译，请将
  `include/mcp_thread_pool.h` 中的 `std::result_of<F(Args...)>` 改回上游的
  `std::invoke_result<F, Args...>`。
- **验证环境**：目前仅在 Linux + GCC 5.4 上完成测试（C++14 与 C++17）。Windows / MSVC / MinGW 尚未验证。
- **行为一致性**：所有改动仅为编译器兼容性适配，未修改任何协议行为，与上游保持一致。

## 许可证

本项目采用 [MIT 许可证](LICENSE)。

本项目为 [hkr04/cpp-mcp](https://github.com/hkr04/cpp-mcp) 的衍生作品，原始版权归
The cpp-mcp authors 所有。

## 致谢

感谢 [cpp-mcp](https://github.com/hkr04/cpp-mcp) 的作者与贡献者。本项目的全部核心实现均来自上游，
本分支仅在此基础上做了编译器兼容性适配。
