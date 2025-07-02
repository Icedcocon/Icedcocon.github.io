# 构建 MCP 驱动的 AI Agent：核心能力与工作流详解


## MCP 装饰器使用说明：构建 Agent 的核心能力

MCP (Model Context Protocol) 提供了三个核心装饰器来定义服务器能力：`@mcp.tool`、`@mcp.resource` 和 `@mcp.prompt`。这些装饰器是构建功能强大、标准化的 AI Agent 服务端的基础。

- **`@mcp.tool`**: 赋予 Agent **行动** 的能力。
- **`@mcp.resource`**: 赋予 Agent **认知** (查阅资料) 的能力。
- **`@mcp.prompt`**: 赋予 Agent **结构化交互** 的能力。

这些装饰器共同构成了功能强大且标准化的 AI Agent 服务端。

### `@mcp.tool` 装饰器：定义 Agent 的"行动"

`@mcp.tool` 用于定义可由 LLM 发现和执行的功能或"行动"。这些工具使模型能够与外部世界进行交互，例如调用 API、查询数据库或操作文件。其选择和使用完全由 LLM 根据当前对话的意图自主决定，是实现 ReAct (Reasoning-Acting) 模式的关键。

> [!TIP]
> 可以将 `@mcp.tool` 视为赋予 LLM 的"超能力"。它类似于 OpenAI Functions，但遵循一个开放、标准化的协议，使其具备更好的通用性和互操作性。

```python
# server.py
from datetime import datetime
import json
import os

@mcp.tool(description="保存内容到本地文件")
def save_to_local(file_name: str, content: str) -> str:
    """
    将文本内容保存到本地文件。
    
    参数:
        file_name (str): 目标文件名, 例如 'report.json'。
        content (str): 要写入文件的文本内容。
    
    返回:
        str: 操作成功或失败的消息。
    """
    try:
        # 定义一个安全的基础路径，防止路径遍历攻击
        base_dir = "./output"
        os.makedirs(base_dir, exist_ok=True)
        safe_path = os.path.join(base_dir, os.path.basename(file_name))

        with open(safe_path, "w", encoding="utf-8") as f:
            f.write(content)
        
        return f"内容已成功保存到: {safe_path}"
    except Exception as e:
        return f"保存文件时出错: {e}"

```

**核心特点**:
- **模型驱动**: 工具被设计为由语言模型自动发现和调用，是 Agent 自主决策的核心。
- **标准化**: 遵循 MCP 规范，使任何兼容的客户端都能使用。
- **参数化**: 支持通过 JSON Schema 定义清晰的输入参数和类型。
- **文档化**: 函数的`description`和文档字符串（docstring）对于模型理解工具的用途至关重要。

> [!WARNING]
> **安全第一**: 工具的执行等同于代码执行。必须对输入参数进行严格验证和清理，以防止注入攻击。同时，任何敏感操作（如文件写入、API 调用）都应有明确的用户授权环节。

### `@mcp.resource` 装饰器：定义"认知"能力

与由模型自主决策调用的 `@mcp.tool` 不同，`@mcp.resource` 用于定义由**客户端应用程序**主动读取的上下文信息源。它赋予了 Agent "查阅特定资料"的能力，但调用哪个资源、何时调用的决策权在于客户端，而非 LLM。

根据官方定义，Resource 是"由客户端应用程序管理的上下文数据"。这通常意味着客户端的业务逻辑需要某个已知的上下文信息（如特定文件或 API 数据）来辅助任务。

> [!TIP]
> 可以将 `@mcp.resource` 理解为一个标准化的"资料库接口"。客户端应用根据需要通过这个接口精确地"借阅"某一份资料（如 `read_resource("file:///path/to/doc.txt")`），而不是让 LLM 自己去"图书馆"里找书。

```python
# server.py
import os

@mcp.resource("qwen-doc://qwen2.md", description="Qwen documentation: qwen2.md")
def get_qwen2_doc() -> str:
    """
    获取 Qwen2 的文档内容。
    此函数演示了如何安全地将本地文件注册为可供客户端访问的资源。
    
    返回:
        str: Qwen2 文档的内容，如果文件不存在或读取失败则返回错误信息。
    """
    file_path = "./docs/qwen2.md"

    try:
        os.makedirs("./docs", exist_ok=True)
        with open(file_path, "r", encoding="utf-8") as f:
            return f.read()
    except FileNotFoundError:
        return "Error: The document 'qwen2.md' was not found."
    except Exception as e:
        return f"An error occurred while reading the file: {e}"

```

