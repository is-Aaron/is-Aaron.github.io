---
title: "深入浅出 JSON-RPC：轻量级远程过程调用协议"
date: 2026-03-11 10:00:00 +0800
categories: [Web, 后端]
tags: [JSON-RPC, RPC, 协议, Python, 接口设计]
---

在构建分布式系统或微服务架构时，系统之间的通信协议选择至关重要。虽然 RESTful API 和 GraphQL 是目前最流行的两种数据交互方式，但在某些场景下，我们需要一种更简单、更直接的调用方式。这时候，**JSON-RPC** 就成了一个非常优秀的选择。

本文将带你了解 JSON-RPC 的核心概念、协议结构，并通过实战演示如何快速搭建一个 JSON-RPC 服务。

## 什么是 JSON-RPC？

JSON-RPC 是一个无状态且轻量级的远程过程调用（RPC, Remote Procedure Call）协议。顾名思义，它使用 JSON（JavaScript Object Notation）作为数据格式。

与 RESTful API 强调“资源（Resources）”不同，RPC 更强调“动作（Actions）”或“方法（Methods）”。在 JSON-RPC 中，客户端发送一个包含方法名和参数的请求到服务端，服务端执行该方法并将结果返回给客户端。整个过程就像在本地调用一个函数一样自然。

JSON-RPC 的最新版本是 2.0，它具有以下特点：
- **轻量级**：数据载荷小，协议规则简单。
- **语言无关**：任何支持 JSON 的编程语言都可以实现。
- **传输无关**：可以在 HTTP、WebSocket、TCP 等多种底层协议上运行。

## 核心协议结构

JSON-RPC 2.0 的规范非常精简，主要由三种类型的数据结构组成：请求（Request）、响应（Response）和通知（Notification）。

### 1. 请求（Request）

客户端发起的 RPC 调用必须包含以下字段：
- `jsonrpc`：必须精确指定为 `"2.0"`。
- `method`：包含所要调用方法名称的字符串。
- `params`（可选）：调用方法所需的参数，可以是数组（位置参数）或对象（命名参数）。
- `id`：客户端生成的一个唯一标识符（字符串、数字或 null）。服务端在响应时必须返回这个相同的 `id`，以便客户端将响应与请求匹配。

```json
{
  "jsonrpc": "2.0",
  "method": "subtract",
  "params": [42, 23],
  "id": 1
}
```

### 2. 响应（Response）

服务端处理完请求后，返回的结果：
- `jsonrpc`：必须为 `"2.0"`。
- `result`：如果调用成功，返回方法执行的结果。与 `error` 互斥。
- `error`：如果调用失败，返回一个包含错误信息的对象（包括 `code` 和 `message`）。与 `result` 互斥。
- `id`：与请求中的 `id` 保持一致。

成功响应示例：
```json
{
  "jsonrpc": "2.0",
  "result": 19,
  "id": 1
}
```

错误响应示例：
```json
{
  "jsonrpc": "2.0",
  "error": {"code": -32601, "message": "Method not found"},
  "id": 1
}
```

### 3. 通知（Notification）

如果客户端不需要服务端的响应（例如发送日志、触发异步事件），可以发送一个**没有 `id` 字段**的请求。这种请求被称为“通知”。服务端收到通知后，不需要（也不应该）返回任何响应。

```json
{
  "jsonrpc": "2.0",
  "method": "update",
  "params": [1, 2, 3]
}
```

## 实战：使用 Python 搭建 JSON-RPC 服务

为了更好地理解 JSON-RPC，我们将使用 Python 编写一个简单的服务端，并使用命令行工具进行测试。

首先，我们需要初始化项目并安装所需的依赖。根据项目规范，我们使用 `uv` 来管理 Python 项目环境。

### 环境准备命令

```bash
uv init rpc-demo && cd rpc-demo && uv add jsonrpcserver werkzeug
```

根据命令解析规范，以下是该 Shell 命令的**全面解析**：

*   **主要用途与整体功能**：
    该命令的作用是使用 `uv` 工具初始化一个新的 Python 项目，进入该项目目录，并安装用于构建 JSON-RPC 服务的依赖库 `jsonrpcserver` 和 `werkzeug`。

*   **参数、选项和标志的详细分解**：
    *   `uv`：一个极速的 Python 包和项目管理工具（由 Rust 编写），用于替代 pip、venv 等工具。
    *   `init`：`uv` 的子命令，用于初始化一个新的 Python 项目，生成 `pyproject.toml` 等必要文件。
    *   `rpc-demo`：传递给 `init` 命令的参数，指定要创建的项目（或目录）名称。
    *   `cd`：(change directory) 改变当前工作目录的内置 Shell 命令。
    *   `rpc-demo`：传递给 `cd` 的参数，指定要进入的目标目录。
    *   `add`：`uv` 的子命令，用于向当前项目中添加（并安装）新的依赖包。
    *   `jsonrpcserver` 和 `werkzeug`：传递给 `add` 命令的参数，指定需要安装的具体 Python 依赖包的名称。

