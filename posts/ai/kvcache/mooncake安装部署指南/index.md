# MoonCake安装部署指南


## 1. 项目简介

Mooncake 是一个以此 KVCache 为中心的分布式架构，旨在分离 Prefill 和 Decoding 集群，并利用未充分利用的 CPU、DRAM 和 SSD 资源来实现 KVCache 的解耦。

> [!NOTE]
> 本文旨在整理 Mooncake 的核心概念及快速上手指南，后续将持续补充深度架构解析与性能调优内容。

## 2. 核心架构解析

Mooncake 由三个主要组件组成，协同工作以实现高效的分布式 LLM 服务。

### 2.1 传输引擎 (Transfer Engine)

**传输引擎 (TE)** 作为基础，提供统一的批量数据传输接口，支持各种存储设备和网络协议。

*   **支持协议**：TCP、RDMA (InfiniBand/RoCEv2/eRDMA/NVIDIA GPUDirect)、CXL/共享内存和 NVMe over Fabric (NVMe-of)。
*   **核心能力**：通过拓扑感知路径选择和带宽聚合实现快速可靠的数据移动。
*   **TODO: 补充传输引擎的性能基准测试数据 (Benchmarks)**

### 2.2 存储系统 (Storage System)

基于传输引擎构建的两种存储解决方案：

*   **P2P Store**：用于在集群节点间共享临时对象（如检查点文件）的去中心化架构。
*   **Mooncake Store**：专为 LLM 推理设计的分布式 KVCache 存储引擎，支持多副本和高带宽利用率。
*   **TODO: 补充 Mooncake Store 的数据分布策略与一致性模型**

### 2.3 调度系统 (Scheduler)

*   **TODO: 补充 KVCache-centric 调度器的设计原理**
*   **TODO: 补充预测性拒绝策略 (Prediction-based Early Rejection Policy) 的工作机制**

### 2.4 LLM 集成框架

与 vLLM、SGLang 和 LMCache 等领先推理系统的无缝集成，支持 prefill-decode disaggregation 和分层 KV 缓存。

**SGLang HiCache 集成优势：**
*   **无限容量 (Scalable Capacity)**：将集群内存聚合成巨大的分布式池。
*   **缓存共享 (Cache Sharing)**：KV Cache 可被集群内所有实例共享。
*   **RDMA 加速**：直接内存访问消除 CPU 开销，降低延迟。
*   **零拷贝 (Zero Copy)**：L2 与 Mooncake 间直接传输，最大化吞吐。
*   **容错性 (Fault Tolerance)**：分布式架构提供节点级故障恢复能力。

## 3. 快速开始：部署与验证

本节介绍如何快速部署 Mooncake 并验证其与 SGLang 的集成。

### 3.1 环境准备

首先需要安装 Mooncake 传输引擎和 SGLang 路由组件。确保已安装 SGLang 且支持 HiCache。

> [!TIP]
> 建议使用清华源加速下载。

```bash
pip install -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple  mooncake-transfer-engine
pip install -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple  sglang-router
```

> [!NOTE]
> **源码安装 (可选)**：
> 如果需要最新功能，可从源码编译安装：
> ```bash
> git clone https://github.com/kvcache-ai/Mooncake --recursive
> cd Mooncake && bash dependencies.sh
> mkdir build && cd build && cmake .. && make -j && sudo make install
> ```

### 3.2 启动核心服务 (Mooncake Master)

Mooncake Master 负责管理集群元数据。在单机部署模式下，我们可以通过参数直接启用内置的 HTTP Metadata Server。

```bash
mooncake_master --enable_http_metadata_server=true --http_metadata_server_port=8080 --eviction_high_watermark_ratio=0.95
```

**参数说明：**
*   `--enable_http_metadata_server=true`: 启用内置 HTTP 元数据服务器 (无需单独启动 `mooncake.http_metadata_server`)。
*   `--http_metadata_server_port=8080`: 指定元数据服务器端口。
*   `--eviction_high_watermark_ratio=0.95`: 设置驱逐高水位比例。

### 3.3 场景验证一：SGLang 集成 (HiCache)

启动 SGLang Server，并配置 Mooncake 作为分层缓存（Hierarchical Cache）的后端存储 (L3)。

> [!WARNING]
> 请确保 `MOONCAKE_MASTER` 环境变量指向正确的 Master 地址。
> `--hicache-size` 参数单位为 GB。

