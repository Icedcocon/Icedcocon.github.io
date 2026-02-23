# vllm 使用 API 操作 Scheduler 指南

## 概览

- 目标：梳理 V1 架构下，如何通过 HTTP API 驱动 Engine 与 Scheduler 的行为，并以“休眠”示例串联端到端调用路径。
- 结论：API 请求通过 FastAPI Router → EngineClient 协议 → AsyncLLM → EngineCoreClient（ZMQ，多进程）→ EngineCore（UTILITY 调度）→ Executor/Scheduler 实际执行。

> [!NOTE]
> 文章保留所有关键引用（文件路径、行号、函数名），并补充可点击的源码链接，便于交叉定位。

## 路由入口

- 路径：/Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/entrypoints/serve/
- 典型开发端点：sleep、lora、profile、tokenize 等

```Plain
entrypoints/serve
|-- disagg/api_router.py
|-- elastic_ep/api_router.py
|-- instrumentator/api_router.py
|-- lora/api_router.py
|-- profile/api_router.py
|-- rlhf/api_router.py
|-- sleep/api_router.py
`-- tokenize/api_router.py
```

- 休眠入口：[api_router.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/entrypoints/serve/sleep/api_router.py#L22-L29)

```python
# entrypoints/serve/sleep/api_router.py(22-29): sleep
@router.post("/sleep")
async def sleep(raw_request: Request):
    level = raw_request.query_params.get("level", "1")
    await engine_client(raw_request).sleep(int(level))
    return Response(status_code=200)
```

> [!WARNING]
> 这些开发端点在生产环境默认关闭，仅在开发模式启用（参见同文件中的 attach_router）。

## 客户端协议与流向

- 协议定义：[protocol.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/engine/protocol.py#L125-L128)

```python
# engine/protocol.py(125-128): sleep
@abstractmethod
async def sleep(self, level: int = 1) -> None:
    ...
```

- 客户端实现：[async_llm.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/async_llm.py#L747-L752)

```python
# v1/engine/async_llm.py(747-752): sleep
async def sleep(self, level: int = 1) -> None:
    await self.reset_prefix_cache()
    await self.engine_core.sleep_async(level)
    if self.logger_manager is not None:
        self.logger_manager.record_sleep_state(1, level)
```

- EngineCore 客户端抽象：[core_client.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core_client.py#L218-L219)
- Async 多进程实现：[core_client.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core_client.py#L977-L978)

```python
# v1/engine/core_client.py(977-978): sleep_async
async def sleep_async(self, level: int = 1) -> None:
    await self.call_utility_async("sleep", level)
```

> [!TIP]
> AsyncMPClient 通过 UTILITY 消息远程调用 EngineCore 的同名方法，是实现“控制面”的统一入口。

## MP 通信与请求类型

- 发送消息（ZMQ）：[core_client.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core_client.py#L910-L933)

```python
# v1/engine/core_client.py(910-933): _send_input_message
msg = (engine,) + message
return self.input_socket.send_multipart(msg, copy=False, track=True)
```

- UTILITY 封装与排队：[core_client.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core_client.py#L935-L936), [core_client.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core_client.py#L938-L950)

```python
# v1/engine/core_client.py(935-950): call_utility_async/_call_utility_async
async def call_utility_async(self, method: str, *args) -> Any:
    return await self._call_utility_async(method, *args, engine=self.core_engine)
```

- 请求类型枚举：

```python
# v1/engine/__init__.py(189): EngineCoreRequestType
class EngineCoreRequestType(enum.Enum):
    """
    Request types defined as hex byte strings, so it can be sent over sockets
    without separate encoding step.
    """
    ADD = b"\x00"
    ABORT = b"\x01"
    START_DP_WAVE = b"\x02"
    UTILITY = b"\x03"
    # Sentinel used within EngineCoreProc.
    EXECUTOR_FAILED = b"\x04"
```

## EngineCore 循环与 UTILITY 调度

- 主循环：[core.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core.py#L878-L914)
- 分发 UTILITY 调用：[core.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core.py#L928-L958)

```python
# v1/engine/core.py(878-914): run_busy_loop
def run_busy_loop(self):
    while True:
        self._process_input_queue()
        self._process_engine_step()
