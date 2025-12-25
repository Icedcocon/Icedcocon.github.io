# NIM 快速开始

## NVCR 账号注册与镜像拉取

参考以下[地址](https://docs.nvidia.com/ngc/latest/ngc-catalog-user-guide.html#account-signup)注册NVCR账号
获取API后登录

```bash
root@ubuntu-gpu:~# docker login nvcr.io
Username: $oauthtoken
Password:

WARNING! Your credentials are stored unencrypted in '/root/.docker/config.json'.
Configure a credential helper to remove this warning. See
https://docs.docker.com/go/credential-store/

Login Succeeded
```

## 一、NIM 概述

### 1. 引言

NVIDIA NIM 是 NVIDIA AI Enterprise 的一部分，是**一套易于使用的微服务**，旨在加速企业中生成式 AI 的部署。 它支持广泛的 AI 模型，包括 NVIDIA AI 基础模型和自定义模型，并利用行业标准 API，确保在本地或云端实现无缝、可扩展的 AI 推理。

NVIDIA NIM 为大型语言模型 (LLM) 提供预构建容器，在**自托管环境中部署大型语言模型** (LLM)，并**提供行业标准 API**，可用于开发聊天机器人、内容分析器或任何需要理解和生成人类语言的应用程序。 每个 NIM 都包含一个容器和一个模型，并使用**适用于所有 NVIDIA GPU 的 CUDA 加速运行时**，并针对多种配置提供特殊优化。无论是在本地还是在云端，NIM 都是实现大规模加速生成式 AI 推理的最快方式。

NIM 使 IT 和 DevOps 团队能够轻松地在自己的托管环境中自托管大型语言模型 (LLM)，同时为开发人员提供行业标准 API，让他们能够构建强大的副驾驶、聊天机器人和 AI 助手，从而改变他们的业务。利用 NVIDIA 的尖端 GPU 加速和可扩展部署，NIM 提供了无与伦比性能的最快推理路径。

### 2. NIM 分类选项

无论您的首要任务是使用特定的 LLM 实现最佳性能，还是拥有运行多种不同模型的多功能性，NIM 都能简化安全企业环境中的自托管。适用于大型语言模型 (LLM) 的 NVIDIA NIM 提供两种选择：

- **多 LLM 通用 NIM** （**Multi-LLM compatible NIM**）：一个容器即可部署多种的模型，提供最大的灵活性。
- **单 LLM 专用 NIM** （**LLM-specific NIM**）：每个容器都专注于单个模型或模型系列，提供最佳性能。

#### 如何选择 NIM 选项？

| NIM 容器选项 | 多 LLM 通用 NIM 容器                                       | 单 LLM 专用 NIM 容器                                  |
| -------- | ----------------------------------------------------- | ------------------------------------------------ |
| **推荐场景** | 当 NVIDIA 尚未为您要部署的模型提供专用容器时                            | 当 NVIDIA 为您要部署的模型提供专用容器时                         |
| **性能**   | 提供良好的基线性能，具有为支持的模型即时构建优化引擎以获得更高吞吐量的灵活性                | 为**特定模型/GPU 组合提供预构建**的优化引擎，为支持的配置提供开箱即用的最大性能     |
| **灵活性**  | **最大灵活性**。支持来自各种来源（NGC、HuggingFace、本地磁盘）的广泛模型、格式和量化类型 | **每个容器仅限于单个模型**                                  |
| **安全性**  | 您负责验证来自非 NVIDIA 位置的模型的安全性和完整性                         | NVIDIA 策划、安全扫描并提供模型和容器                           |
| **支持**   | NVIDIA AI Enterprise 为 NIM 容器提供支持。对模型本身的支持可能有所不同      | NVIDIA AI Enterprise 为 NIM 容器提供支持。对模型本身的支持可能有所不同 |

### 3. 高性能特性

NIM 抽象了模型推理内部机制，如执行引擎和运行时操作。无论是使用 TRT-LLM、vLLM 还是其他技术，它们都是可用的最高性能选项。NIM 提供以下高性能特性：

- **可扩展部署**：性能卓越，可以轻松无缝地从少数用户扩展到数百万用户
- **高级语言模型支持**：为各种尖端 LLM 架构提供预构建的优化引擎
- **灵活集成**：轻松将微服务集成到现有工作流程和应用程序中。为开发人员提供与 OpenAI API 兼容的编程模型和用于附加功能的自定义 NVIDIA 扩展
- **企业级安全**：通过使用 safetensors、持续监控和修补堆栈中的 CVE 以及进行内部渗透测试来强调安全性

### 4. 应用场景

NIM 的潜在应用广泛，跨越各个行业和用例：

- **聊天机器人和虚拟助手**：为机器人提供类人的语言理解和响应能力
- **内容生成和摘要**：轻松生成高质量内容或将冗长文章提炼成简洁摘要
- **情感分析**：实时了解用户情感，推动更好的业务决策
- **语言翻译**：通过高效准确的翻译服务打破语言障碍
- **更多应用**：NIM 的潜在应用非常广泛，涵盖各个行业和用例

### 5. 架构概述

#### 多 LLM NIM 架构

多 LLM NIM 容器支持广泛的模型架构。它包含一个可以在任何具有足够内存的 NVIDIA GPU 上运行的运行时，某些模型和 GPU 组合提供额外的优化。

通过 NVIDIA NGC 目录将此 NIM 下载为 NGC 容器镜像。与容器一起使用的模型可以来自 NGC、Hugging Face 或本地磁盘，为您在部署和管理模型方面提供灵活性。

> **安全警告**：NVIDIA 无法保证托管在非 NVIDIA 系统（如 HuggingFace）上的任何模型的安全性。恶意或不安全的模型可能导致严重的安全风险，包括完全远程代码执行。我们强烈建议在尝试加载任何非 NVIDIA 提供的模型之前，手动验证其安全性。


## 二、多 LLM NIM 部署

### 1. 概述

多 LLM NIM 容器是一个通用的推理容器，支持广泛的文本生成模型架构。它提供了灵活性，允许您部署来自不同来源的各种模型，包括 NGC、Hugging Face 和本地磁盘。

### 2. 支持的推理引擎

多 LLM NIM 支持三种主要的推理引擎，每种引擎都有其特定的优势：

| 特性 | vLLM | TensorRT-LLM | SGLang |
|------|------|--------------|--------|
| **吞吐量** | 高 | 最高 | 中等 |
| **延迟** | 中等 | 最低 | 中等 |
| **内存效率** | 高 | 最高 | 中等 |
| **模型支持** | 广泛 | 广泛 | 有限 |
| **结构化输出** | 基础 | 基础 | 高级 |
| **易用性** | 高 | 中等 | 高 |

### 3. 支持的模型架构

多 LLM NIM 支持以下模型架构（按官方架构名称简化列出，列表完整）：

- BartForConditionalGeneration
- BloomForCausalLM
- ChatGLMModel
- DeciLMForCausalLM
- DeepseekV2ForCausalLM
- DeepseekV3ForCausalLM
- FalconForCausalLM
- FalconMambaForCausalLM
- GemmaForCausalLM
- Gemma2ForCausalLM
- GlmForCausalLM
- GPTBigCodeForCausalLM
- GPT2LMHeadModel
- GPTNeoXForCausalLM
- GraniteForCausalLM
- GraniteMoeForCausalLM
- GritLM
- InternLM2ForCausalLM
- InternLM3ForCausalLM
- JambaForCausalLM
- LlamaForCausalLLM
- MambaForCausalLM
- MistralForCausalLM
- MolmoForCausalLM
- Olmo2ForCausalLM
- OlmoeForCausalLM
- PhiMoEForCausalLM
- Phi3ForCausalLM
- Phi3SmallForCausalLM
- QWenLMHeadModel
- Qwen2ForCausalLM
- Qwen2MoeForCausalLM
- RWForCausalLM
- SolarForCausalLM

### 4. 支持的模型格式

- **Safetensors**：推荐格式，提供更好的安全性
- **PyTorch (.bin)**：传统 PyTorch 格式
- **GGUF**：量化模型格式

### 5. 量化支持

- **FP16**：半精度浮点，平衡性能和精度
- **INT8**：8位整数量化，显著减少内存使用
- **INT4**：4位量化，最大化内存效率
- **GPTQ**：后训练量化方法
- **AWQ**：激活感知权重量化

### 6. 功能特性

- **基础模型支持**：支持各种预训练语言模型
- **LoRA 适配器**：支持低秩适应微调
- **函数调用**：支持工具使用和函数调用
- **引导解码**：支持约束生成和格式化输出
- **动态批处理**：自动优化批处理大小
- **KV 缓存优化**：高效的注意力缓存管理
- **多 GPU 支持**：支持模型并行和张量并行
- **流式输出**：支持实时流式响应

### 7. 部署指南（对齐官方手册，高密度版）

#### 7.1 先决条件与登录
- 安装 Docker（含 NVIDIA Container Toolkit），具备可用的 NVIDIA GPU（计算能力 ≥ 7.0，bfloat16 需 ≥ 8.0）。
- 申请并导出 `NGC_API_KEY`，用于拉取 NIM 镜像与下载模型资产。

##### 安装 NGC CLI
- 用于列出、下载 NIM 镜像与模型资产。访问官方页面下载对应系统的安装包：https://org.ngc.nvidia.com/setup/installers/cli
- 将 `ngc` 放入系统 `PATH` 并验证版本后，配置 API Key。

Linux：
```bash
wget --content-disposition https://api.ngc.nvidia.com/v2/resources/nvidia/ngc-apps/ngc_cli/versions/4.5.2/files/ngccli_linux.zip -O ngccli_linux.zip && unzip ngccli_linux.zip
chmod u+x ngc-cli/ngc
sudo mv ngc /usr/local/bin/ && cd /usr/local/bin/
echo "export PATH=\"\$PATH:$(pwd)/ngc-cli\"" >> ~/.bash_profile && source ~/.bash_profile
ngc --version
```

配置 API Key（非交互方式）：
```bash
ngc config set apiKey "$NGC_API_KEY"
```
或交互式：`ngc config set`。

```bash
echo "$NGC_API_KEY" | docker login nvcr.io --username '$oauthtoken' --password-stdin
ngc registry image list --format_type csv nvcr.io/nim/*  # 列出可用 NIM 仓库与版本
```

#### 7.2 多 LLM 通用容器启动（提供自选模型）
- 当暂无 LLM 专用容器时，使用多 LLM 通用容器；需指定模型来源。
```bash

# Set HF_TOKEN for downloading HuggingFace repository
export HF_TOKEN=hf_xxxxxx

# Choose a HuggingFace model 
export NIM_MODEL_NAME=hf://google/gemma-3-27b-it

# Choose a served model name 
export NIM_SERVED_MODEL_NAME=google/gemma-3-27b-it

# Choose a path on your system to cache the downloaded models
export LOCAL_NIM_CACHE=~/.cache/nim
mkdir -p "$LOCAL_NIM_CACHE"

# Add write permissions to the NIM cache for downloading model assets
chmod -R a+w "$LOCAL_NIM_CACHE"

# Start the LLM NIM
docker run -it --rm --name=LLM-NIM \
  --runtime=nvidia \
  --gpus all \
  --shm-size=16GB \
  -e HF_TOKEN=$HF_TOKEN \
  -e NIM_MODEL_NAME=$NIM_MODEL_NAME \
  -e NIM_SERVED_MODEL_NAME=$NIM_SERVED_MODEL_NAME \
  -v "$LOCAL_NIM_CACHE:/opt/nim/.cache" \
  -u $(id -u) \
  -p 8000:8000 \
  nvcr.io/nim/nvidia/llm-nim:latest
   
# docker run -it --rm --name=LLM-NIM   --runtime=nvidia   --gpus '"device=1"'   --shm-size=16GB   -e HF_TOKEN=$HF_TOKEN   -e NIM_MODEL_NAME=/mnt/models   -e NIM_SERVED_MODEL_NAME=qwen3   -e NIM_MAX_IMAGES_PER_PROMPT=0   -v "/root/Models/NIM-cache:/opt/nim/.cache" -v "/root/Models/LLMs/Qwen3-1.7B:/mnt/models"  -u $(id -u)   -p 8000:8000   nvcr.io/nim/nvidia/llm-nim:latest
```
- 也可改为本地模型目录：`-e NIM_MODEL_NAME=/path/to/model_dir`（需合法结构）。

### 章节参考

https://docs.nvidia.com/nim/large-language-models/latest/introduction.html
https://docs.nvidia.com/nim/large-language-models/latest/supported-architectures.html


## 三、 单 LLM NIM 部署

### 1. 概述

单 LLM 专用 NIM（LLM-specific NIM）为单一模型或模型家族提供预构建、深度优化的推理容器。相比通用容器，它在指定的模型与 GPU 组合上提供更高的吞吐与更低的延迟，并简化了部署（无需再指定外部模型来源）。

关键特性：
- 针对特定模型/GPU 的预构建引擎（优先选择优化 profile，自动 fallback 到通用 profile 如 `local_build` 或 `vllm`）
- 统一 API（OpenAI 兼容的 `/v1/chat/completions` 等），开箱即用
- 企业级安全与维护（由 NVIDIA 扫描与策划的镜像与模型）

参考模型列表与版本能力（官方持续更新）：
- 支持模型、版本、功能（LoRA、工具调用、并行工具调用、Suffix 支持）详见官方页面 [0]
- 不同模型页面可查看硬件需求（显存与 GPU 代际要求） [0]

注：[0] Supported Models for NVIDIA NIM for LLMs — NVIDIA NIM for Large Language Models (LLMs)，https://docs.nvidia.com/nim/large-language-models/latest/supported-models.html

### 2. 选择镜像与版本（Profile 选择）

单 LLM NIM 镜像命名规则通常为：
`nvcr.io/nim/<organization>/<model-id>:<version-or-profile>`

说明：
- `<organization>/<model-id>` 与官方“Supported Models”表中的“Organization/Model ID”一致（例如 `meta/llama-3.1-8b-instruct`、`deepseek-ai/deepseek-r1` 等）[0]
- `<version-or-profile>` 使用表格中列出的支持版本或指定 profile（例如 `1.13.1`、`1.7.3`、或某些带 `-RTX` 的 profile）[0]
- 若当前硬件不满足优化 profile，容器会自动选择通用 profile（如 `local_build` 或 `vllm`）以最大兼容性 [0]

### 3. 快速部署示例（单模型容器）

以下示例展示如何选择镜像并运行容器。请根据实际硬件与期望功能选择对应版本。

示例 A：Llama 3.1 8B Instruct（支持 LoRA、工具调用、并行工具调用）[0]
```bash
# 选择镜像（以官方列出的支持版本为准）
export NIM_IMAGE=nvcr.io/nim/meta/llama-3.1-8b-instruct:1.13.1

# 启动容器（仅文本；关闭图像输入）
docker run -it --rm --name=llm-nim-llama31-8b \
  --runtime=nvidia --gpus all \
  --shm-size=16GB \
  -e NGC_API_KEY=$NGC_API_KEY \
  -u $(id -u) \
  -p 8000:8000 \
  nvcr.io/nim/meta/llama-3.1-8b-instruct:1.13.1
```
### 4. 参考


https://docs.nvidia.com/nim/large-language-models/latest/getting-started.html

## 其他NIM部署

### 1.  Embedding NIM

```bash
export NGC_API_KEY=<PASTE_API_KEY_HERE>
export LOCAL_NIM_CACHE=~/.cache/nim
mkdir -p "$LOCAL_NIM_CACHE"
docker run -it --rm \
    --gpus all \
    --shm-size=16GB \
    -e NGC_API_KEY \
    -v "$LOCAL_NIM_CACHE:/opt/nim/.cache" \
    -u $(id -u) \
    -p 8000:8000 \
    nvcr.io/nim/snowflake/arctic-embed-l:1.0.1

```
### 1.  bge-m3  NIM

```bash
export NGC_API_KEY=<PASTE_API_KEY_HERE>
export LOCAL_NIM_CACHE=~/.cache/nim
mkdir -p "$LOCAL_NIM_CACHE"
docker run -it --rm \
    --gpus all \
    --shm-size=16GB \
    -e NGC_API_KEY \
    -v "$LOCAL_NIM_CACHE:/opt/nim/.cache" \
    -u $(id -u) \
    -p 8000:8000 \
    nvcr.io/nim/baai/bge-m3:latest

```

## 四、 API 调用（OpenAI 兼容）

单 LLM 容器默认提供 OpenAI 兼容的接口。常用示例：

Chat Completions：
```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta/llama-3.1-8b-instruct",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "用三句话解释什么是NIM。"}
    ],
    "temperature": 0.3,
    "stream": false
  }'
```

流式输出：
```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta/llama-3.1-8b-instruct",
    "messages": [
      {"role": "user", "content": "用要点列出NIM的优势。"}
    ],
    "stream": true
  }'
```

工具调用（如模型支持 Tools/Function Calling 能力）[0]：
```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta/llama-3.1-8b-instruct",
    "messages": [
      {"role": "user", "content": "查询上海当前天气并给出穿衣建议。"}
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "get_weather",
          "description": "Get current weather by city name.",
          "parameters": {
            "type": "object",
            "properties": {
              "city": {"type": "string"}
            },
            "required": ["city"]
          }
        }
      }
    ],
    "tool_choice": "auto"
  }'
```

## 五、镜像启动脚本

### 1. 单 LLM 专用镜像
#### 1.1 入口脚本
-  entrypoints:
	- `/opt/nvidia/nvidia_entrypoint.sh`
- cmd:
	- `/opt/nim/start_server.sh`

#### 1.2  `/opt/nvidia/nvidia_entrypoint.sh`

本脚本用于在容器或自动化环境中作为**统一入口点（entrypoint）**执行。  
它的主要功能包括：

1. **自动加载并按字母顺序执行入口片段文件**（位于 `entrypoint.d/` 目录下）；
    
2. 支持 `.txt` 文件直接输出、`.sh` 文件按顺序执行；
    
3. 当无命令参数时，自动进入交互式 Bash；
    
4. 当提供命令参数时，将该命令作为主进程执行（使用 `exec` 替换当前进程）。


```bash
#!/bin/bash

# Gather parts in alpha order
# 启用 Bash 的 **扩展通配符** 功能（`extglob`）和 **空匹配返回空数组** 功能
shopt -s nullglob extglob
# 获取当前脚本所在目录的绝对路径。
_SCRIPT_DIR="$(dirname "$(readlink -f "${BASH_SOURCE[0]}")")"
# 收集 `entrypoint.d/` 下的所有 `.txt` 或 `.sh` 文件，按字母顺序排序存入数组。
declare -a _PARTS=( "${_SCRIPT_DIR}/entrypoint.d"/*@(.txt|.sh) )
shopt -u nullglob extglob

# 辅助函数：打印指定字符若干次（常用于边框线）。
print_repeats() {
  local -r char="$1" count="$2"
  local i
  for ((i=1; i<=$count; i++)); do echo -n "$char"; done
  echo
}
# 辅助函数：以指定字符生成一个包围文字的横幅样式输出，用于美观的日志输出。
print_banner_text() {
  # $1: Banner char
  # $2: Text
  local banner_char=$1
  local -r text="$2"
  local pad="${banner_char}${banner_char}"
  print_repeats "${banner_char}" $((${#text} + 6))
  echo "${pad} ${text} ${pad}"
  print_repeats "${banner_char}" $((${#text} + 6))
}

# Execute the entrypoint 
# 遍历 `_PARTS` 数组中的文件；若为 `.txt` 文件：直接输出其内容；若为 `.sh` 文件：使用 `source` 执行（在当前进程中执行，而非子进程）；这样能共享变量、函数、环境配置。
for _file in "${_PARTS[@]}"; do
  case "${_file}" in
    *.txt) cat "${_file}";;
    *.sh)  source "${_file}";;
  esac
done

echo

# This script can either be a wrapper around arbitrary command lines,
# or it will simply exec bash if no arguments were given
# 无参数启动时：进入交互式 Bash；
# 有参数启动时：用 `exec` 替换当前进程执行指定命令。
if [[ $# -eq 0 ]]; then
  exec "/bin/bash"
else
  exec "$@"
fi
```

##### 1.3  `/opt/nim/start_server.sh`
该脚本为 **NIM（NVIDIA Inference Microservice）** 的启动入口，用于在容器或推理节点环境中：

- 初始化日志系统；
    
- 配置 VLLM / TensorRT-LLM / Uvicorn 的日志等级；
    
- 根据运行模式选择日志输出格式（可读文本或 JSONL）；
    
- 启动主推理服务进程；
    
- 处理 SIGINT/SIGTERM 信号，实现优雅关闭（graceful shutdown）。

```bash
#!/bin/bash

# Gather parts in alpha order
shopt -s nullglob extglob
_SCRIPT_DIR="$(dirname "$(readlink -f "${BASH_SOURCE[0]}")")"
declare -a _PARTS=( "${_SCRIPT_DIR}/entrypoint.d"/*@(.txt|.sh) )
shopt -u nullglob extglob

print_repeats() {
  local -r char="$1" count="$2"
  local i
  for ((i=1; i<=$count; i++)); do echo -n "$char"; done
  echo
}

print_banner_text() {
  # $1: Banner char
  # $2: Text
  local banner_char=$1
  local -r text="$2"
  local pad="${banner_char}${banner_char}"
  print_repeats "${banner_char}" $((${#text} + 6))
  echo "${pad} ${text} ${pad}"
  print_repeats "${banner_char}" $((${#text} + 6))
}

# Execute the entrypoint parts
for _file in "${_PARTS[@]}"; do
  case "${_file}" in
    *.txt) cat "${_file}";;
    *.sh)  source "${_file}";;
  esac
done

echo

# This script can either be a wrapper around arbitrary command lines,
# or it will simply exec bash if no arguments were given
if [[ $# -eq 0 ]]; then
  exec "/bin/bash"
else
  exec "$@"
fi
root@c7a38a1d075a:/# cat /opt/nim/start_server.sh
#!/bin/bash
set -u
export VLLM_CONFIGURE_LOGGING=1

NIM_LOG_LEVEL="${NIM_LOG_LEVEL:-DEFAULT}"

case "${NIM_LOG_LEVEL}" in
  "TRACE")
    export TLLM_LOG_LEVEL="TRACE"
    export VLLM_NVEXT_LOG_LEVEL="DEBUG"
    export UVICORN_LOG_LEVEL="trace"
    ;;
  "DEBUG")
    export TLLM_LOG_LEVEL="DEBUG"
    export VLLM_NVEXT_LOG_LEVEL="DEBUG"
    export UVICORN_LOG_LEVEL="debug"
    ;;
  "INFO")
    export TLLM_LOG_LEVEL="INFO"
    export VLLM_NVEXT_LOG_LEVEL="INFO"
    export UVICORN_LOG_LEVEL="info"
    ;;
  "DEFAULT")
    export TLLM_LOG_LEVEL="ERROR"
    export VLLM_NVEXT_LOG_LEVEL="INFO"
    export UVICORN_LOG_LEVEL="info"
    ;;
  "WARNING")
    export TLLM_LOG_LEVEL="WARNING"
    export VLLM_NVEXT_LOG_LEVEL="WARNING"
    export UVICORN_LOG_LEVEL="warning"
    ;;
  "ERROR")
    export TLLM_LOG_LEVEL="ERROR"
    export VLLM_NVEXT_LOG_LEVEL="ERROR"
    export UVICORN_LOG_LEVEL="error"
    ;;
  "CRITICAL")
    export TLLM_LOG_LEVEL="ERROR"
    export VLLM_NVEXT_LOG_LEVEL="CRITICAL"
    export UVICORN_LOG_LEVEL="critical"
    ;;
  *)
    echo "Unsupported value ('${NIM_LOG_LEVEL}') of NIM_LOG_LEVEL. Supported values are: TRACE, DEBUG, INFO, DEFAULT, WARNING, ERROR, CRITICAL." >&2
    exit 1
    ;;
esac

# 优雅关闭（Graceful Shutdown）机制
# 使用 `pgrep -x "pt_main_thread"` 查找主推理线程进程 PID；
#  1. 向子进程发送 `SIGTERM`；
#  2. 等待 5 秒；
#  3. 若仍存活，则发送 `SIGKILL` 强制终止；
#  4. 使用 `wait` 等待清理；
#  最后 `exit 0` 确保容器正常退出。
function terminate() {
  NIM_LLM_PID=$(pgrep -x "pt_main_thread")
  echo "Captured NIM_LLM_PID: ${NIM_LLM_PID:-0}"
  echo "$0 received SIGTERM, attempting graceful shutdown..."
  if [ -z "$NIM_LLM_PID" ]
  then
    echo "$0 has no NIM_LLM process to terminate"
    exit 0
  fi
  # Attempt graceful shutdown.
  echo "Attempting to shutdown process ${NIM_LLM_PID:-0}"
  kill -TERM "$NIM_LLM_PID"
  sleep 5 # Wait 5 seconds for graceful shutdown
  # Check if the child process is still running by sending signal 0.
  if kill -0 "$NIM_LLM_PID" 2>/dev/null; then
    echo "$0 graceful shutdown failed, forcefully killing process..."
    kill -9 "$NIM_LLM_PID"
  fi
  wait "$NIM_LLM_PID"
  exit 0
}
# Trap SIGINT or SIGTERM and call the terminate function.
trap terminate SIGINT SIGTERM

jsonl_config="/etc/nim/config/python_jsonl_logging_config.json"
readable_config="/etc/nim/config/python_readable_logging_config.json"
NIM_JSONL_LOGGING="${NIM_JSONL_LOGGING:-0}"
if [ "${NIM_JSONL_LOGGING}" = 1 ]; then
  export VLLM_LOGGING_CONFIG_PATH="${jsonl_config}"
  export VLLM_NVEXT_LOGGING_CONFIG_PATH="${jsonl_config}"
  python3 -m nim_llm_sdk.entrypoints.launch |& python3 -m nim_llm_sdk.logging.pack_all_logs_into_json
elif [ "${NIM_JSONL_LOGGING}" = 0 ]; then
  export VLLM_LOGGING_CONFIG_PATH="${readable_config}"
  export VLLM_NVEXT_LOGGING_CONFIG_PATH="${readable_config}"
  python3 -m nim_llm_sdk.entrypoints.launch
else
  echo "ERROR: Unsupported value ('${NIM_JSONL_LOGGING}') of NIM_JSONL_LOGGING env variable. Supported values are 0 and 1" >&2
  exit 1
fi
```

NIM 服务启动指令为 `python3 -m nim_llm_sdk.entrypoints.launch`
#### 1.4 `/opt/nim/llm/nim_llm_sdk/`

```bash
root@c7a38a1d075a:/opt/nim/llm/nim_llm_sdk# tree .
.
|-- __init__.py
|-- constants.py
|-- engine
|   |-- __init__.py
|   |-- async_trtllm_engine.py
|   |-- async_trtllm_engine_factory.py
|   |-- metrics.py
|   |-- trtllm_errors.py
|   `-- trtllm_model_runner.py
|-- entrypoints
|   |-- __init__.py
|   |-- args.py
|   |-- launch.py
|   |-- llamastack
|   |   |-- protocol.py
|   |   |-- serving_chat.py
|   |   |-- serving_completion.py
|   |   `-- serving_engine.py
|   `-- openai
|       |-- __init__.py
|       |-- api_extensions.py
|       `-- api_server.py
|-- env_utils.py
|-- envs.py
|-- hub
|   |-- __init__.py
|   |-- hardware_inspect.py
|   |-- info.py
|   |-- local_cache_manager
|   |   `-- local_cache.py
|   |-- ngc_download.py
|   |-- ngc_injector.py
|   |-- ngc_profile.py
|   |-- pre_download.py
|   |-- utils.py
|   `-- workspace.py
|-- logger.py
|-- logging
|   |-- __init__.py
|   |-- const.py
|   |-- json_formatter.py
|   `-- pack_all_logs_into_json.py
|-- lora
|   |-- __init__.py
|   `-- source.py
|-- mem_estimators
|   |-- __init__.py
|   |-- base.py
|   |-- mem_estimator.py
|   `-- utils.py
|-- model_specific_modules
|   |-- README.md
|   |-- __init__.py
|   `-- dynamic_module_loader.py
|-- perf
|   |-- __init__.py
|   |-- benchmark.py
|   |-- llm_inputs
|   |   |-- __init__.py
|   |   |-- farewell.txt
|   |   |-- genai_default_synthetic_prompt_generator.py
|   |   |-- repeated_context_prompt_generator.py
|   |   |-- table.txt
|   |   |-- table_questions.txt
|   |   |-- tokenizer.py
|   |   `-- utils.py
|   |-- stats.py
|   `-- timer.py
|-- trtllm
|   |-- __init__.py
|   |-- configs.py
|   |-- parse_config.py
|   |-- request.py
|   |-- utils.py
|   `-- weight_manager
|       |-- __init__.py
|       |-- build_utils.py
|       `-- nim_optimize.py
`-- utils
    |-- __init__.py
    `-- caches.py
```
