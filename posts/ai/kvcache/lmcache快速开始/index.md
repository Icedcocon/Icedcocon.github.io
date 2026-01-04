# LMCache快速开始


## 1. 项目简介

LMCache 是一个专为大语言模型（LLM）推理设计的高效 KV Cache 管理系统。它旨在通过优化 KV Cache 的存储与传输，解决 LLM 推理中的显存瓶颈和重复计算问题。LMCache 支持多种后端存储（如 CPU 内存、磁盘、网络存储），并提供灵活的共享机制，能够显著降低首字延迟（TTFT）并提高系统吞吐量。

其核心价值在于：
*   **降低延迟**：通过复用计算过的 KV Cache，减少 Prefill 阶段的计算时间。
*   **节省资源**：支持将 KV Cache 卸载到廉价存储（如 CPU 内存），释放宝贵的 GPU 显存。
*   **提升吞吐**：在分离式架构中优化资源分配，提升整体服务能力。

## 2. 核心功能解析

### 2.1 KV Cache Offloading (KV 缓存卸载)
允许将 KV Cache 从 GPU 显存移动到 CPU 内存或其他存储设备。
*   **适用场景**：
    *   请求共享相同前缀（如长 System Prompt、聊天历史）。
    *   GPU 显存不足以保存所有 KV Cache。
*   **收益**：降低 TTFT，减少 GPU 计算周期。

### 2.2 KV Cache Sharing (KV 缓存共享)
支持在不同 LLM 实例之间共享 KV Cache。
*   **适用场景**：
    *   系统运行多个 LLM 实例。
    *   共享相同前缀的请求被分发到不同实例。
*   **收益**：消除跨实例的冗余计算。

### 2.3 Disaggregated Prefill (分离式预填充)
将推理过程中的 Prefill（预填充）和 Decode（解码）阶段分离到不同的计算资源上。
*   **适用场景**：大规模部署，需要极致的资源效率和稳定的生成速度。
*   **收益**：允许为 Prefill 和 Decode 阶段分配专用硬件（如计算密集型 vs 显存带宽密集型），提高资源利用率。

### 2.4 Standalone Starter (独立启动器)
LMCache 引擎可以作为独立服务运行，不依赖 vLLM 或 GPU。
*   **适用场景**：测试与开发环境、CPU-only 部署、分布式缓存场景、自定义应用集成。

## 3. 快速开始：部署与验证

本节将分别演示如何将 LMCache 集成到 vLLM 和 SGLang 中，并验证其核心功能。

### 3.1 场景验证一：vLLM 集成

本场景演示如何在 vLLM 中启用 LMCache，并通过两个重叠请求验证缓存复用效果。

**1. 安装依赖**

```bash
uv venv --python 3.12
source .venv/bin/activate
uv pip install lmcache vllm
```

**2. 启动带 LMCache 的 vLLM**

**方式一：使用 `kv-transfer-config` 参数（推荐）**

```bash
# LMCACHE_CHUNK_SIZE 仅作演示，生产环境建议使用默认值 256
LMCACHE_CHUNK_SIZE=8 \
vllm serve Qwen/Qwen3-8B-Instruct \
    --port 8000 --kv-transfer-config \
    '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_both"}'
```