```bash
MOONCAKE_MASTER=127.0.0.1:50051 uv run python3 -m sglang.launch_server --model-path /mnt/inaisfs/loki/bussiness/LLMs/Qwen3-1.7B --trust-remote-code --enable-hierarchical-cache --hicache-size 24 --page-size 64 --hicache-io-backend kernel --hicache-mem-layout page_first --hicache-storage-backend mooncake
```

**关键参数解析：**
*   `--enable-hierarchical-cache`: 启用分层缓存机制。
*   `--hicache-size 24`: 设置 Mooncake 使用的缓存大小为 24GB。
*   `--hicache-storage-backend mooncake`: 指定使用 Mooncake 作为存储后端。
*   `--hicache-io-backend kernel`: 指定 IO 后端实现。
*   `--hicache-storage-prefetch-policy timeout`: (推荐) 决定何时停止从存储预取。`timeout` 通常在 Mooncake 作为后端时提供最佳平衡。

**冒烟测试 (Smoke Test):**

```bash
curl -X POST http://127.0.0.1:30000/generate \
    -H "Content-Type: application/json" \
    -d '{
  "text": "Let me tell you a long story ",
  "sampling_params": {
    "temperature": 0
  }
}'
```

> [!TIP]
> 如果遇到 HuggingFace 超时，可设置 `export SGLANG_USE_MODELSCOPE=true`。
> 使用 `--tp-size` 可启用多 GPU 张量并行。

### 3.4 场景验证二：SGLang Prefill-Decode 分离 (Disaggregation)

该架构运行三个进程：Prefill Worker、Decode Worker 和 Router。请在不同的终端窗口中分别启动。

**1. 启动 Prefill Worker**

```bash
MOONCAKE_MASTER=127.0.0.1:50051 uv run python3 -m sglang.launch_server \
    --model-path /mnt/inaisfs/loki/bussiness/LLMs/Qwen3-1.7B \
    --trust-remote-code \
    --page-size 64 \
    --enable-hierarchical-cache \
    --hicache-storage-prefetch-policy timeout \
    --hicache-storage-backend mooncake \
    --disaggregation-mode prefill \
    --disaggregation-ib-device "mlx5_1" \
    --base-gpu-id 0 \
    --port 30000
```
> [!NOTE]
> `--disaggregation-ib-device` 是可选的，SGLang 会自动检测，但在多网卡环境下建议显式指定。

**2. 启动 Decode Worker**

```bash
uv run python3 -m sglang.launch_server \
    --model-path /mnt/inaisfs/loki/bussiness/LLMs/Qwen3-1.7B \
    --trust-remote-code \
    --page-size 64 \
    --disaggregation-mode decode \
    --disaggregation-ib-device "mlx5_1" \
    --base-gpu-id 1 \
    --port 30001
```
> [!TIP]
> 可选参数 `--disaggregation-decode-enable-offload-kvcache`：将 Decode Worker 的输出写回 Mooncake (L3 持久化)。

**3. 启动 Router**

```bash
uv run python3 -m sglang_router.launch_router \
    --pd-disaggregation \
    --prefill "http://127.0.0.1:30000" \
    --decode "http://127.0.0.1:30001" \
    --host 0.0.0.0 \
    --port 8000
```

**4. 验证测试**

```bash
curl -X POST http://127.0.0.1:8000/generate \
    -H "Content-Type: application/json" \
    -d '{"text": "Mooncake is ", "sampling_params": {"temperature": 0}}'
```

### 3.5 场景验证三：vLLM 原生集成 (Mooncake Connector)

> [!NOTE]
> vLLM v1 引擎后端已支持通过 Mooncake Connector 实现 Prefill-Decode (PD) 分离架构。该集成利用 RDMA 技术实现高效的跨节点 KV 缓存传输。

本节展示如何使用 MooncakeConnector 在 vLLM v1 后端实现 PD 分离服务。

1. **前置准备**

确保已安装 vLLM，并安装 Mooncake 传输引擎：

```bash
pip install mooncake-transfer-engine
```

**2. 启动服务**

假设我们有两台节点（或在同一台机器上使用不同端口模拟）：
*   Prefiller Node: 192.168.0.2
*   Decoder Node: 192.168.0.3

**Prefiller Node (KV Producer)**

```bash
vllm serve Qwen/Qwen2.5-7B-Instruct \
  --port 8010 \
  --kv-transfer-config '{"kv_connector":"MooncakeConnector","kv_role":"kv_producer"}'
```

**Decoder Node (KV Consumer)**

```bash
vllm serve Qwen/Qwen2.5-7B-Instruct \
  --port 8020 \
  --kv-transfer-config '{"kv_connector":"MooncakeConnector","kv_role":"kv_consumer"}'
```