**核心特点**:
- **应用驱动**: 资源的调用由客户端发起，用于获取已知的、精确的上下文信息。
- **标准化接口**: 无论后端是文件、数据库还是 API，都通过统一的 `read_resource` 接口暴露。
- **只读性**: 设计上主要用于提供只读信息，保障了后方数据源的安全。

### `@mcp.prompt` 装饰器：定义"交互快捷方式"

`@mcp.prompt` 用于定义可重用的、参数化的提示模板。它能将一个复杂的多轮对话或指令封装成一个简单的命令，作为引导用户或高效完成特定任务的"快捷方式"。

> [!TIP]
> 将 `@mcp.prompt` 视为预设的"对话流程"或"指令模板"。它可以将多步操作或复杂的指令封装成一个简单的命令，极大提升用户体验和效率。

```python
# server.py
from mcp.server.fastmcp.prompts import base

@mcp.prompt(description="生成一份简洁的周报")
def generate_weekly_report(user_name: str) -> list[base.Message]:
    """
    根据用户名生成一个周报模板。
    """
    return [
        base.UserMessage(f"请帮我生成一份周报。我的名字是 {user_name}，请总结我本周完成的主要任务、遇到的问题以及下周的计划。"),
        base.AssistantMessage("好的，请列出您本周完成的主要任务。")
    ]
```

**核心特点**:
- **可重用性**: 将常用或复杂的多步提示封装成单一、易于调用的函数。
- **参数化**: 支持传入参数，动态地生成针对特定用户或情境的提示。
- **体验优化**: 在客户端可以作为推荐指令出现，引导用户快速发起复杂任务。


## 能力匹配与选择策略

在 MCP 框架中，能力的有效运用不仅依赖于服务端的清晰定义，也取决于客户端如何根据用户意图智能地选择和调用这些能力。以下是针对三种核心能力的基础选择策略。

### `@mcp.tool` 的选择策略：模型驱动的自主决策

`@mcp.tool` 的选择完全由大语言模型（LLM）驱动，这是实现 Agent 自主性的核心。

其工作流如下：
1.  **能力声明**: 客户端在与 LLM 交互时，会将所有可用工具的元数据（包括功能描述和参数结构）作为上下文提供给模型。
2.  **模型决策**: LLM 基于其对用户意图的理解，自主判断是否需要、需要哪个以及如何调用工具来完成任务。模型的决策依据是工具的 `description` 与用户指令的语义匹配度。
3.  **执行请求**: 如果模型决定使用工具，它会生成一个结构化的调用请求（例如 JSON 对象），明确指定工具名称和参数。客户端随后负责执行此请求。

该策略构成了 ReAct (Reasoning and Acting) 模式的基础，使 LLM 从一个语言生成器转变为能够规划并执行行动的智能代理。

### `@mcp.resource` 的选择策略：客户端驱动的精确调用

与模型驱动的工具不同，`@mcp.resource` 的选择由**客户端应用程序**的业务逻辑决定。

- **决策来源**: 调用哪个资源、何时调用的决策权在于客户端，而非 LLM。客户端根据自身逻辑需要，主动、精确地请求某个已知的上下文信息。
- **调用方式**: 客户端通过 `read_resource` 函数并指定资源的唯一 URI 来直接获取数据。
- **核心用途**: 此机制为 LLM 提供精确、可信的上下文，通常用于在生成回答前，向模型"喂送"必要的背景资料，如文档、数据报告或 API 的实时信息。

这种**应用驱动**的模式确保了上下文注入的可控性和确定性。

### `@mcp.prompt` 的选择策略：客户端引导的交互启动

`@mcp.prompt` 的选择通常由客户端在交互开始时处理，用于将用户输入映射到预定义的任务流程上。

- **决策来源**: 客户端负责将用户的初始输入与可用的提示模板进行匹配。
- **匹配方式**: 匹配逻辑可以从简单的关键词或正则表达式匹配，到更高级的基于向量嵌入的语义相似度搜索。
- **核心用途**: 作为一个"交互快捷方式"，`@mcp.prompt` 旨在快速启动一个标准化的、多步骤的对话流程，从而优化用户体验并提高特定任务的完成效率。

在实际应用中，开发者可以根据具体场景，从这几种策略中进行选择或组合使用，以达到最佳的用户体验和系统效率。

## 客户端核心逻辑：一个完整的请求处理流程

定义好服务端的各项能力后，客户端的核心任务是将这些能力智能地串联起来，以完成用户的复杂请求。这通常在一个循环中完成，该循环模拟了模型的"思考-行动-观察"链条，是实现高级 Agent 功能的关键。

### 前置步骤：能力发现