*   **特殊字符或符号的精确含义与功能**：
    *   `&&` *(逻辑与操作符)*：Shell 中的命令连接符。它的精确功能是：**仅当**前一个命令执行成功（退出状态码为 0）时，才执行紧接着的后一个命令。在这里，它确保了只有在项目初始化成功后才进入目录，只有在成功进入目录后才开始安装依赖，保证了多步操作的连贯性和安全性。
    *   `-` *(连字符)*：在 `rpc-demo` 和 `jsonrpcserver` 中，作为普通文本字符出现，常用于项目名或包名中的单词分隔，无特殊 Shell 语法含义。

### 编写服务端代码

在 `rpc-demo` 目录下创建一个 `server.py` 文件：

```python
from werkzeug.wrappers import Request, Response
from werkzeug.serving import run_simple
from jsonrpcserver import method, dispatch, Success, Result

# 定义一个名为 "ping" 的 RPC 方法
@method
def ping() -> Result:
    return Success("pong")

# 定义一个带参数的方法
@method
def add(a: int, b: int) -> Result:
    return Success(a + b)

# 处理 HTTP 请求的函数
@Request.application
def application(request: Request) -> Response:
    # 接收请求内容并分发给对应的 RPC 方法
    response = dispatch(request.get_data(as_text=True))
    return Response(response, mimetype="application/json")

if __name__ == "__main__":
    print("Starting JSON-RPC server on http://localhost:5000")
    run_simple("localhost", 5000, application)
```

启动服务端（可以通过 `uv run python server.py` 运行）。

### 客户端测试命令

服务启动后，我们可以直接使用 `curl` 命令行工具来模拟客户端发送 JSON-RPC 请求。

```bash
curl -X POST -H "Content-Type: application/json" -d '{"jsonrpc": "2.0", "method": "add", "params": {"a": 10, "b": 20}, "id": 1}' http://localhost:5000
```

根据命令解析规范，以下是该 Shell 命令的**全面解析**：

*   **主要用途与整体功能**：
    该命令的作用是使用 `curl` 工具向本地运行的服务器（端口 5000）发送一个 HTTP POST 请求，请求体包含一段符合 JSON-RPC 2.0 规范的 JSON 数据，用于调用远端的 `add` 方法并传递命名参数，随后接收并在终端显示服务器返回的计算结果。

*   **参数、选项和标志的详细分解**：
    *   `curl`：(Client URL) 一个强大的用于在命令行中通过 URL 传输数据的网络工具。
    *   `-X POST`：(--request) 指定 HTTP 请求的方法为 `POST`。在 curl 中使用 `-d` 发送数据时会自动采用 POST 方法，但显式指定使命令意图更清晰。
    *   `-H "Content-Type: application/json"`：(Header) 向请求中添加自定义的 HTTP 请求头。这里明确告诉服务器，发送的数据主体格式是 JSON。
    *   `-d '...'`：(data) 指定要发送给服务器的请求体载荷数据。
    *   `http://localhost:5000`：请求的目标 URL 地址，即我们刚才启动的服务端监听地址。

*   **特殊字符或符号的精确含义与功能**：
    *   `-` *(连字符)*：用于引导命令的短格式选项标志（如 `-X`, `-H`, `-d`）。
    *   `"` *(双引号)*：包围 `-H` 的参数，防止 Shell 解析其中的空格，将整个内容作为一个字符串传递给 curl。内部在 JSON 数据中，它是 JSON 语法的强制要求，用于包裹所有的键名和字符串值（如 `"jsonrpc"`, `"2.0"`）。
    *   `'` *(单引号)*：强引用符号。包围 `-d` 的参数（JSON 字符串）。在 Shell 中，单引号内的所有字符都会被绝对视作字面量，不进行任何变量替换或特殊字符解释。这使得我们在里面可以自由使用双引号而无需使用反斜杠转义。
    *   `{` 和 `}` *(大括号)*：在 JSON 字符串中，代表 JSON 对象的开始和结束边界。
    *   `:` *(冒号)*：在 JSON 字符串内部用于分隔键和对应的值；在 URL（`http://localhost:5000`）中用作协议分隔符和端口号分隔符。

执行上述请求后，如果服务端正常运行，终端将会输出如下标准 JSON-RPC 响应：
```json
{"jsonrpc": "2.0", "result": 30, "id": 1}
```

## 总结

JSON-RPC 凭借其轻量、直观和极简的规范，在很多内部微服务通信、区块链节点通信（如以太坊 Web3 接口）以及部分特定的前后端交互场景中占据了重要地位。如果你的业务需求仅仅是纯粹的方法调用，不需要 REST 那样复杂的资源表述与状态转移，那么 JSON-RPC 会是一个高效且优雅的选择。