> **注意**：如需更多自定义配置，请创建配置文件。详细选项请参考 [LMCache 配置文档](https://docs.lmcache.ai/api_reference/configurations.html)。

**方式二：使用简化参数**

```bash
vllm serve <MODEL NAME> \
    --kv-offloading-backend lmcache \
    --kv-offloading-size <SIZE IN GB> \
    --disable-hybrid-kv-cache-manager
```
> `kv-offloading-size` 单位为 GB。`--disable-hybrid-kv-cache-manager` 标志是必须的。

**3. 验证测试**

打开一个新的终端，发送第一个请求：

```bash
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-8B-Instruct",
    "prompt": "Qwen3 is the latest generation of large language models in Qwen series, offering a comprehensive suite of dense and mixture-of-experts",
    "max_tokens": 100,
    "temperature": 0.7
  }'
```

发送第二个请求（与第一个请求有重叠前缀）：

```bash
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-8B-Instruct",
    "prompt": "Qwen3 is the latest generation of large language models in Qwen series, offering a comprehensive suite of dense and mixture-of-experts (MoE) models",
    "max_tokens": 100,
    "temperature": 0.7
  }'
```

**4. 结果验证**

你应该会看到类似以下的 LMCache 日志：

```text
(EngineCore_DP0 pid=4108) [2025-12-26 18:27:07,860] LMCache INFO: Reqid: cmpl-c98946b90a994947add17e2540ac7025-0, Total tokens 32, LMCache hit tokens: 24, need to load: 8 (vllm_v1_adapter.py:1330:lmcache.integration.vllm.vllm_v1_adapter)
(EngineCore_DP0 pid=4108) [2025-12-26 18:27:07,973] LMCache INFO: Retrieved 8 out of 8 required tokens (from 24 total tokens). size: 0.0009 gb, cost 0.3358 ms, throughput: 2.5449 GB/s; (cache_engine.py:531:lmcache.v1.cache_engine)
(EngineCore_DP0 pid=4108) [2025-12-26 18:27:07,982] LMCache INFO: Storing KV cache for 8 out of 32 tokens (skip_leading_tokens=24) for request cmpl-c98946b90a994947add17e2540ac7025-0 (vllm_v1_adapter.py:1209:lmcache.integration.vllm.vllm_v1_adapter)
(EngineCore_DP0 pid=4108) [2025-12-26 18:27:07,982] LMCache INFO: Stored 8 out of total 8 tokens. size: 0.0009 gb, cost 0.2088 ms, throughput: 4.0933 GB/s; offload_time: 0.1963 ms, put_time: 0.0124 ms (cache_engine.py:303:lmcache.v1.cache_engine)
```

这意味着：第一个请求缓存了 Prompt。第二个请求复用了缓存的前缀，只加载了缺失的块。

### 3.2 场景验证二：SGLang 集成

本场景演示如何在 SGLang 中启用 LMCache。

**1. 安装依赖**

```bash
uv venv --python 3.12
source .venv/bin/activate
uv pip install --prerelease=allow lmcache "sglang"
```

**2. 启动带 LMCache 的 SGLang**

首先创建配置文件 `lmc_config.yaml`：

```bash
cat > lmc_config.yaml <<'EOF'
chunk_size: 8  # 仅供演示；生产环境建议使用 256
local_cpu: true
use_layerwise: true
max_local_cpu_size: 10  # GB
EOF
```

设置环境变量并启动服务：

```bash
export LMCACHE_USE_EXPERIMENTAL=True
export LMCACHE_CONFIG_FILE=$PWD/lmc_config.yaml

python -m sglang.launch_server \
  --model-path Qwen/Qwen3-14B-Instruct \
  --host 0.0.0.0 \
  --port 30000 \
  --enable-lmcache
```

> **注意**：通过配置文件配置 LMCache。

**3. 验证测试**

发送第一个请求：

```bash
curl http://localhost:30000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-14B-Instruct",
    "messages": [{"role": "user", "content": "Qwen3 is the latest generation of large language models in Qwen series, offering a comprehensive suite of dense and mixture-of-experts"}],
    "max_tokens": 100,
    "temperature": 0.7
  }'
```

发送第二个请求（重叠）：

```bash
curl http://localhost:30000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-14B-Instruct",
    "messages": [{"role": "user", "content": "Qwen3 is the latest generation of large language models in Qwen series, offering a comprehensive suite of dense and mixture-of-experts (MoE) models"}],
    "max_tokens": 100,
    "temperature": 0.7
  }'
```

**4. 结果验证**

你应该会看到类似以下的 LMCache 日志：

```text
Prefill batch, #new-seq: 1, #new-token: 3, #cached-token: 32, token usage: 0.00, #running-req: 0, #queue-req: 0
```

### 3.3 场景验证三：KV Cache Sharing (多实例共享)

LMCache 支持通过集中式服务器或 P2P 方式共享 KV Cache，从而减少后续调用的生成时间。

**前置条件**

*   **Centralized Sharing 端口配置**:
    *   vLLM 实例: 8000, 8001
    *   LMCache Server: 65432
*   **P2P Sharing 端口配置**:
    *   vLLM 实例: 8010, 8011
    *   P2P Init: 8200, 8202
    *   P2P Lookup: 8201, 8203
    *   Controller Pull/Reply: 8300, 8400
    *   LMCache Workers: 8500, 8501
    *   Controller Main: 9000

#### 3.3.1 集中式共享 (Centralized Sharing)

此模式演示如何使用集中式 LMCache 服务器在多个 vLLM 实例之间共享 KV Cache。

**1. 创建配置文件**

创建名为 `lmcache_config.yaml` 的文件：

```yaml
chunk_size: 256
local_cpu: true
remote_url: "lm://localhost:65432"
remote_serde: "cachegen"
```


**2. 启动 LMCache Server**

```bash
lmcache_server localhost 65432
```

**3. 启动 vLLM 实例**

在终端 2 中启动第一个实例（GPU 0, Port 8000）：

```bash
LMCACHE_CONFIG_FILE=lmcache_config.yaml \
CUDA_VISIBLE_DEVICES=0 \
vllm serve meta-llama/Meta-Llama-3.1-8B-Instruct \
    --gpu-memory-utilization 0.8 \
    --port 8000 --kv-transfer-config \
    '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_both"}'
```

在终端 3 中启动第二个实例（GPU 1, Port 8001）：

```bash
LMCACHE_CONFIG_FILE=lmcache_config.yaml \
CUDA_VISIBLE_DEVICES=1 \
vllm serve meta-llama/Meta-Llama-3.1-8B-Instruct \
    --gpu-memory-utilization 0.8 \
    --port 8001 \
    --kv-transfer-config \
    '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_both"}'
```

等待两个引擎准备就绪。

**4. 验证测试**

向实例 1 发送请求（生成缓存）：

```bash
curl -X POST http://localhost:8000/v1/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "meta-llama/Meta-Llama-3.1-8B-Instruct",
        "prompt": "Explain the significance of KV cache in language models.",
        "max_tokens": 10
    }'
```

向实例 2 发送相同请求（复用缓存）：

```bash
curl -X POST http://localhost:8001/v1/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "meta-llama/Meta-Llama-3.1-8B-Instruct",
        "prompt": "Explain the significance of KV cache in language models.",
        "max_tokens": 10
    }'
```

第二个请求将自动从第一个实例检索并复用 KV Cache，显著减少生成时间。

#### 3.3.2 P2P 共享 (Peer-to-Peer Sharing)

此模式演示如何使用对等传输在多个 vLLM 实例之间共享 KV Cache。

**1. 配置 LMCache 实例**

创建两个配置文件，分别用于两个实例。

`p2p_example1.yaml` (实例 1 配置):

```yaml
chunk_size: 256
local_cpu: true
max_local_cpu_size: 5
enable_async_loading: True

# P2P configurations
enable_p2p: true
p2p_host: "localhost"
p2p_init_ports: 8200
p2p_lookup_ports: 8201
transfer_channel: "nixl"

# Controller configurations
enable_controller: true
lmcache_instance_id: "lmcache_instance_1"
controller_pull_url: "localhost:8300"
controller_reply_url: "localhost:8400"
lmcache_worker_ports: 8500

extra_config:
  lookup_backoff_time: 0.001
```

`p2p_example2.yaml` (实例 2 配置):

```yaml
chunk_size: 256
local_cpu: true
max_local_cpu_size: 5
enable_async_loading: True

# P2P configurations
enable_p2p: true
p2p_host: "localhost"
p2p_init_ports: 8202
p2p_lookup_ports: 8203
transfer_channel: "nixl"

# Controller configurations
enable_controller: true
lmcache_instance_id: "lmcache_instance_2"
controller_pull_url: "localhost:8300"
controller_reply_url: "localhost:8400"
lmcache_worker_ports: 8501

extra_config:
  lookup_backoff_time: 0.001
```

**2. 运行 P2P 共享工作流 (Docker 环境)**

建议在 Docker 容器中运行以确保环境一致性。

配置环境并进入容器：

```bash
docker pull vllm/vllm-openai:latest
export WEIGHT_DIR="/models"          # 模型权重目录
export CONTAINER_NAME="lmcache_vllm" # 容器名称
export YAML_FILES="/path/to/yaml"    # 包含 yaml 文件的目录（请替换为实际路径）
docker run --name "$CONTAINER_NAME" \
        --detach \
        --ipc=host \
        --network host \
        --gpus all \
        --volume "$WEIGHT_DIR:$WEIGHT_DIR" \
        --volume "$YAML_FILES:$YAML_FILES" \
        --entrypoint "/bin/bash" \
        vllm/vllm-openai:latest -c "time sleep 452d"
docker exec -it "$CONTAINER_NAME" /bin/bash
pip install -U lmcache # 更新 lmcache 到最新版本
```

**3. 启动 Controller**

在容器内启动 LMCache 控制器和监控端点：

```bash
PYTHONHASHSEED=123 lmcache_controller --host localhost --port 9000 --monitor-ports '{"pull": 8300, "reply": 8400}'
```

**4. 启动 vLLM 引擎**

在新的终端（容器内）启动第一个引擎：

```bash
# 假设 yaml 文件在容器内的 $YAML_FILES 路径下
LMCACHE_CONFIG_FILE=$YAML_FILES/p2p_example1.yaml \
CUDA_VISIBLE_DEVICES=0 \
vllm serve meta-llama/Meta-Llama-3.1-8B-Instruct \
    --gpu-memory-utilization 0.8 \
    --port 8010 --kv-transfer-config \
    '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_both"}'
```

在新的终端（容器内）启动第二个引擎：

```bash
LMCACHE_CONFIG_FILE=$YAML_FILES/p2p_example2.yaml \
CUDA_VISIBLE_DEVICES=1 \
vllm serve meta-llama/Meta-Llama-3.1-8B-Instruct \
    --gpu-memory-utilization 0.8 \
    --port 8011 --kv-transfer-config \
    '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_both"}'
```

验证方式与集中式共享类似，分别向端口 8010 和 8011 发送请求即可。

### 3.4 场景验证四：Disaggregated Prefill (分离式架构)

使用 LMCache 作为传输层，实现 Prefill 和 Decode 的分离。此架构包含三个主要组件：

*   **Prefiller Server** (Port 7100): 负责处理 Prompt 阶段，生成 KV Cache 并通过 LMCache 传输。
*   **Decoder Server** (Port 7200): 负责处理 Decoding 阶段，接收 KV Cache 并生成后续 Token。
*   **Proxy Server** (Port 9100): 协调预填充和解码服务器之间的请求路由。

**1. 前置条件**

*   至少 2 个 GPU。
*   Python 包: `lmcache` (0.2.1+), `nixl`, `vllm` (latest main), `httpx`, `fastapi`, `uvicorn`。
*   Hugging Face Token (`HF_TOKEN`)，且有权访问 Llama 3.1 8B 模型。
*   (推荐) 启用 NVLink 或 RDMA 的机器。

**2. 配置文件**

**Prefiller 配置 (`lmcache-prefiller-config.yaml`)**

```yaml
local_cpu: False

# PD-related configurations
enable_pd: True
transfer_channel: "nixl"  # Using NIXL for transfer
pd_role: "sender"          # Prefiller acts as KV cache sender
pd_proxy_host: "localhost" # Host where proxy server is running
pd_proxy_port: 7500        # Port where proxy server is listening
pd_buffer_size: 1073741824  # 1GB buffer for KV cache transfer
pd_buffer_device: "cuda"   # Use GPU memory for buffer
```

**Decoder 配置 (`lmcache-decoder-config.yaml`)**

```yaml
local_cpu: False

# PD-related configurations
enable_pd: True
transfer_channel: "nixl" # Using NIXL for transfer
pd_role: "receiver"        # Decoder acts as KV cache receiver
pd_peer_host: "localhost"  # Host where decoder is listening
pd_peer_init_port: 7300    # Port where initialization happens
pd_peer_alloc_port: 7400   # Port for memory allocation
pd_buffer_size: 1073741824  # 1GB buffer for KV cache transfer
pd_buffer_device: "cuda"   # Use GPU memory for buffer
```

**3. 启动服务**

**步骤 1: 设置环境变量**

```bash
export HF_TOKEN=your_hugging_face_token
```

**步骤 2: 启动 Decoder (GPU 1)**

```bash
UCX_TLS=cuda_ipc,cuda_copy,tcp \
    LMCACHE_CONFIG_FILE=lmcache-decoder-config.yaml \
    CUDA_VISIBLE_DEVICES=1 \
    vllm serve meta-llama/Llama-3.1-8B-Instruct \
    --port 7200 \
    --disable-log-requests \
    --kv-transfer-config \
    '{"kv_connector":"LMCacheConnectorV1","kv_role":"kv_consumer","kv_connector_extra_config": {"discard_partial_chunks": false, "lmcache_rpc_port": "consumer1"}}'
```

**步骤 3: 启动 Prefiller (GPU 0)**

```bash
UCX_TLS=cuda_ipc,cuda_copy,tcp \
    LMCACHE_CONFIG_FILE=lmcache-prefiller-config.yaml \
    CUDA_VISIBLE_DEVICES=0 \
    vllm serve meta-llama/Llama-3.1-8B-Instruct \
    --port 7100 \
    --disable-log-requests \
    --kv-transfer-config \
    '{"kv_connector":"LMCacheConnectorV1","kv_role":"kv_producer","kv_connector_extra_config": {"discard_partial_chunks": false, "lmcache_rpc_port": "producer1"}}'
```

**步骤 4: 启动 Proxy Server**

代理服务器代码位于 vLLM 仓库中。

```bash
python3 ../disagg_proxy_server.py \
  --host localhost \
  --port 9100 \
  --prefiller-host localhost \
  --prefiller-port 7100 \
  --num-prefillers 1 \
  --decoder-host localhost \
  --decoder-port 7200  \
  --decoder-init-port 7300 \
  --decoder-alloc-port 7400 \
  --proxy-host localhost \
  --proxy-port 7500 \
  --num-decoders 1
```

**4. 验证测试**

**步骤 1: 检查服务状态**

确保以下 URL 可访问：
*   Prefiller: `http://localhost:7100/v1/completions`
*   Decoder: `http://localhost:7200/v1/completions`
*   Proxy: `http://localhost:9100/v1/completions`

**步骤 2: 发送请求**

向 Proxy (Port 9100) 发送请求：

```bash
curl http://localhost:9100/v1/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "meta-llama/Llama-3.1-8B-Instruct",
        "prompt": "Tell me a story",
        "max_tokens": 100
    }'
```

**步骤 3: 性能基准测试**

```bash
git clone https://github.com/vllm-project/vllm.git
cd vllm/benchmarks
vllm bench serve --port 9100 --seed $(date +%s) \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --dataset-name random --random-input-len 5000 --random-output-len 200 \
    --num-prompts 50 --burstiness 100 --request-rate 1
```

> **注意**: Prefiller 实例将记录 KV Cache 传输的吞吐量。

### 3.5 场景验证五：Standalone Starter (独立运行)

无需 vLLM 即可运行 LMCache 引擎，适合调试与开发。

**基本用法:**
```bash
python -m lmcache.v1.standalone --config examples/cache_with_configs/example.yaml
```

**CPU-Only 模式:**
```bash
python -m lmcache.v1.standalone \
    --config examples/cache_with_configs/example.yaml \
    --device=cpu \
    --model_name my_model
```

**支持特性:**
*   **MLA (Multi-Level Attention)**: 通过 `--use_mla` 开启。
*   **自定义 KV Shape**: 通过 `--kvcache_shape_spec` 支持多层分组配置，例如 `(2,2,256,4,16):float16:2`。

## 4. 存储后端配置指南

LMCache 支持多种后端存储，以满足不同场景下的性能和成本需求。本节将详细介绍 Redis、Mooncake、Nixl 和 InfiniStore 的配置与部署方法。

### 4.1 存储后端概览

LMCache 遵循自然的存储层级结构：优先使用 GPU 显存，其次是 CPU 内存（Local Storage），最后是远程存储（Remote Offloading）。通过 `remote_url` 参数配置远程连接。

### 4.2 Redis 后端

Redis 是一个内存键值存储系统，也是 LMCache 支持的远程 KV 缓存卸载选项之一。本节主要介绍单节点 Redis 的配置，同时也包含 Redis Sentinel 高可用集群和 LMCache Server 的设置方法。

#### 4.2.1 配置方式

LMCache 支持通过 **环境变量** 和 **配置文件** 两种方式配置 Redis 卸载。

**方式一：环境变量**

```bash
# 设置 KV Chunk 大小 256
export LMCACHE_CHUNK_SIZE=256

# 设置 Redis 地址
export LMCACHE_REMOTE_URL="redis://your-redis-host:6379"

# 或者设置 Redis Sentinel 地址 (高可用模式)
# export LMCACHE_REMOTE_URL="redis-sentinel://localhost:26379,localhost:26380,localhost:26381"

# 或者设置 LMCache Server 地址
# export LMCACHE_REMOTE_URL="lm://localhost:65432"

# 设置远程传输序列化方式: "naive" (默认) 或 "cachegen"
export LMCACHE_REMOTE_SERDE="naive"
```

**方式二：配置文件**

通过 `LMCACHE_CONFIG_FILE` 环境变量指定配置文件路径（例如 `redis-offload.yaml`）：

```yaml
# 256 Tokens per KV Chunk
chunk_size: 256

# Redis host
remote_url: "redis://your-redis-host:6379"

# Redis Sentinel hosts (for high availability)
# remote_url: "redis-sentinel://localhost:26379,localhost:26380,localhost:26381"

# LMCache Server host
# remote_url: "lm://localhost:65432"

# 序列化方式
remote_serde: "naive" # "naive" (default) or "cachegen"
```

> [!TIP]
> **远程存储说明**：LMCache 遵循自然的存储层级：优先 CPU RAM，其次本地存储，最后是远程卸载。`remote_url` 格式为 `connector_type://host:port`。如果设置为 `None`，则不使用远程存储。

> [!NOTE]
>  `remote_url` 的示例：
>```
>remote_url: "redis://your-redis-host:6379"
>remote_url: "redis-sentinel://localhost:26379,localhost:26380,localhost:26381"
>remote_url: "lm://localhost:65432"
>remote_url: "infinistore://127.0.0.1:12345"
>remote_url: "mooncakestore://127.0.0.1:50051"
>```


#### 4.2.2 Redis 部署示例

本节演示最基础的单机 Redis 部署方案。

**前置条件**：
*   拥有一台带 GPU 的机器。
*   已安装 `vllm` 和 `lmcache`。
*   拥有 Hugging Face Token (`HF_TOKEN`)。

**1. 启动 Redis 服务**

安装并启动 Redis：

```bash
# Ubuntu / Debian 安装
sudo apt-get install redis
redis-server # 在默认端口 6379 启动
```

检查服务状态：

```bash
redis-cli ping
# 预期输出: PONG
```

**2. 启动 vLLM**

创建配置文件 `redis.yaml`：

```yaml
chunk_size: 256
remote_url: "redis://localhost:6379"
remote_serde: "naive"
```

启动 vLLM：

```bash
LMCACHE_CONFIG_FILE=redis.yaml \
vllm serve meta-llama/Meta-Llama-3.1-8B-Instruct \
    --gpu-memory-utilization 0.8 \
    --kv-transfer-config \
    '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_both"}'
```

#### 4.2.3 Redis 高可用部署示例

若需高可用支持，可配置 Redis Sentinel 监控主节点并自动故障转移。

**1. 启动 Redis 副本**

```bash
redis-server --port 6380 --replicaof 127.0.0.1 6379
```

**2. 配置并启动 Sentinel**

创建 3 个 Sentinel 配置文件（`sentinel-26379.conf`, `sentinel-26380.conf`, `sentinel-26381.conf`），内容如下（注意修改端口）：

```text
port 26379
sentinel monitor mymaster 127.0.0.1 6379 1
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1
```

启动 3 个 Sentinel 实例：

```bash
redis-server sentinel-26379.conf --sentinel &
redis-server sentinel-26380.conf --sentinel &
redis-server sentinel-26381.conf --sentinel &
```

验证 Sentinel 状态：

```bash
redis-cli -p 26379 sentinel master mymaster
```

**3. 启动 vLLM**

创建配置文件 `redis-sentinel.yaml`：

```yaml
chunk_size: 256
# 连接到 Sentinel 集群
remote_url: "redis-sentinel://localhost:26379,localhost:26380,localhost:26381"
remote_serde: "naive"
```

启动 vLLM：

```bash
LMCACHE_CONFIG_FILE=redis-sentinel.yaml \
vllm serve meta-llama/Meta-Llama-3.1-8B-Instruct \
    --gpu-memory-utilization 0.8 \
    --kv-transfer-config \
    '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_both"}'
```

#### 4.2.4 LMCache Server 部署示例

LMCache Server 是一个轻量级的远程存储替代方案，目前仅支持 CPU 存储。

**1. 启动 LMCache Server**

```bash
# 格式: lmcache_server <host> <port> <device>
lmcache_server localhost 65432 cpu
```

**2. 启动 vLLM**

创建配置文件 `lmcache-server.yaml`：

```yaml
chunk_size: 256
remote_url: "lm://localhost:65432"
remote_serde: "naive"
```

启动 vLLM：

```bash
LMCACHE_CONFIG_FILE=lmcache-server.yaml \
vllm serve meta-llama/Meta-Llama-3.1-8B-Instruct \
    --gpu-memory-utilization 0.8 \
    --kv-transfer-config \
    '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_both"}'
```

### 4.3 Mooncake 后端

Mooncake 是专为 LLM 推理设计的开源分布式 KV 缓存存储系统。它通过聚合多个客户端节点的空闲内存（DRAM）和 SSD 资源，构建统一的分布式内存池，从而最大化集群资源利用率。

**核心特性**：
*   **分布式内存池化**：汇聚多节点资源。
*   **高带宽利用率**：支持条带化和并行 I/O 传输，充分利用多网卡聚合带宽。
*   **RDMA 优化**：基于 Transfer Engine 构建，支持 TCP 和 RDMA (InfiniBand/RoCEv2/eRDMA/NVIDIA GPUDirect)。
*   **动态资源伸缩**：支持动态增减节点。

> [!WARNING]
> lmcache v0.13.1 及之前版本中存在 `CacheEngineKey.to_string()` 格式化类型错误的问题。
>
> **解决方案**：
>    **手动修复**：如果无法升级，可以修改 `lmcache/utils.py` 文件中 `CacheEngineKey` 类的 `to_string` 方法。
>    将：
>    ```python
>    f"@{self.worker_id}@{self.chunk_hash:x}@{self._dtype_str}"
>    ```
>    修改为：
>    ```python
>    f"@{self.worker_id}@{self.chunk_hash}@{self._dtype_str}"
>    ```

#### 4.3.1 安装与环境准备

**前置条件**：
*   至少一台带 GPU 的机器。
*   支持 RDMA 的网络硬件及驱动（推荐）或 TCP 网络。
*   Python 3.8+。

**安装 Mooncake**：

```bash
pip install mooncake-transfer-engine
```

该包包含 `mooncake_master`（集群元数据管理）和 `mooncake_http_metadata_server`（HTTP 元数据服务）等组件。

#### 4.3.2 启动基础服务

在配置 vLLM 之前，需要先启动 Mooncake Master 服务。

```bash
# 启动 Master 服务（开启内置 HTTP 元数据服务）
# -v=1 可开启详细日志
mooncake_master --enable_http_metadata_server=1
```

预期输出示例：
```text
Master service started on port 50051
HTTP metrics server started on port 9003
```

*   Port **50051**: Master RPC 服务端口。
*   Port **9003**: Metrics 监控端口（可访问 `http://localhost:9003` 查看指标）。

#### 4.3.3 配置指南

创建配置文件 `mooncake-config.yaml`。

```yaml
# LMCache 基础配置
chunk_size: 16 # 用于验证，生产环境建议 256 或以上
save_unfull_chunk: true
local_cpu: False
# Mooncake 连接 URL (指向 Master RPC 端口)
remote_url: "mooncakestore://localhost:50051/"
max_local_cpu_size: 2  # 即使 local_cpu=False 也需配置缓冲大小
numa_mode: null      # 多 NUMA/多网卡系统建议设为 auto 以降低长尾延迟
pre_caching_hash_algorithm: sha256_cbor_64bit

# Mooncake 引擎高级配置
extra_config:
  use_exists_sync: true
  save_chunk_meta: False  # 启用块元数据优化
  local_hostname: "localhost" # 当前节点标识
  metadata_server: "http://localhost:8080/metadata" # HTTP 元数据服务地址
  protocol: "tcp"        # 传输协议: "rdma" 或 "tcp"
  device_name: ""         # 留空以自动检测设备
  global_segment_size: 21474836480 # 每个 Worker 分配的内存段大小 (20 GiB)
  master_server_address: "localhost:50051" # Master RPC 地址
  local_buffer_size: 0    # 依赖 LMCache local_cpu 作为缓冲
  mooncake_prefer_local_alloc: true # 优先本地分配
```

> [!TIP]
> **一致性哈希说明**：如果遇到缓存未命中问题，请确保跨进程的 `PYTHONHASHSEED` 固定（例如 `export PYTHONHASHSEED=0`）。

#### 4.3.4 启动 vLLM 与验证

**启动 vLLM**：

```bash
export PYTHONHASHSEED=0
LMCACHE_CONFIG_FILE="mooncake-config.yaml" \
vllm serve \
    meta-llama/Llama-3.1-8B-Instruct \
    --max-model-len 65536 \
    --kv-transfer-config \
    '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_both"}'
```

**验证部署**：

发送推理请求以测试集成：

```bash
curl -X POST "http://localhost:8000/v1/completions" \
     -H "Content-Type: application/json" \
     -d '{
       "model": "meta-llama/Llama-3.1-8B-Instruct",
       "prompt": "The future of AI is",
       "max_tokens": 100,
       "temperature": 0.7
     }'
```

**调试建议**：
*   **查看服务状态**：`ps aux | grep mooncake` 或 `netstat -tlnp | grep -E "(8080|50051)"`。
*   **监控指标**：访问 `http://localhost:9003`。
*   **详细日志**：启动 Master 时添加 `-v=1` 参数。

### 4.4 InfiniStore 后端

InfiniStore 是一款开源的高性能 KV 存储系统，专为 LLM 推理集群设计。它支持 RDMA 传输，能够在集群节点间实现低延迟的 KV 缓存传输与复用，适用于P-D分离架构（Prefill-Decoding Disaggregation）和非分离（Non-disaggregated）集群中的大容量缓存扩展。

**注意**：LMCache 的 InfiniStore 连接器目前仅支持 RDMA 传输协议。

#### 4.4.1 安装与环境准备

**前置条件**：
*   至少一台带 GPU 的机器。
*   支持 RDMA 的网络硬件及驱动（InfiniBand 或 RoCE）。
*   Python 3.8+。

**安装 InfiniStore**：

```bash
pip install infinistore
```

#### 4.4.2 启动 InfiniStore Server

根据您的网络硬件类型选择启动命令：

**场景 A：InfiniBand (IB) 网络**

```bash
# --dev-name 指定 IB 设备名 (如 mlx5_0)
infinistore --service-port 12345 --dev-name mlx5_0 --link-type IB
```

**场景 B：RoCE (Ethernet) 网络**

```bash
# --link-type 设为 Ethernet
infinistore --service-port 12345 --dev-name mlx5_0 --link-type Ethernet
```

> [!TIP]
> **Kubernetes 环境提示**：在 K8s 环境中，可以使用 `--hint-gid-index` 选项指定 GID 索引。

#### 4.4.3 配置指南

创建配置文件 `infinistore-config.yaml`。请确保 `device` 参数指向当前机器上可用的 RDMA 设备（通常应与 Server 端一致或互通）。

```yaml
chunk_size: 256
# 格式: infinistore://<server_ip>:<port>/?device=<local_rdma_device>
remote_url: "infinistore://127.0.0.1:12345/?device=mlx5_1"
remote_serde: "naive"
local_cpu: False
max_local_cpu_size: 5
```

#### 4.4.4 启动 vLLM 与验证

**启动 vLLM**：

```bash
LMCACHE_CONFIG_FILE="infinistore-config.yaml" \
vllm serve \
    Qwen/Qwen2.5-7B-Instruct \
    --seed 42 \
    --max-model-len 16384 \
    --gpu-memory-utilization 0.8 \
    --kv-transfer-config \
    '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_both"}'
```

**验证部署**：

发送测试请求：

```bash
curl -X POST "http://localhost:8000/v1/completions" \
     -H "Content-Type: application/json" \
     -d '{
       "model": "Qwen/Qwen2.5-7B-Instruct",
       "prompt": "The future of AI is",
       "max_tokens": 100,
       "temperature": 0.7
     }'
```

**调试建议**：
*   **查看服务状态**：`ps aux | grep infinistore` 或 `netstat -tlnp | grep 12345`。
*   **开启调试日志**：启动 Server 时添加 `--log-level=debug`。

### 4.5 Nixl 后端

NIXL (NVIDIA Inference Xfer Library) 是一个高性能库，专为加速 AI 推理框架中的点对点通信而设计。它通过模块化的插件架构提供对各种类型内存（CPU 和 GPU）和存储的抽象，实现推理管道不同组件之间的高效数据传输和协调。
LMCache 支持使用 NIXL 作为存储后端，允许使用 NIXL 将 GPU 或 CPU 内存保存到存储中。

#### 4.5.1 前置条件

*   **LMCache**: 已通过 `pip install lmcache` 安装。
*   **NIXL**: 已从 NIXL GitHub 仓库安装。
*   **模型访问权限**: 拥有有效的 Hugging Face token (HF_TOKEN)。

#### 4.5.2 配置 LMCache NIXL Offloading

通过 `LMCACHE_CONFIG_FILE` 环境变量指定配置文件（例如 `lmcache-config.yaml`）。

**1. POSIX 后端配置示例**

适用于本地文件系统。

```yaml
chunk_size: 256
nixl_buffer_size: 1073741824 # 1GB
nixl_buffer_device: cpu
extra_config:
  enable_nixl_storage: true
  nixl_backend: POSIX
  nixl_pool_size: 64
  nixl_path: /mnt/nixl/cache/
  use_direct_io: True
```

**关键设置说明**：

*   `nixl_buffer_size`: NIXL 传输的缓冲区大小。
*   `nixl_pool_size`: 初始化时为 NIXL 后端打开的描述符数量。
*   `nixl_path`: 存储文件保存的目录（例如 `/mnt/nixl/`）。对于存储到文件的 NIXL 后端是必需的。
*   `nixl_buffer_device`: 指定 NIXL 管理的内存所在位置。
    *   支持 "cpu" 或 "cuda" 的后端："GDS", "GDS_MT"。
    *   必须为 "cpu" 的后端："POSIX", "HF3FS", "OBJ"。
*   `nixl_backend`: 配置用于存储的 NIXL 后端类型。支持的后端包括：`["GDS", "GDS_MT", "POSIX", "HF3FS", "OBJ"]`。

**2. OBJ 后端 (S3) 配置示例**

适用于 S3 兼容的对象存储。

```yaml
chunk_size: 256
nixl_buffer_size: 1073741824 # 1GB
nixl_buffer_device: cpu
extra_config:
  enable_nixl_storage: true
  nixl_backend: OBJ
  nixl_pool_size: 64
  nixl_path: /mnt/nixl/cache/
  nixl_backend_params:
    access_key: <your_access_key>
    secret_key: <your_secret_key>
    bucket: <your_bucket>
    region: <your_region>
```

> **注意**：后端特定的参数应通过 `extra_config.nixl_backend_params` 提供。具体参数请参考 NIXL 文档。

## 5. 参考资料

*   [LMCache GitHub Repository](https://github.com/LMCache/LMCache)
*   [LMCache Documentation: Storage Backends](https://docs.lmcache.ai/kv_cache/storage_backends/index.html)
    *   [Redis Backend Documentation](https://docs.lmcache.ai/kv_cache/storage_backends/redis.html)
    *   [Mooncake Backend Documentation](https://docs.lmcache.ai/kv_cache/storage_backends/mooncake.html)
    *   [Nixl Backend Documentation](https://docs.lmcache.ai/kv_cache/storage_backends/nixl.html)
*   [LMCache Quickstart Documentation](https://github.com/LMCache/LMCache/blob/dev/docs/source/getting_started/quickstart/index.rst)
*   [LMCache vLLM Integration Guide](https://docs.lmcache.ai/getting_started/quickstart.rst)

## 附录一：LMCache Server 部署参考 (Docker Compose)

### docker-compose.yaml

```yaml
version: '3.8'

services:
  # 1. LMCache Server 实例
  # 作用：作为集中式 KV Cache 存储后端，负责跨实例的数据交换
  lmcache-server:
    image: vllm/vllm-openai:latest
    container_name: lmcache-server
    # 启动命令：监听所有网络接口的 65432 端口
    # 注意：官方镜像不包含 lmcache，需要手动安装
    entrypoint: ["/bin/bash", "-c"]
    command: >
      pip install lmcache &&
      lmcache_server 0.0.0.0 65432
    ports:
      - "65432:65432"
    environment:
      # 关键配置：确保 Python 哈希种子一致，这对于 KV Cache Key 的生成至关重要
      - PYTHONHASHSEED=0
    networks:
      - lmcache-net
    # 资源隔离配置：模拟 Kubernetes Pod 的资源限制
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 8G

  # 2. vLLM 实例 1
  # 作用：第一个 LLM 推理服务，连接 LMCache Server
  vllm-instance-1:
    image: vllm/vllm-openai:latest
    container_name: vllm-instance-1
    depends_on:
      - lmcache-server
    volumes:
      # 挂载 LMCache 配置文件，其中指定了 remote_url 为 lm://lmcache-server:65432
      - ./lmcache_config.yaml:/etc/lmcache_config.yaml
      # 挂载本地模型路径到容器内
      - ${MODEL_PATH}:/models/qwen
    environment:
      # 指定配置文件路径
      - LMCACHE_CONFIG_FILE=/etc/lmcache_config.yaml
      # 必须与 Server 和其他 vLLM 实例保持一致
      - PYTHONHASHSEED=0
    ports:
      - "8080:8080"
    # 启动命令说明：
    # --kv-transfer-config: 启用 LMCache 连接器
    # --gpu-memory-utilization: 设置 GPU 显存占用率
    # --served-model-name: 指定对外的模型名称
    # 使用 bash entrypoint 安装 lmcache 并启动服务
    entrypoint: ["/bin/bash", "-c"]
    command: >
      pip install lmcache &&
      vllm serve /models/qwen
      --served-model-name qwen
      --gpu-memory-utilization 0.8
      --port 8080
      --kv-transfer-config '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_both"}'
    networks:
      - lmcache-net
    # GPU 资源配置
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
              # 如果需要绑定特定 GPU，取消注释下行并指定 ID
              # device_ids: ['0']

  # 3. vLLM 实例 2
  # 作用：第二个 LLM 推理服务，共享实例 1 的 KV Cache
  vllm-instance-2:
    image: vllm/vllm-openai:latest
    container_name: vllm-instance-2
    depends_on:
      - lmcache-server
    volumes:
      - ./lmcache_config.yaml:/etc/lmcache_config.yaml
      - ${MODEL_PATH}:/models/qwen
    environment:
      - LMCACHE_CONFIG_FILE=/etc/lmcache_config.yaml
      - PYTHONHASHSEED=0
    ports:
      - "8081:8080"
    # 启动参数与实例 1 保持一致（除了宿主机映射端口），容器内端口均为 8080
    entrypoint: ["/bin/bash", "-c"]
    command: >
      pip install lmcache &&
      vllm serve /models/qwen
      --served-model-name qwen
      --gpu-memory-utilization 0.8
      --port 8080
      --kv-transfer-config '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_both"}'
    networks:
      - lmcache-net
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
              # 如果需要绑定特定 GPU，取消注释下行并指定 ID
              # device_ids: ['1']

networks:
  lmcache-net:
    driver: bridge
```

### lmcache_config.yaml

```yaml
chunk_size: 256
local_cpu: true
remote_url: "lm://lmcache-server:65432"
remote_serde: "cachegen"
```

## 附录二：Mooncake + LMCache + vLLM P-D 分离部署 (Docker Compose)

本附录提供基于 Mooncake 后端实现 Prefill-Decode 分离架构的 Docker Compose 部署方案。该方案包含 Mooncake Master、Prefiller、Decoder 和 Proxy Server 四个服务。

### docker-compose.yaml

```yaml
version: '3.8'

services:
  # 1. Mooncake Master
  # 负责管理集群元数据
  mooncake-master:
    image: vllm/vllm-openai:latest
    container_name: mooncake-master
    entrypoint: ["/bin/bash"]
    command: >
      pip install mooncake-transfer-engine &&
      mooncake_master -port 50052 -max_threads 64 -metrics_port 9004
      --enable_http_metadata_server=true
      --http_metadata_server_host=0.0.0.0
      --http_metadata_server_port=8080
    ports:
      - "8080:8080"   # Metadata Server
      - "50052:50052" # RPC Port
      - "9004:9004"   # Metrics Port
    networks:
      - lmcache-net

  # 2. Prefiller (KV Producer)
  # 负责处理 Prompt 阶段，生成 KV Cache
  prefiller:
    image: vllm/vllm-openai:latest
    container_name: prefiller
    depends_on:
      - mooncake-master
    volumes:
      - ./mooncake-prefiller-config.yaml:/app/mooncake-prefiller-config.yaml
    environment:
      - LMCACHE_CONFIG_FILE=/app/mooncake-prefiller-config.yaml
      - LMCACHE_USE_EXPERIMENTAL=True
      - VLLM_ENABLE_V1_MULTIPROCESSING=1
      - PYTHONHASHSEED=0
    entrypoint: ["/bin/bash"]
    command: >
      pip install lmcache mooncake-transfer-engine &&
      vllm serve /models/qwen
      --port 8100
      --kv-transfer-config '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_producer"}'
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    networks:
      - lmcache-net

  # 3. Decoder (KV Consumer)
  # 负责处理 Decoding 阶段，消费 KV Cache
  decoder:
    image: vllm/vllm-openai:latest
    container_name: decoder
    depends_on:
      - mooncake-master
    volumes:
      - ./mooncake-decoder-config.yaml:/app/mooncake-decoder-config.yaml
    environment:
      - LMCACHE_CONFIG_FILE=/app/mooncake-decoder-config.yaml
      - LMCACHE_USE_EXPERIMENTAL=True
      - VLLM_ENABLE_V1_MULTIPROCESSING=1
      - PYTHONHASHSEED=0
    entrypoint: ["/bin/bash"]
    command: >
      pip install lmcache mooncake-transfer-engine &&
      vllm serve /models/qwen
      --port 8200
      --kv-transfer-config '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_consumer"}'
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    networks:
      - lmcache-net

  # 4. Proxy Server
  # 负责请求路由
  proxy:
    image: python:3.10-slim
    container_name: proxy
    depends_on:
      - prefiller
      - decoder
    # 假设 disagg_proxy_server.py 已存在于当前目录
    volumes:
      - ./disagg_proxy_server.py:/app/disagg_proxy_server.py
    working_dir: /app
    command: >
      python3 disagg_proxy_server.py
      --host 0.0.0.0 --port 9000
      --prefiller-host prefiller --prefiller-port 8100
      --decoder-host decoder --decoder-port 8200
    ports:
      - "9000:9000"
    networks:
      - lmcache-net

networks:
  lmcache-net:
    driver: bridge
```

### 配置文件

**1. Decoder 配置 (`mooncake-decoder-config.yaml`)**

```yaml
chunk_size: 16
remote_url: "mooncakestore://mooncake-master:50052/"
remote_serde: "naive"
local_cpu: False
max_local_cpu_size: 2
numa_mode: null 

extra_config:
  local_hostname: "decoder"
  metadata_server: "http://mooncake-master:8080/metadata"
  protocol: "tcp" # 容器环境默认使用 TCP，如需 RDMA 请改为 "rdma" 并挂载设备
  device_name: ""
  master_server_address: "mooncake-master:50052"
  global_segment_size: 32212254720
  local_buffer_size: 1073741824
  transfer_timeout: 1
  save_chunk_meta: False
```

**2. Prefiller 配置 (`mooncake-prefiller-config.yaml`)**

```yaml
chunk_size: 16
remote_url: "mooncakestore://mooncake-master:50052/"
remote_serde: "naive"
local_cpu: False
max_local_cpu_size: 2
numa_mode: null 

extra_config:
  local_hostname: "prefiller"
  metadata_server: "http://mooncake-master:8080/metadata"
  protocol: "tcp"
  device_name: ""
  master_server_address: "mooncake-master:50052"
  global_segment_size: 32212254720
  local_buffer_size: 1073741824
  transfer_timeout: 1
  save_chunk_meta: False
```

> **注意**：运行此配置需要确保 `disagg_proxy_server.py` 文件存在于当前目录中。该脚本可从 vLLM 或 LMCache 仓库获取。