在进入核心循环之前，客户端会与服务端进行一次"握手"，获取所有可用的工具、资源和提示模板的清单。这份清单是模型后续进行规划和决策的基础。

```python
# client.py (初始化阶段)
class McpClient:
    async def initialize(self):
        # 建立会话
        self.session = await mcp.connect("ws://localhost:8080")
        
        # 发现并缓存所有能力
        self.available_tools = (await self.session.list_tools()).tools
        self.available_resources = (await self.session.list_resources()).resources
        self.available_prompts = (await self.session.list_prompts()).prompts
        
        # 初始化消息历史
        self.messages = []
```

### 核心循环：思考与行动

以下是客户端处理用户请求的核心 `while` 循环。这个循环将 `@mcp.prompt`、`@mcp.resource` 和 `@mcp.tool` 的选择策略有机地结合起来，完整地模拟了 Agent 的"思考-行动-观察"链条，是实现高级功能的关键。

```python
# client.py (核心处理逻辑)
async def handle_request(self, query: str) -> str:
    """处理用户的单个请求"""

    # 1. (可选) 匹配提示模板
    # 使用 @mcp.prompt 的选择策略，尝试将用户输入匹配到一个预设的模板
    prompt_messages = await self.select_prompt_template(query)
    if prompt_messages:
        # 如果匹配成功，使用模板生成的消息作为对话的开始
        self.messages = prompt_messages
    else:
        # 否则，直接使用用户的原始输入
        self.messages = [{"role": "user", "content": query}]

    # 核心循环，模拟 Agent 的 "思考-行动" 链条
    while True:
        # 2. 上下文增强：应用驱动的直接访问
        # 根据客户端的应用逻辑，判断是否需要主动获取某个已知的 @mcp.resource 来丰富上下文
        # 注意：此步骤由客户端业务逻辑驱动，并非 RAG 的语义检索或LLM路由，存在改进点
        if self.messages[-1]["role"] == "user":
            enriched_content = await self.add_relevant_resources(
                self.messages[-1]["content"] # 基于最新的消息进行检索
            )
            self.messages[-1]["content"] = enriched_content
        
        # 3. 调用 LLM 进行决策
        # 将当前对话历史和 @mcp.tool 定义的工具列表一起发送给大语言模型
        response = await self.client.chat.completions.create(
            model=self.model,
            messages=self.messages,
            tools=self.available_tools,
            tool_choice="auto", # 让模型自主决定是否调用工具
        )
        assistant_message = response.choices[0].message
        self.messages.append(assistant_message) # 将模型的回复加入历史

        # 4. 判断并执行工具调用
        if not assistant_message.tool_calls:
            # 如果模型没有请求调用工具，说明它已准备好最终答案
            return assistant_message.content

        # 如果模型请求调用工具，则执行它们
        tool_messages = []
        for tool_call in assistant_message.tool_calls:
            tool_name = tool_call.function.name
            tool_args = json.loads(tool_call.function.arguments)

            print(f"调用工具: {tool_name}({tool_args})")
            # 通过 session.call_tool() 真正调用服务端 @mcp.tool 定义的工具
            result = await self.session.call_tool(tool_name, tool_args)
            
            tool_messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result.content[0].text, # 工具的执行结果
            })

        # 5. 将工具结果反馈给 LLM
        # 将所有工具的执行结果追加到历史中，以便模型进行下一步思考
        self.messages.extend(tool_messages)
        
        # 循环将继续，LLM 会接收到工具结果并决定下一步行动
```

#### 流程详解与策略关联

1.  **匹配提示模板 (Prompt Matching)**
    *   **做什么**: 在处理用户请求的第一步，系统会尝试使用 `@mcp.prompt` 的选择策略（如语义相似度搜索）来判断用户的输入是否能匹配上一个预定义的模板。
    *   **如何做**: `select_prompt_template` 函数会将用户的查询与所有 `available_prompts` 的描述进行比较。如果找到一个高匹配度的模板（例如，用户说"写周报"，匹配到 `generate_weekly_report`），则直接使用该模板生成初始对话消息。这是一个高效的"快捷路径"。如果未匹配到，则进入通用处理流程。

2.  **上下文增强：应用驱动的直接访问**
    *   **做什么**: 在调用 LLM 之前，系统会根据**客户端的应用逻辑**，判断是否需要主动获取某个**已知的** `@mcp.resource` 来丰富上下文。这与包含 Embedding 和 Rerank 的 RAG 语义检索有本质区别。
    *   **如何做**: 此步骤由客户端的业务逻辑驱动，而非模型。例如，`add_relevant_resources` 函数可以实现简单的规则匹配：如果用户问题中包含特定关键词（如 "Qwen2 资料"），则直接触发对预定义资源 `resource_map["qwen2"]` (即 `docs://qwen/qwen2.md`) 的读取。这个决策是精确和程序化的，它调用的是一个**已知的**、**特定的**资源，而不是在一个庞大的向量数据库中进行模糊的语义搜索。

