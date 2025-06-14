# MCP 快速开始


## 概述

模型上下文协议（Model Context Protocol，MCP）是一种专为大型语言模型（LLM）设计的标准化协议，它允许LLM以安全、一致的方式与外部系统（如API、数据库、本地文件等）进行交互。MCP常被比作"AI的USB-C接口"，旨在提供一种统一的方式来连接LLM与它们可以利用的各种资源和工具。

> [!TIP]
> MCP的核心价值在于**标准化**和**解耦**。它解决了AI模型与外部服务交互时的碎片化问题，让模型可以跨平台、跨语言地调用各种工具，而无需关心底层的具体实现。

MCP协议的核心功能包括：
- **Resources**：类似于HTTP的`GET`端点，用于将信息加载到LLM的上下文中。
- **Tools**：类似于HTTP的`POST`端点，用于执行代码或产生实际效果。
- **Prompts**：可重用的LLM交互模板。

## 核心概念：传输方式

MCP定义了多种客户端（Client）与服务端（Server）的通信方式，称为"传输（Transport）"。其中最常见的两种是`stdio`和`sse`。

- **`stdio` (标准输入/输出)**:
  - **原理**：客户端通过启动一个子进程来运行MCP Server，并使用标准输入（stdin）和标准输出（stdout）与Server交换JSON-RPC消息。
  - **优点**：简单直接，平台兼容性高，非常适合本地集成和命令行工具。
  - **缺点**：功能相对基础，不适合网络环境下的实时交互。

- **`sse` (Server-Sent Events)**:
  - **原理**：基于HTTP协议，客户端发起一个长连接请求，服务器可以通过这个连接持续向客户端推送事件和数据。
  - **优点**：轻量级，适合服务器到客户端的单向实时数据流，例如AI模型的"打字机"效果。
  - **缺点**：通信是单向的（服务器->客户端），客户端发送消息需要额外的HTTP请求。

## MCP Server 实现

为了让MCP Client能够工作，我们首先需要一个MCP Server。这里是一个使用官方Python SDK `mcp`（集成了`FastMCP`）实现的简单示例。

> [!NOTE]
> 你需要先安装MCP的Python库: `pip install "mcp[cli]"`

这个Server提供了两个工具：`add_2_numbers`（加法）和 `multiply_2_numbers`（乘法）。

```python
# server.py
from mcp.server.fastmcp import FastMCP
from mcp.server.fastmcp.prompts import base

# To run as an SSE server, you could initialize like this:
mcp = FastMCP(name="demo", host="127.0.0.1", port=8256, sse_path="/sse")
# mcp = FastMCP()

@mcp.tool()
def add_2_numbers(a: int, b: int) -> int:
    """Adds two numbers together."""
    return a + b

@mcp.resource("config://app")
def get_config() -> str:
    """Static configuration data."""
    return "App configuration here"

@mcp.prompt()
def debug_error(error: str) -> list[base.Message]:
    """A sample prompt to help debug an error."""
    return [
        base.UserMessage("I'm seeing this error:"),
        base.UserMessage(error),
        base.AssistantMessage("I'll help debug that. What have you tried so far?"),
    ]

@mcp.tool()
def multiply_2_numbers(a: int, b: int) -> int:
    """Multiplies two numbers."""
    return a * b

if __name__ == "__main__":
    # You can choose the transport method here.
    # To run as an SSE server, use: mcp.run(transport='sse')
    # To run as a stdio server, use: mcp.run(transport='stdio')
    print("Starting MCP server with stdio transport...")
    mcp.run(transport='sse')

```
你可以通过命令行启动这个Server：
- **stdio模式**: `python server.py`
- **sse模式**: 修改`server.py`中的`mcp.run`参数为`sse`，然后运行`python server.py`。

## MCP Client 实现

MCP Client负责连接到MCP Server，列出并调用Server提供的工具。下面的Python代码展示了一个经过优化的客户端，它可以与一个大语言模型（如OpenAI的GPT系列）协作，根据用户的指令来决定调用哪个工具。

这个客户端被设计为可以灵活地通过`stdio`或`sse`方式连接到Server。