```

```python
# v1/engine/core.py(928-958): _handle_client_request
elif request_type == EngineCoreRequestType.UTILITY:
    client_idx, call_id, method_name, args = request
    method = getattr(self, method_name)
    result = method(*self._convert_msgspec_args(method, args))
    self.output_queue.put_nowait((client_idx, EngineCoreOutputs(utility_output=output)))
```

> [!NOTE]
> UTILITY 允许调用 EngineCore 的任意公开方法，是添加“控制接口”（如 sleep、wake_up、动态切换）的关键挂载点。

## Sleep 示例：执行链与落地位置

- EngineCore 的 sleep 方法与落地逻辑：[core.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core.py#L76-L216), [core.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core.py#L584-L677), [core.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core.py#L516-L516)

```python
# v1/engine/core.py(516): sleep
def sleep(self, level: int = 1):
    self.model_executor.sleep(level)
```

- Executor 中的集群广播与状态管理：v1/executor/abstract.py#L297-L270

```python
# v1/executor/abstract.py(297): sleep
self.collective_rpc("sleep", kwargs=dict(level=level))
self.sleeping_tags = {"weights", "kv_cache"}
self.is_sleeping = True
```

- 终止请求示例（ABORT 与 Scheduler）：[core.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core.py#L302-L302)

```python
# v1/engine/core.py(302): abort_requests
self.scheduler.finish_requests(request_ids, RequestStatus.FINISHED_ABORTED)
```

## Scheduler 关键判断引用

- 位置：调度等待队列阶段的“外部缓存匹配”判断
- 引用：[scheduler.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/core/sched/scheduler.py#L483-L491)

```python
# v1/core/sched/scheduler.py(483-491): schedule 外部缓存判断
# Get externally-cached tokens if using a KVConnector.
if self.connector is not None:
    ext_tokens, load_kv_async = self.connector.get_num_new_matched_tokens(
        request, num_new_local_computed_tokens
    )
```

> [!TIP]
> 只要 `self.connector` 存在，调度器就会尝试使用外部 KVConnector 进行匹配与异步加载判断，因此“动态开关”可通过使该引用返回 None 或在 Connector 层引入启停标志来实现。

---

## 实例：动态开关外部缓存（KVConnector）接口设计

目标：实现一个 HTTP 接口，允许运行中动态启用/禁用 Scheduler 的外部缓存（KVConnector），从而影响调度阶段的“外部匹配”与“异步 KV 载入”逻辑。

> [!NOTE]
> 本节仅给出完整代码方案与落点说明，不修改现有 *.py 文件。实现思路基于现有 UTILITY 调用链，新增一个控制方法与路由。

### 行为与范围

- 启用：`self.connector.enabled = True`；调度器在 [scheduler.schedule](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/core/sched/scheduler.py#L476-L506) 中走“外部缓存匹配”分支。
- 禁用：`self.connector.enabled = False`；调度器跳过分支，仅读取本地缓存；异步载入标志关闭。

### 变更点与代码方案（文档内示例）

1) 新增开发路由（建议路径：entrypoints/serve/connector/api_router.py）

文件：/Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/entrypoints/serve/connector/api_router.py

```python
from fastapi import APIRouter, FastAPI, Request
from fastapi.responses import JSONResponse, Response
from vllm.engine.protocol import EngineClient

def engine_client(request: Request) -> EngineClient:
    return request.app.state.engine_client

router = APIRouter()

@router.post("/kv_connector/toggle")
async def kv_connector_toggle(raw_request: Request):
    enabled = raw_request.query_params.get("enabled", "true").lower() == "true"
    await engine_client(raw_request).set_kv_connector_enabled(enabled)
    return Response(status_code=200)

@router.get("/kv_connector/enabled")
async def kv_connector_enabled(raw_request: Request):
    enabled = await engine_client(raw_request).is_kv_connector_enabled()
    return JSONResponse(content={"enabled": enabled})

def attach_router(app: FastAPI):
    app.include_router(router)
```

> [!WARNING]
> 与 sleep 路由一致，建议仅在开发模式下挂载此路由，避免生产误操作。

2) 扩展 EngineClient 协议（不修改现有方法，新增两项）

文件：/Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/engine/protocol.py

```python
class EngineClient(ABC):
    @abstractmethod
    async def set_kv_connector_enabled(self, enabled: bool) -> None: ...

    @abstractmethod
    async def is_kv_connector_enabled(self) -> bool: ...
