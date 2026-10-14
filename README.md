# cpp-mcp-gcc5

> 让 [hkr04/cpp-mcp](https://github.com/hkr04/cpp-mcp) 能在 **GCC 5/6/7（C++14）** 这类老工具链上编译的移植版。

**🌐 语言 / Language：** [中文](README.md) ｜ [English](README_EN.md)

---

## 为什么需要它

上游 cpp-mcp 要求 **C++17**，代码里用到了 `std::optional`、`std::filesystem`、
结构化绑定（structured bindings）、`std::invoke_result` —— 这些特性 GCC 要到 7/8
才提供或才完整。但很多在役工程根本升不了工具链：

- Ubuntu 16.04 自带的 **GCC 5.4**
- 绑死老 Qt 的工程（Qt 4.8 / 5.7 等）
- 嵌入式、工业现场里被锁死的老编译器

本项目把这几处**替换成等价的旧写法**，让 cpp-mcp 在这些环境下也能直接编译运行，
**协议行为完全不变**。

## 项目特色

- **兼容 GCC 5.4+ / C++14** —— 不依赖 `std::optional`、`std::filesystem`、
  结构化绑定、`std::invoke_result`
- **改动最小、可审计** —— 只改 8 个文件、约 20 处，全部是等价替换，可逐处对照上游 diff
- **API 与上游 100% 一致** —— 上层调用代码一个字都不用改
- **零外部依赖** —— `httplib.h`、`json.hpp`、`base64.hpp` 全部自带，无需安装任何第三方库
- **服务端 + 客户端都适配** —— Streamable HTTP / SSE / stdio 三种传输均可用
- **C++14 与 C++17 双兼容** —— 实测两种标准下都可编译

## 相对上游改了什么

| 文件 | 改动 | 原因（GCC 5 缺什么） |
|---|---|---|
| `common/base64.hpp` | 参数名 `alphabet` 重命名为 `alph` | 参数遮蔽了同名枚举类型，GCC 5 的名字查找更严格 |
| `include/mcp_thread_pool.h` | `std::invoke_result` → `std::result_of` | GCC 5 没有 `invoke_result` |
| `include/mcp_tool.h` | 结构化绑定 → `std::get<>` | GCC 5 不支持结构化绑定 |
| `src/mcp_server.cpp` | 11 处结构化绑定 → 遍历 map 的 `kv` | 同上 |
| `src/mcp_resource.cpp` | `std::filesystem` → POSIX `stat()`；2 处结构化绑定 | GCC 5 没有 `<filesystem>` |
| `include/mcp_server.h` | `<optional>` 改为 `#ifdef MCP_SSL` 内包含；删除未使用的 `auto_lock`/`get_lock()` | GCC 5 无 `<optional>`；后者按值返回依赖 C++17 保证拷贝消除 |
| `src/mcp_stdio_client.cpp` | 2 处结构化绑定 | 同结构化绑定 |
| `src/mcp_sse_client.cpp` | 1 处结构化绑定 | 同上 |

## 快速开始

把整个仓库作为子目录放进你的工程（例如 `third_party/cpp-mcp`）。

### qmake（.pro）

```qmake
QMAKE_CXXFLAGS += -std=c++14

# 必需：不定义这两个宏会编译失败
DEFINES += MCP_MAX_SESSIONS=10 MCP_SESSION_TIMEOUT=30

INCLUDEPATH += third_party/cpp-mcp/include third_party/cpp-mcp/common

SOURCES += third_party/cpp-mcp/src/mcp_message.cpp \
           third_party/cpp-mcp/src/mcp_tool.cpp \
           third_party/cpp-mcp/src/mcp_resource.cpp \
           third_party/cpp-mcp/src/mcp_server.cpp
# 需要客户端功能再补上：
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

- **C++20 需注意**：`std::result_of` 在 C++20 中已被移除。若要用 C++20 编译，请把
  `include/mcp_thread_pool.h` 里的 `std::result_of<F(Args...)>` 换回上游的
  `std::invoke_result<F, Args...>`。
- **测试环境**：仅在 Linux + GCC 5.4 上实测（C++14 / C++17）。Windows / MSVC / MinGW 未验证。
- **行为一致性**：所有改动只为适配编译器，**未改动任何协议行为**，与上游保持一致。

## 许可证

[MIT](LICENSE)。本项目是 [hkr04/cpp-mcp](https://github.com/hkr04/cpp-mcp) 的衍生版本，
原始版权归 The cpp-mcp authors 所有。