```python
# client.py
import asyncio
import json
import os

os.chdir(os.path.dirname(os.path.abspath(__file__)))

from contextlib import AsyncExitStack
from typing import Optional

from mcp import ClientSession, StdioServerParameters, stdio_client
from mcp.client.sse import sse_client
from openai import AsyncOpenAI

class GenericMCPClient:
    """
    A generic MCP client that can connect to an MCP server
    via stdio or sse, and interact with an LLM to use the server's tools.
    """
    def __init__(self, api_key: str, base_url: str, model: str):
        self.session: Optional[ClientSession] = None
        self.exit_stack = AsyncExitStack()
        self.client = AsyncOpenAI(api_key=api_key, base_url=base_url)
        self.model = model
        self.messages = []
        self.available_tools = []

    async def connect_to_stdio_server(self, command: str, args: list, env: dict = None):
        """Connects to a local MCP server using stdio."""
        print("Connecting to stdio server...")
        server_params = StdioServerParameters(command=command, args=args, env=env or {})
        stdio_transport = await self.exit_stack.enter_async_context(stdio_client(server_params))
        read, write = stdio_transport
        self.session = await self.exit_stack.enter_async_context(ClientSession(read, write))
        await self._initialize_session("stdio")

    async def connect_to_sse_server(self, server_url: str, headers: dict = None):
        """Connects to a remote MCP server using sse."""
        print(f"Connecting to SSE server at {server_url}...")
        sse_transport = await self.exit_stack.enter_async_context(sse_client(server_url, headers))
        read, write = sse_transport
        self.session = await self.exit_stack.enter_async_context(ClientSession(read, write))
        await self._initialize_session("sse")

    async def _initialize_session(self, connection_type: str):
        """Initializes the client session and lists available tools."""
        await self.session.initialize()
        response = await self.session.list_tools()
        tools = response.tools
        print(f"Successfully connected via {connection_type}. Available tools: {[tool.name for tool in tools]}")
        self.available_tools = [
            {
                "type": "function",
                "function": {
                    "name": tool.name,
                    "description": tool.description,
                    "parameters": tool.inputSchema,
                },
            }
            for tool in tools
        ]

    async def process_query(self, query: str):
        """Processes a user query by potentially calling tools via an LLM."""
        self.messages.append({"role": "user", "content": query})

        while True:
            response = await self.client.chat.completions.create(
                model=self.model,
                messages=self.messages,
                tools=self.available_tools
            )

            assistant_message = response.choices[0].message
            self.messages.append(assistant_message)

            if not assistant_message.tool_calls:
                return assistant_message.content

            for tool_call in assistant_message.tool_calls:
                tool_name = tool_call.function.name
                tool_args = json.loads(tool_call.function.arguments)

                print(f"Calling tool '{tool_name}' with args: {tool_args}")
                result = await self.session.call_tool(tool_name, tool_args)
                tool_output = result.content[0].text
                print(f"Tool '{tool_name}' result: {tool_output}")

                self.messages.append({
                    "role": "tool",
                    "content": tool_output,
                    "tool_call_id": tool_call.id
                })

    async def chat_loop(self):
        """Main chat loop to interact with the user."""
        self.messages = []
        while True:
            try:
                query = input("\nQuery: ").strip()
                if query.lower() == 'quit':
                    break
                if not query:
                    continue
                response_text = await self.process_query(query)
                print("\nAI:", response_text)
            except Exception as e:
                print(f"\nAn error occurred: {e}")

    async def cleanup(self):
        """Cleans up resources by closing the async exit stack."""
        print("Cleaning up and closing connections...")
        await self.exit_stack.aclose()

async def main():
    # Load configuration from a file (e.g., config.json)
    # The config file should contain your LLM API key, base URL, and model name.
    try:
        with open("config.json", "r") as f:
            config = json.load(f)
    except FileNotFoundError:
        print("Error: config.json not found. Please create it with your LLM configuration.")
        return

    client = GenericMCPClient(config["llm"]["api_key"], config["llm"]["base_url"], config["llm"]["model"])

    try:
        # --- Choose your connection method ---
        # 1. Connect via stdio
        # await client.connect_to_stdio_server("python", ["server.py"])

        # 2. Connect via sse (uncomment to use)
        # Make sure your server.py is running in SSE mode on the correct host/port.
        await client.connect_to_sse_server("http://127.0.0.1:8256/sse")

        await client.chat_loop()
    except Exception as e:
        print(f"A critical error occurred: {e}")
    finally:
        await client.cleanup()

if __name__ == '__main__':
    asyncio.run(main())
```