**3. 启动 Proxy Server**

目前可以使用 vLLM 提供的示例代理服务器进行路由：

```bash
# 在 vllm 根目录下执行
python tests/v1/kv_connector/nixl_integration/toy_proxy_server.py \
  --prefiller-host 192.168.0.2 --prefiller-port 8010 \
  --decoder-host 192.168.0.3 --decoder-port 8020
```

 **4. 验证测试**

向 Proxy (默认端口 8000) 发送请求进行测试：

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-7B-Instruct",
    "messages": [
      {"role": "user", "content": "Tell me a long story about artificial intelligence."}
    ]
  }'
```

### 3.6 场景验证四：vLLM + LMCache 集成

Mooncake 作为 LMCache 的后端存储引擎，支持在 vLLM v1 环境下实现基于 KV Cache 复用的 Prefill-Decode (PD) 分离架构。此方案利用 Mooncake 的分布式存储池和 RDMA 传输能力，结合 LMCache 的灵活缓存管理，实现高效的跨节点 KV 传输。

> [!NOTE]
> **基于集中式存储的 KV Cache 共享 (Centralized Storage Shared KVCache)**
>
> 在 Mooncake + LMCache 的 PD 分离架构中，Prefill 和 Decode 节点通过共享的 Mooncake Store 集群进行交互。Prefill 节点生成的 KV Cache 被写入 Mooncake 分布式存储池，Decode 节点则直接从池中读取所需的 KV Cache。
> 这种架构消除了点对点 (P2P) 传输的复杂性，允许 Decode 节点“零拷贝”地（在同一集群视角下）访问 Prefill 的结果，极大地提升了首字延迟 (TTFT) 和系统吞吐量。
> 
> 更多关于集中式 KV 共享的机制，请参考 LMCache 文档：[Example: Share KV cache across multiple LLMs](https://docs.lmcache.ai/getting_started/quickstart/share_kv_cache.html)

**部署步骤：**

**Step 1: 启动 Mooncake Master (Machine A)**

Mooncake Master 负责管理集群元数据。

```bash
mooncake_master -port 50052 -max_threads 64 -metrics_port 9004 \
  --enable_http_metadata_server=true \
  --http_metadata_server_host=0.0.0.0 \
  --http_metadata_server_port=8080
```

**Step 2: 准备配置文件**

需分别为 Decoder 和 Prefiller 准备 YAML 配置文件。

**Decoder 配置 (`mooncake-decoder-config.yaml`)**:
```yaml
chunk_size: 16 # 用于验证，生产环境建议 256 或以上
remote_url: "mooncakestore://{IP_of_Machine_A}:50052/"
remote_serde: "naive"
local_cpu: False
max_local_cpu_size: 2
numa_mode: null 

extra_config:
  local_hostname: "{IP_of_Machine_A}" # Decoder 所在节点 IP
  metadata_server: "http://{IP_of_Machine_A}:8080/metadata"
  protocol: "rdma" # 或 tcp
  device_name: "mlx5_0" 
  master_server_address: "{IP_of_Machine_A}:50052"
  global_segment_size: 32212254720 # 30GB
  local_buffer_size: 1073741824 # 1GB
  transfer_timeout: 1
  save_chunk_meta: False
```

**Prefiller 配置 (`mooncake-prefiller-config.yaml`)**:
```yaml
# 与 Decoder 配置类似，注意修改 local_hostname
chunk_size: 16 # 用于验证，生产环境建议 256 或以上
remote_url: "mooncakestore://{IP_of_Machine_A}:50052/"
remote_serde: "naive"
local_cpu: False
max_local_cpu_size: 2
numa_mode: null 

extra_config:
  local_hostname: "{IP_of_Machine_B}" # Prefiller 所在节点 IP
  metadata_server: "http://{IP_of_Machine_A}:8080/metadata"
  protocol: "rdma"
  device_name: "mlx5_0"
  master_server_address: "{IP_of_Machine_A}:50052"
  global_segment_size: 32212254720
  local_buffer_size: 1073741824
  transfer_timeout: 1
  save_chunk_meta: False
```

**Step 3: 启动 vLLM 实例**

**Decoder (Machine A):**
```bash
LMCACHE_CONFIG_FILE=mooncake-decoder-config.yaml \
LMCACHE_USE_EXPERIMENTAL=True \
VLLM_ENABLE_V1_MULTIPROCESSING=1 \
vllm serve Qwen/Qwen2.5-7B-Instruct-GPTQ-Int4 \
    --port 8200 \
    --kv-transfer-config '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_consumer"}'
