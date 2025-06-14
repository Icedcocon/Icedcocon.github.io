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

一个强大的 MCP 服务端不仅需要定义有用的能力，客户端还需要智能地根据用户输入选择合适的能力。以下是针对这三种核心能力目前主流的匹配与选择策略。

### `@mcp.tool` 的选择策略：模型驱动的自主决策

工具的选择几乎完全由大语言模型（LLM）本身驱动，这体现了现代 Agent 的核心理念——"思考-行动"（Reasoning-Acting）。

1.  **能力注册与描述**: 客户端首先会获取所有可用工具的列表，每个工具都附有清晰的、人类可读的 `description`（描述）和结构化的参数定义（如 JSON Schema）。
2.  **上下文情景化**: 在向 LLM 发起请求时，客户端会将这份工具列表作为上下文的一部分（通常是通过 `tools` 参数）一并发送给模型。
3.  **模型自主判断**: LLM 在理解用户查询（Query）的意图后，会利用其强大的推理能力来判断：
    *   **是否需要使用工具？** 如果用户的请求可以通过自身知识库直接回答，则无需调用工具。
    *   **使用哪个工具？** 模型会匹配用户意图和各个工具的 `description`。例如，当用户说"把这个保存成文件"，模型会发现 `save_to_local` 工具的描述"保存内容到本地文件"最为匹配。
    *   **如何设置参数？** 模型会从用户查询中提取执行工具所需的具体参数值，如 `file_name` 和 `content`。
4.  **结构化输出**: 如果模型决定调用工具，它会返回一个结构化的请求（如 JSON 对象），明确指出要调用的工具名称和填充好的参数。

这种方式类似于 **ReAct (Reasoning and Acting)** 框架，LLM 不仅是语言生成器，更是一个能够规划和执行动作的"大脑"。


### `@mcp.resource` 的选择策略：应用驱动与智能检索

`@mcp.resource` 的选择由**客户端应用**驱动，这一点与模型驱动的 `@mcp.tool` 有本质区别。它为 Agent 提供了获取精确上下文的能力。然而，为了实现更智能的交互，客户端在决定"调用哪个资源"时，可以采用从简单到复杂的多种策略。

要理解高级策略，我们首先需要区分 `@mcp.resource` 和 **RAG (Retrieval-Augmented Generation, 检索增强生成)**。前者是一个**标准化的数据接口**，而后者是一个**智能检索的应用模式**。

- **RAG** 是一种用于回答**开放式、未知问题**的复杂模式。它的核心思想是：当用户提出一个问题时（例如"我们的新产品有什么竞争优势？"），系统并不知道答案具体在哪份文档里。因此，RAG 会执行一个复杂的多阶段流程，通常包括：
    1.  **检索 (Retrieval)**: 将用户问题通过**嵌入 (Embedding)** 模型转换为向量，然后在预先建立的**向量数据库**中进行语义搜索，找出最相关的信息片段。
    2.  **重排 (Re-ranking)**: (可选) 使用更精密的模型对初步检索到的结果进行**重排序 (Rerank)**，将最相关的内容置于顶端。
    3.  **增强生成 (Augmented Generation)**: 将经过筛选和排序的信息片段与原始问题一起交给 LLM，让其"阅读理解"后生成最终答案。

下表总结了 `@mcp.resource` 作为接口与 RAG 作为模式在应用上的区别与联系：

| 特性 | `@mcp.resource` (作为接口) | RAG (作为模式) |
| :--- | :--- | :--- |
| **决策来源** | **客户端应用逻辑**决定**何时**调用以及调用**哪个**URI。 | **用户自然语言查询**驱动，通过语义相似度匹配。 |
| **数据访问** | **直接、精确** (通过指定URI调用 `read_resource`)。 | **间接、模糊** (通过语义相似度搜索向量数据库)。 |
| **核心用途** | 提供一个**标准化的上下文数据源**，如文件、API 端点等。 | 从**海量、非结构化**知识中发现并合成答案。 |
| **实现方式** | 在服务端定义一个函数，并用 `@mcp.resource` 装饰。 | 客户端或独立服务实现"检索-重排-生成"的完整流程。 |
| **结合点** | **客户端的路由逻辑**可以使用 RAG 模式来**智能地决定调用哪个 `@mcp.resource`**。 |

`@mcp.resource` 本身不包含检索逻辑，但它为客户端实现智能检索提供了完美的构件。客户端的 `add_relevant_resources` 函数正是这种结合的绝佳示例。根据复杂度和智能化程度，客户端可以选择以下三种实现策略：

#### 策略一：简单规则匹配
最直接的方法是使用关键词或正则表达式进行匹配。

*   **实现**: 在 `client.py` 的初始版本中，`add_relevant_resources` 通过一个硬编码的 `keywords_map` 实现。
    ```python
    # client.py (简单实现)
    keywords_map = {
        "qwen2": ["qwen-doc://qwen2.md"],
        "千问": ["qwen-doc://qwen1.5.md", "qwen-doc://qwen2.md"],
    }
    # ... 如果用户问题包含 "qwen2"，则读取 "qwen-doc://qwen2.md" ...
    ```
*   **优缺点**: 这种方式简单、快速，但扩展性差，无法理解同义词或更复杂的表达。

#### 策略二：LLM 路由 (LLM Router)
这是一种将决策权再次交给模型，但在复杂度和成本上介于关键词匹配和完整 RAG 之间的方案。其核心思路是进行一次廉价、快速的 LLM 调用，让其充当"路由"角色。
    