> [!TIP]
> **如何运行客户端**
> 1.  将上述代码保存为`client.py`。
> 2.  创建一个`config.json`文件，填入你的大模型API信息，例如：
>     ```json
>     {
>         "llm": {
>             "api_key": "YOUR_API_KEY",
>             "base_url": "YOUR_API_BASE_URL",
>             "model": "YOUR_MODEL_NAME"
>         }
>     }
>     ```
>     `YOUR_API_BASE_URL` 的格式为 `scheme://IP:PORT/v1`。
> 3.  确保`server.py`正在运行（以`stdio`或`sse`模式）。
> 4.  在另一个终端中运行`python client.py`。
> 5.  现在你可以在客户端终端提问，例如"12加34等于多少？"或"12乘以34呢？"，AI会调用MCP Server提供的工具来计算结果。


## 故障排查

### HTTP 404 Not Found 错误

当您尝试使用 `sse` 模式连接时，可能会遇到 `404 Not Found` 错误。这通常有两种常见原因：

1. **服务器无法找到所请求的URL路径**（例如 `/sse`）
2. **LLM API 基础URL配置错误**（缺少 `/v1` 路径）

> [!WARNING]
> 第一种错误只会在 **SSE** 模式下发生，因为 `stdio` 模式不使用HTTP网络。第二种错误则与您的LLM API连接配置相关。

请按照以下步骤检查您的配置：

#### SSE 连接问题
1.  **确认服务端已在SSE模式下运行**：
    - 您必须**先启动 `server.py`**，再启动 `client.py`。
    - 确保 `server.py` 被正确配置为以SSE模式启动。您需要修改两处：
      - **初始化**:
        ```python
        # 在 server.py 中
        mcp = FastMCP(name="demo", host="127.0.0.1", port=8256, sse_path="/sse")
        ```
      - **运行命令**:
        ```python
        # 在 server.py 的末尾
        mcp.run(transport='sse')
        ```
2.  **确认客户端配置正确**：
    - 在 `client.py` 中，确保您正在调用 `connect_to_sse_server` 方法，并且连接的URL与服务器的配置（host, port, sse_path）完全匹配。
      ```python
      # 在 client.py 中
      await client.connect_to_sse_server("http://127.0.0.1:8256/sse")
      ```

#### LLM API 连接问题
3.  **确认 `config.json` 中的 `base_url` 格式正确**：
    - 必须包含 `/v1` 路径后缀，例如：`http://localhost:8000/v1` 或 `https://api.openai.com/v1`
    - 错误示例：`http://localhost:8000`（缺少 `/v1`）
    - 正确示例：`http://localhost:8000/v1`

### 工具调用解析器错误

当使用 vLLM 作为推理后端时，您可能会遇到以下错误：

```bash
Error code: 400 - {'object': 'error', 'message': '"auto" tool choice requires --enable-auto-tool-choice and --tool-call-parser to be set', 'type': 'BadRequestError', 'param': None, 'code': 400}
```

这个错误表明 vLLM 服务器需要特定的命令行参数来支持自动工具选择。要解决此问题，请在启动 vLLM 服务器时添加以下参数：

```bash
python3 -m vllm.entrypoints.openai.api_server ....  --enable-auto-tool-choice --tool-call-parser hermes
```

这些参数启用了自动工具选择功能并配置了正确的工具调用解析器，使模型能够正确处理工具调用请求。

## 总结

MCP为语言模型与外部世界搭建了一座标准化的桥梁。通过使用`mcp`的Python库，我们可以轻松地创建服务端来暴露功能（Tools），并编写客户端来消费这些功能。这种架构使得AI应用的功能可以被无限扩展，同时也保持了代码的模块化和可维护性。

## 参考文献

- [JavaScript 版本的 MCP client 实现](https://zhuanlan.zhihu.com/p/30869685315)
- [如何使用MCP开发一个客户端和服务端](https://www.cnblogs.com/danieldaren/p/18912626)
- [工良出品 | 长文讲解 MCP 和案例实战](https://www.cnblogs.com/whuanle/p/18837493)