```

**Prefiller (Machine B):**
```bash
LMCACHE_CONFIG_FILE=mooncake-prefiller-config.yaml \
LMCACHE_USE_EXPERIMENTAL=True \
VLLM_ENABLE_V1_MULTIPROCESSING=1 \
vllm serve Qwen/Qwen2.5-7B-Instruct-GPTQ-Int4 \
    --port 8100 \
    --kv-transfer-config '{"kv_connector":"LMCacheConnectorV1", "kv_role":"kv_producer"}'
```

**Step 4: 启动 Proxy Server**

使用 LMCache 提供的 `disagg_proxy_server.py` 进行请求路由。

```bash
# TODO: 确认 wait_decode_kv_ready(req_id) 是否仍需注释掉 (参考 LMCache#1342)
python3 disagg_proxy_server.py \
    --host localhost --port 9000 \
    --prefiller-host {IP_of_Machine_B} --prefiller-port 8100 \
    --decoder-host {IP_of_Machine_A} --decoder-port 8200
```

详细指南请参考：[vLLM V1 Disaggregated Serving with Mooncake Store and LMCache](https://github.com/kvcache-ai/Mooncake/blob/main/docs/source/getting_started/examples/vllm-integration/vllmv1-lmcache-integration.md)

## 4. 进阶配置与调优

### 4.1 网络配置 (RDMA vs TCP)

*   **TODO: 补充如何配置 RDMA 环境以获得最佳性能**
*   **TODO: 补充常见网络故障排查指南**

### 4.2 缓存策略

*   **TODO: 补充驱逐策略 (Eviction Policy) 的详细配置说明**

## 5. 参考资料

### 官方文档与代码
*   [**Mooncake GitHub Repository**](https://github.com/kvcache-ai/Mooncake): Mooncake 官方代码仓库，包含最新源码和 Release 信息。
*   [**Mooncake Documentation**](https://kvcache-ai.github.io/Mooncake/): 官方文档主页，涵盖架构设计、安装指南和 API 说明。

### 集成指南 (SGLang)
*   [**SGLang HiCache Quick Start**](https://github.com/kvcache-ai/Mooncake/blob/main/docs/source/getting_started/examples/sglang-integration/hicache-quick-start.md): SGLang HiCache 快速上手指南，包含基础配置和冒烟测试。
*   [**SGLang HiCache Complete Guide**](https://github.com/kvcache-ai/Mooncake/blob/main/docs/source/getting_started/examples/sglang-integration/hicache-integration-v1.md): SGLang HiCache 完整集成指南，包含深度架构解析和高级配置。
*   [**SGLang HiCache Integration Deep Dive**](https://zread.ai/kvcache-ai/Mooncake/16-sglang-hicache-integration): SGLang 与 Mooncake 集成的深度解析文章 (zread.ai)。

### 架构解析与场景实践
*   [**Mooncake Architecture Overview**](https://zread.ai/kvcache-ai/Mooncake/1-overview): Mooncake 核心架构与设计理念概览。
*   [**vLLM PD Disaggregation**](https://zread.ai/kvcache-ai/Mooncake/15-vllm-prefill-decode-disaggregation): Mooncake 与 vLLM 集成实现 Prefill-Decode 分离的实践文档。

### 社区讨论 (Issues)
*   [**SGLang Issue #15444**](https://github.com/sgl-project/sglang/issues/15444): 关于 Mooncake 集成的社区讨论与问题追踪。
*   [**SGLang Issue #14885**](https://github.com/sgl-project/sglang/issues/14885): 关于 HiCache 支持的技术细节讨论。

vLLM debug 建议
https://github.com/vllm-project/vllm/issues/13120

### LMCache 集成资料
*   [**vLLM V1 Disaggregated Serving with Mooncake Store and LMCache**](https://github.com/kvcache-ai/Mooncake/blob/main/docs/source/getting_started/examples/vllm-integration/vllmv1-lmcache-integration.md): Mooncake 与 LMCache 集成的官方示例指南。
*   [**Example: Share KV cache across multiple LLMs**](https://docs.lmcache.ai/getting_started/quickstart/share_kv_cache.html): LMCache 官方文档关于 KV Cache 共享（集中式与 P2P）的说明。
*   [**LMCache x Mooncake Blog**](https://blog.lmcache.ai/2025-05-08-mooncake/): 介绍 LMCache 与 Mooncake 联合推出的 KVCache-Centric 服务架构。
*   [**Mooncake | LMCache Docs**](https://docs.lmcache.ai/kv_cache/mooncake.html): LMCache 官方文档中关于 Mooncake 后端的详细配置说明。