*   **方法**:
    1.  客户端获取所有可用的 `@mcp.resource` 列表，提取每个资源的 `name` 和 `description`。
    2.  将用户问题与这份资源清单组合成一个简单的提示，例如："用户问：'{user_question}'。根据以下可用文档列表，哪个或哪些文档最有助于回答该问题？请仅返回最相关文档的 URI。可用文档：\n - {resource_1_uri}: {resource_1_description}\n - {resource_2_uri}: {resource_2_description}\n..."
    3.  向一个高速、低成本的 LLM（例如专门用于分类或路由的微调模型）发起这个请求。
    4.  LLM 返回它认为最相关的资源 URI，客户端再通过 `read_resource` 精确调用。
    
*   **优缺点**:
    *   **优点**: 比关键词匹配智能得多，能理解一定的语义和意图。比完整的 Embedding 方案实现更简单，无需构建和维护向量数据库。
    *   **缺点**: 增加了一次额外的 LLM 调用开销和延迟。对于非常微妙的语义差异，其准确性可能不如 Embedding 检索。
        
> [!NOTE]
> 经网络检索，这种利用 LLM 进行路由决策的模式，是当前如 **Cursor** 等先进的本地代码知识库辅助工具所采用的高级策略之一。它们通常将其与手动指定（`@`符号）、规则匹配和更复杂的语义检索（Reranking）相结合，构成一个分层的上下文管理系统，以兼顾效率、精确性和智能化。

#### 策略三：Embedding 语义检索 (Mini-RAG)
为了让资源选择更加智能，我们可以将 `add_relevant_resources` 函数升级为基于 Embedding 的语义检索，这实际上是在客户端实现了一个**针对可用资源的 "Mini-RAG"**：

*   **步骤 1: 离线索引 (Offline Indexing)**
    客户端启动时，获取所有可用 `@mcp.resource` 的列表。对每个资源的 `name` 和 `description` 进行嵌入（Embedding），生成代表其语义的向量，并构建一个内存中的向量索引。

*   **步骤 2: 实时检索 (Real-time Retrieval)**
    当 `add_relevant_resources` 被调用时，它会接收用户的问题，并使用相同的模型将其嵌入为查询向量。

*   **步骤 3: 语义匹配与决策 (Semantic Match & Decision)**
    在向量索引中，通过计算查询向量与所有资源描述向量的余弦相似度，找到最匹配的 Top-K 个资源。

*   **步骤 4: 精确调用 (Precise Invocation)**
    一旦确定了最相关的资源（例如 `qwen-doc://qwen2.md`），客户端就使用其精确的 URI 调用 `session.read_resource` 来获取内容，并将其注入到上下文中。

这种演进后的函数看起来像这样：
```python
# client.py (概念上的高级实现)
# self.resource_vectors: 预先计算好的资源描述向量
# self.resources: 资源的完整信息列表

async def add_relevant_resources_with_embeddings(self, user_question: str) -> str:
    """通过 Embedding 语义搜索，查找并添加最相关的资源到上下文中。"""
    
    # 1. 将用户问题嵌入为向量
    question_vector = self.embedding_model.embed(user_question)

    # 2. 计算与所有可用资源描述的相似度
    similarities = cosine_similarity(question_vector, self.resource_vectors)

    # 3. 找到最相关的资源 (例如，选择相似度最高的)
    top_resource_index = find_most_similar(similarities)
    if similarities[top_resource_index] > THRESHOLD:
        resource_uri = self.resources[top_resource_index]['uri']
        
        # 4. 读取资源并构建上下文 (后续逻辑与原函数类似)
        resource_content = self.resources_dict[resource_uri]
        return user_question + f"\n\n相关信息 ({resource_uri}):\n\n{resource_content}"
        
    return user_question
```

**总结**

-   **`@mcp.resource`** 提供了一个**接口**，让服务端能力以标准化的方式暴露。
-   **RAG** 是一个**模式**，用于从大型知识库中智能检索信息。

最佳实践是将两者结合：在客户端实现一个 RAG 或语义搜索逻辑，用它来**智能地选择**要调用哪个 `@mcp.resource`。这使得 Agent 既能利用语义理解的灵活性，又能享受 MCP 带来的标准化和模块化优势。


### `@mcp.prompt` 的选择策略：分类、语义或模型路由

提示模板的选择相对灵活，可以根据应用的复杂度和成本要求，采用不同的策略：

1.  **简单规则匹配**:
    *   **方法**: 使用正则表达式（Regex）或关键词匹配。例如，如果用户输入包含"周报"或"总结"，就选择 `generate_weekly_report` 模板。
    *   **优缺点**: 实现简单、速度快、成本低。但规则僵化，难以应对多变的自然语言表达，扩展性差。

2.  **语义相似度搜索**:
    *   **方法**: 这是一种更先进的方案，类似于 RAG 的检索。将每个提示模板的 `description` 或代表性用法进行嵌入，存为向量。当用户查询到来时，将其嵌入后，通过计算向量相似度来找到最匹配的提示模板。
    *   **优缺点**: 能够理解语义相关性而非表面文字匹配，效果远好于规则匹配，是性能和成本之间的良好平衡。

3.  **LLM 路由 (Router)**:
    *   **方法**: 将选择权再次交给 LLM。设计一个专门的"路由"步骤，向一个（通常是更小、更快的）LLM 发起请求，请求中包含用户的查询和所有可用提示模板的描述列表，让模型直接决定应该使用哪一个模板。
    *   **优缺点**: 最灵活、最智能的方式，能够理解非常复杂和微妙的用户意图。但缺点是会增加额外的 LLM 调用开销和响应延迟。

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