3.  **调用 LLM 进行决策**
    *   **做什么**: 将增强后的提示、完整的对话历史以及所有可用工具的定义 (`available_tools`) 一同发送给 LLM。
    *   **如何做**: LLM 在这里扮演决策者角色。它会根据上下文，遵循 `@mcp.tool` 的选择策略，即 **模型驱动的自主决策**。模型会判断是否需要、需要哪个以及如何调用工具来完成任务。

4.  **判断与执行**
    *   **做什么**: 检查 LLM 的返回是否包含 `tool_calls`。
    *   **如何做**: 如果没有，说明 LLM 认为它已经掌握了足够的信息来直接回答用户。此时，它的 `content` 就是最终答案，循环终止。如果包含，客户端会解析这些请求并通过 `session.call_tool` 来执行对应的 `@mcp.tool`。

5.  **结果反馈与再思考**
    *   **做什么**: 将工具的执行结果（成功消息或错误信息）反馈给 LLM。
    *   **如何做**: 结果被包装成一个新的 `role: "tool"` 消息追加到对话历史中。然后，循环回到步骤 2，整个增强后的对话历史会再次被发送给 LLM。这使得 LLM 能够"看到"它指令的执行结果，并基于这个新信息进行下一步的"思考"，可能是生成最终答复，也可能是调用另一个工具。

### 用户交互流程示例：端到端演示

让我们通过一个完整的用户请求，看看上述循环是如何工作的。

> **用户**: "你好，请帮我查一下 Qwen2 的资料，并把关键信息保存到 `qwen2_summary.txt` 文件里。"

**AI 助手的处理流程:**

1.  **循环 - 第 1 次迭代**:
    *   **接收输入**: `messages` 初始化为 `[{"role": "user", "content": "你好，请帮我..."}]`。
    *   **步骤 1 (上下文增强)**: `add_relevant_resources` 函数根据**预设规则**分析查询。发现查询包含 "Qwen2 的资料" 关键词，它**直接**查找并匹配到已知的资源 URI `docs://qwen/qwen2.md`。函数读取该资源内容，并将其与原始问题合并，形成增强的上下文。**这并非 RAG 的语义搜索过程**。
    *   **步骤 2 (LLM 调用)**: LLM 接收到关于 Qwen2 的详细资料和保存文件的请求。它阅读资料，在内部生成了一段摘要，并判断出下一步需要调用 `save_to_local` 工具。
    *   **步骤 3 & 4 (工具决策)**: LLM 的返回中包含一个 `tool_call`，请求调用 `save_to_local`，参数为 `file_name="qwen2_summary.txt"` 和 `content="Qwen2 是..."` (摘要内容)。
    *   **步骤 5 (结果追加)**: 客户端 **不会** 在这一步执行工具，而是先将模型的这个"意图"（即 `tool_call` 请求）追加到 `messages` 历史中。

2.  **循环 - 第 2 次迭代**:
    *   **工具执行**: 在这次循环的开始（或作为上次循环的结尾），客户端检测到 `tool_call`，于是执行 `await session.call_tool(...)`。`save_to_local` 工具成功运行，返回 `"内容已成功保存到: output/qwen2_summary.txt"`。
    *   **步骤 5 (结果反馈)**: 这个返回消息被包装成 `role: "tool"`，并追加到 `messages` 历史中。
    *   **步骤 1 & 2 (再次调用 LLM)**: 完整的对话历史（用户原始问题 -> 增强上下文 -> 模型的工具调用请求 -> 工具的成功回执）被再次发送给 LLM。
    *   **步骤 3 (最终回答)**: LLM 现在看到了整个任务已经成功完成。它生成了最终的、面向用户的友好答复。返回的消息中 **不包含** `tool_calls`。
    *   **循环终止**: 由于没有新的 `tool_calls`，循环结束。

3.  **返回最终结果**:
    *   客户端将 LLM 的最后一条消息内容呈现给用户。

> **AI 助手**: "好的，我已经查询了 Qwen2 的相关资料，并已将摘要保存到 `output/qwen2_summary.txt` 文件中了。"

这个经过优化的流程清晰地展示了现代 AI Agent 如何通过一个"感知-思考-行动"的循环，将信息检索（RAG）、模型推理和外部工具调用无缝地结合起来，以完成用户的复杂指令。