```

3) 在 AsyncLLM 中实现协议并转发到 EngineCore（UTILITY）

文件：/Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/async_llm.py

```python
class AsyncLLM(EngineClient):
    async def set_kv_connector_enabled(self, enabled: bool) -> None:
        await self.engine_core.set_kv_connector_enabled_async(enabled)

    async def is_kv_connector_enabled(self) -> bool:
        return await self.engine_core.is_kv_connector_enabled_async()
```

4) 在 EngineCoreClient 中新增 UTILITY 封装方法

文件：/Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core_client.py

```python
class EngineCoreClient(ABC):
    async def set_kv_connector_enabled_async(self, enabled: bool) -> None: ...
    async def is_kv_connector_enabled_async(self) -> bool: ...

class AsyncMPClient(MPClient):
    async def set_kv_connector_enabled_async(self, enabled: bool) -> None:
        await self.call_utility_async("set_kv_connector_enabled", enabled)

    async def is_kv_connector_enabled_async(self) -> bool:
        return await self.call_utility_async("is_kv_connector_enabled")
```

> [!NOTE]
> 以上实现方式与现有 [sleep_async](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core_client.py#L977-L978) 完全一致，均走 UTILITY 通道。

5) 在 EngineCore 中实现控制方法（方案 B：保留对象，仅切换 enabled）

文件：/Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core.py

```python
class EngineCore:
    def set_kv_connector_enabled(self, enabled: bool) -> None:
        if self.scheduler.connector is None:
            self.scheduler.connector = KVConnectorFactory.create_connector(
                config=self.vllm_config,
                role=KVConnectorRole.SCHEDULER,
                kv_cache_config=self.scheduler.kv_cache_config,
            )
        self.scheduler.connector.enabled = enabled

    def is_kv_connector_enabled(self) -> bool:
        return bool(self.scheduler.connector and self.scheduler.connector.enabled)
```

> [!TIP]
> 方案 B 通过保留 connector 实例避免反复构建；核心是引入 `enabled` 标志并让调度判定依赖该标志。

6) Scheduler 的判定增强（方案 B 必需）

文件：/Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/core/sched/scheduler.py（示意）

```python
# 原判定
if self.connector is not None:
    ...
# 增强为（示例）
if (self.connector is not None) and getattr(self.connector, "enabled", True):
    ...
```

7) 在 KVConnector 基类中补充 enabled 属性（方案 B 必需）

文件：/Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/distributed/kv_transfer/kv_connector/v1/base.py

```python
class KVConnectorBase_V1:
    def __init__(self, ...):
        self.enabled = True
```

> [!NOTE]
> 若已有统一的 Connector 基类或配置项，可将 enabled 的初始化放在共同父类或工厂层。

### 接口使用示例

- 关闭外部缓存：

```bash
curl -X POST "http://localhost:8000/kv_connector/toggle?enabled=false"
```

- 开启外部缓存：

```bash
curl -X POST "http://localhost:8000/kv_connector/toggle?enabled=true"
```

- 查询状态：

```bash
curl "http://localhost:8000/kv_connector/enabled"
```

### 生效路径总览（端到端）

- Router → EngineClient → AsyncLLM → EngineCoreClient → EngineCore（UTILITY）→ Scheduler.connector
- 关键引用：
  - 路由：[sleep/api_router.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/entrypoints/serve/sleep/api_router.py#L22-L29)（参考实现风格）
  - 协议：[engine/protocol.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/engine/protocol.py#L125-L128)
  - 客户端：[async_llm.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/async_llm.py#L747-L752)
  - UTILITY 调用：[core_client.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core_client.py#L938-L950)
  - EngineCore 分发：[core.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/engine/core.py#L928-L958)
  - Scheduler 判定：[scheduler.py](file:///Users/admin/Documents/Docs/HugoBlogs/external/vllm/vllm/v1/core/sched/scheduler.py#L483-L491)

> [!NOTE]
> 选择“置空 connector”方案，可保证不触碰调度逻辑与判定语句，从而降低改动面；恢复时按原工厂创建即可。

---


