# vLLM和SGLang开启Function Call


Function Call (或 Tool Calling) 是构建高级 AI Agent 和大语言模型（LLM）应用的核心能力。它赋予模型与外部工具和服务交互的能力，从而极大扩展了其应用边界。作为业界领先的高性能推理框架，vLLM 和 SGLang 均内置了对 Function Call 的支持。

本文旨在深入探讨在这两个框架中启用和配置 Function Call 的关键参数，并提供经过验证的实践指导。

## vLLM 中的 Tool Calling

vLLM 利用 `--tool-call-parser` 参数，根据所选模型，自动对工具调用请求进行格式化并解析模型响应。要启用模型的"自动工具选择"能力，还必须配合 `--enable-auto-tool-choice` 参数。

> [!TIP]
> vLLM 服务可以通过 `vllm serve` 命令启动，这是 `python -m vllm.entrypoints.openai.api_server` 的简化形式。

### `tool-call-parser` 参数详解

vLLM 提供了丰富的解析器选项，以下是各选项及其推荐适配的模型：

| Parser            | 适用模型                                                                    |
| ----------------- | ----------------------------------------------------------------------- |
| `hermes`          | Qwen 及 Nous Research Hermes 系列模型 (例如 `Nous-Hermes-2-Yi-34B`)            |
| `mistral`         | Mistral 原生支持函数调用的模型 (例如 `Mistral-7B-Instruct-v0.2`)                   |
| `llama3_json`     | Llama 3.1, 3.2, and 4 系列模型                                              |
| `phi4_mini_json`  | Microsoft `Phi-4-mini` 模型                                               |
| `granite`         | IBM `granite-8b-code-instruct`                                          |
| `granite-20b-fc`  | IBM `granite-20b-function-calling`                                      |
| `internlm`        | `InternLM2.5` 系列模型                                                      |
| `jamba`           | AI21 `Jamba-1.5` 系列模型                                                   |
| `xlam`            | Salesforce `Llama-xLAM` and `Qwen-xLAM` 系列模型                            |
| `deepseek_v3`     | 例如 `DeepSeek-V3-0324`, `DeepSeek-R1-0528`                                     |
| `pythonic`        | Llama 3.2, Llama 4, ToolACE 等系列模型 |

### 使用示例

启动 vLLM 服务时，为您的模型指定合适的解析器，并确保开启工具自动选择功能。

**示例 1: 使用 `hermes` 解析器**
```bash
vllm serve "NousResearch/Nous-Hermes-2-Yi-34B" \
    --enable-auto-tool-choice \
    --tool-call-parser hermes
```

**示例 2: Llama 3.1 模型使用 `llama3_json`**

对于 Llama 3.1 等特定模型，官方建议配合使用优化过的聊天模板以获得最佳性能。

```bash
vllm serve "meta-llama/Meta-Llama-3.1-8B-Instruct" \
    --enable-auto-tool-choice \
    --tool-call-parser llama3_json \
    --chat-template examples/tool_chat_template_llama3.1_json.jinja
```

## SGLang 中的 Tool Calling

SGLang 同样通过 `--tool-call-parser` 参数支持工具调用。尽管其官方文档目前不如 vLLM 详尽，我们仍可依据命名约定和社区共识进行有效配置。为获得更稳定的结果，官方推荐配合使用为工具调用优化的 `--chat-template`。

### `tool-call-parser` 参数推断

以下是 SGLang 中可用的 `tool-call-parser` 选项及其推断的适用模型：

| Parser        | 推断适用模型                                   |
| ------------- | ---------------------------------------------- |
| `qwen25`      | Qwen 2.5 系列模型                              |
| `mistral`     | Mistral 原生支持函数调用的模型               |
| `llama3`      | Llama 3 系列模型                               |
| `deepseekv3`  | 例如 `DeepSeek-V3-0324`, `DeepSeek-R1-0528`             |
| `pythonic`    | 适配通用 Pythonic tool calling 格式，适用范围广 |

> [!NOTE]
> 上述对应关系主要基于命名约定和社区实践。为确保最佳兼容性，强烈建议在实际部署前查阅最新的 SGLang 官方文档或通过实验进行验证。

> [!TIP]
> 当特定模型的专用解析器不可用时，`pythonic` 解析器通常是一个可靠的备选方案。它基于通用的 Pythonic tool-use 格式，具有较强的普适性。

### 使用示例

SGLang 服务的启动方式与 vLLM 相似。为保证最佳兼容性，建议配合使用官方提供的聊天模板。

```bash
python -m sglang.launch_server \
    --model-path "meta-llama/Meta-Llama-3-8B-Instruct" \
    --port 30000 \
    --tool-call-parser llama3 \
    --chat-template examples/chat_template/tool_chat_template_llama3.jinja
```
> [!NOTE]
> 上述 `tool_chat_template_llama3.jinja` 路径为示例。请根据您 SGLang 安装目录下的 `examples` 文件夹，确认并使用适用于目标模型的正确模板路径。

## 总结

精确配置 `--tool-call-parser` 并配合 `--enable-auto-tool-choice` (vLLM) 和 `--chat-template` (vLLM & SGLang) 是成功实现 Function Calling 的关键。vLLM 提供了清晰的解析器与模型映射，便于快速集成。SGLang 虽然文档尚在完善，但其规范的命名也为开发者提供了有效指引。为保证功能稳定与最优性能，请务必根据您选用的模型，选择最匹配的解析器和聊天模板。


