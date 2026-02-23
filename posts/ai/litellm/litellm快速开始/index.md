# LiteLLM Proxy 快速开始：/v1/chat/completions 与限流 hooks 解析

LiteLLM 实现了生产级别的速率限制机制，支持每分钟请求数 (RPM)、每分钟 Token 数 (TPM)、并行请求以及基于预算的限制。该架构通过基于钩子的拦截点集成在代理层，允许在不修改请求/响应处理逻辑的情况下执行限制。

LiteLLM限流分为可以从维度、策略和配额三个角度进行说明：

- 维度分为：**组织** -- contains -> **团队** -- contains -> **个人** -- contains -> **Key**
    
    - 上述每个维度可以独立设置以下限流策略
        
- 限流策略：**单位时间****请求数（****RPM****）**、**单位时间Token 数(****TPM****)**、**并发请求量(Threads)**、~~预算($)~~
    
    - 不同维度的 RPM/TPM/并发会被顺序校验；
        
    - 预算校验独立于限流 hooks;
        
- 配额管理：**Best effort** **throughput** **(默认模式)、Guaranteed throughput (保证****吞吐量****模式)、Dynamic (动态模式)**
    
    - Best effort throughput：创建key时，下级可以设置上限累计超过上级RPM/TPM设定上限；
        
    - Guaranteed throughput：创建key时，下级的上限不能超过上级RPM/TPM设定上限；
        
    - Dynamic：请求到达时，在上次请求没有触发约束而失败的情况下，不对RPM/TPM进行校验
        

![](https://fcncpgawd4we.feishu.cn/space/api/box/stream/download/asynccode/?code=YmE3NzA3ODdjMWNhMWMzY2I1OTE5ZDRlNTgxY2FhZjNfbmtxdk5pbjNHRFo2QktEdUJnUEFNeFdFMVNkbXVQTlNfVG9rZW46UW83eGJSb1Q0b1ZjVlF4YndLb2MzU09wbkNiXzE3NzE4MTIyOTY6MTc3MTgxNTg5Nl9WNA)

  

This content is only supported in a Feishu Docs

1. ## Litellm 维度
    

用户和团队管理系统围绕由组织、团队和用户组成的三层层次结构构建。

- **组织** 代表最高级别的实体，通常对应于公司或企业客户。
    
- **团队** 是组织内部的逻辑分组，用于部门或基于项目的隔离。
    
- **用户** 是个人成员，可以同时属于多个团队和组织。
    

![](https://fcncpgawd4we.feishu.cn/space/api/box/stream/download/asynccode/?code=ZTgwZmIzMDZlMWI3MzI1OTM5MTEwMjNiNjQyNzU2NmFfUkpJNXpaU3ZIZWF1a2tNYnhBUnU5cXh0NUJrVkpCTDJfVG9rZW46TDdVT2JNUHJBb0g0Y0h4QjVYUmNoSXJwbjJlXzE3NzE4MTIyOTY6MTc3MTgxNTg5Nl9WNA)

### 组织（企业版特性）

组织是 LiteLLM 层次结构中的顶级容器。

- 每个组织都维护自己的**预算**、**模型访问权限**和**团队**集合。
    
- **开源版本不能使用组织功能**，组织通过 `organization_id` 进行标识。
    
- 组织与预算有直接关系，能够跨所有下属团队和用户实现集中化的支出控制。
    

### 团队

团队提供了 LiteLLM 内部的中间组织层。

- 团队隶属于组织并包含具有特定角色的成员。
    
- 每个团队维护自己的配置，包括允许的模型、速率限制（RPM/TPM）、预算约束和成员角色。
    
- 团队通过 `members_with_roles` JSON 字段支持管理员和普通成员角色，从而在团队边界内实现精细的权限控制。
    
- 团队还可以通过 `LiteLLM_ModelTable` 关系定义自定义模型别名。
    

### 用户

用户代表 LiteLLM 生态系统中的个人 API 消费者。

- `LiteLLM_UserTable` 条目包含全面的用户信息，包括唯一的 `user_id`、可选的 `user_email` 以及用于 SSO 集成的可选 `sso_user_id`。
    
- 用户通过 `LiteLLM_TeamMembership` 和 `LiteLLM_OrganizationMembership` 连接表分别在团队和组织中维护成员记录。
    
- 用户可以拥有独立的预算、模型访问限制和速率限制，而不受其团队或组织约束的限制。？？？
    

2. ## 限流策略
    

### 速率限制类型

LiteLLM 实现了四种不同的速率限制维度，每种都针对特定的资源约束。每分钟请求数 (RPM) 控制调用频率，而每分钟 Token 数 (TPM) 基于 Token 消耗管理计算容量。并行请求限制执行并发上限，防止同时请求耗尽资源。基于预算的限制在更长的时间范围内运行，控制每个提供程序、部署或标签的支出分配。

|   |   |   |   |   |
|---|---|---|---|---|
|限制类型|单位|执行点|时间窗口|Redis 支持|
|RPM|请求数|调用前钩子|60 秒滑动窗口|是 (Lua 脚本)|
|TPM|Token 数|调用前/后钩子|60 秒滑动窗口|是 (Lua 脚本)|
|并行请求|并发请求数|调用前/后钩子|请求生命周期|是 (计数器递增)|
|预算|货币金额|路由策略过滤器|可配置 (1d, 7d, 30d)|是 (管道)|

RPM 和 TPM 利用 Redis Lua 脚本实现滑动窗口，确保原子操作并防止竞态条件。并行请求计数器在请求到达时递增，在完成或失败时递减，从而维护准确的并发请求跟踪。预算限制作为路由过滤器运行，移除已超过支出分配的不健康部署。

### 限流实现方案

#### 限流 hook 函数调用路径

This content is only supported in a Feishu Docs

#### pre-call 阶段

1. **Call Type 分流**：
    
    1. 首先检查 `call_type`。例如，如果是批处理 Batch API (如`create_batch`)，则委托给 `BatchRateLimiter` 处理，避免普通请求的限流逻辑干扰批处理任务。
        

Call Type 根据类型可以分为批处理和普通接口：

批处理类型： create_batch、retrieve_batch、cancel_batch ......

普通接口： responses、completion、embedding ......

2. **动态限流前置检查 (Dynamic Rate Limiting)**：
    
    1. 读取用户 Metadata 中的 `rpm_limit_type` 或 `tpm_limit_type`。
        
    2. 如果配置为 `dynamic`，会调用 `_check_model_has_recent_failures` 检查目标模型在 Router 中是否有近期（1分钟内）的故障记录。如果有故障，后续会强制执行限流；如果没有故障，可能会允许突发流量（Best Effort）。
        

配额管理类型：

`rpm_limit_type`和`tpm_limit_type`具备相似的配额管理类型，分别是：**Best effort** **throughput** **(默认模式)、Guaranteed throughput (保证****吞吐量****模式)和Dynamic (动态模式)。**

3. **构建限流描述符 (Descriptors Construction)**：
    
    1. 调用 `_create_rate_limit_descriptors`，聚合所有维度的限流规则：
        
        - **API** **Key 维度**：RPM, TPM, Max Parallel Requests
            
        - **User / Team / End-User 维度**：RPM, TPM
            
        - **Model-specific 维度**：针对特定模型的 Key/Team 级限制
            
    2. 在此阶段，会根据动态限流的检查结果（`model_has_failures`）来决定是否真正 enforce 某些 limit。
        
4. **执行限流检查 (should_rate_limit)**：
    
    1. 它会将所有需要检查的规则转换为 Redis Key（如 `api_key:xxx:requests`）。
        
    2. 使用 Lua 脚本或 Pipeline 批量执行检查和计数（Incr/Get）。
        
    3. 如果任何一个维度的计数超过限制，返回 `OVER_LIMIT`。
        
5. **结果处理**：
    
    1. **超限**：直接抛出 `HTTPException(429)`，中止请求。
        
    2. **通过**：将限流结果（包括当前剩余额度）保存到 `data["litellm_proxy_rate_limit_response"]`。这些信息会在 `post_call_success_hook` 中被取出并写入 HTTP 响应头（`x-ratelimit-remaining-*`）。
        

This content is only supported in a Feishu Docs

#### 构建限流描述符

**构建限流描述符 (Descriptors Construction)**：

- 每个 descriptor = `{key, value, rate_limit}`，其中 `key` 表示维度、`value` 表示该维度的身份，`rate_limit` 可同时包含 RPM/TPM/并发三种策略。
    
- 典型的 descriptor 对象结构如下
    

```Python
RateLimitDescriptor = {
  "key": str,   # 维度名，例如 "api_key" / "user" / "team" / "model_per_key"
  "value": str, # 该维度下的具体身份如 user_id 或 team_id
  "rate_limit": {
    "requests_per_unit": Optional[int],
    "tokens_per_unit": Optional[int],
    "max_parallel_requests": Optional[int],
    "window_size": Optional[int],
  } | None
}
```

1. 调用 `_create_rate_limit_descriptors` 构造“基础 descriptors”（按是否配置限额决定是否加入）：
    
    1. **API** **Key** descriptor：`key="api_key"`, `value=<user_api_key_dict.api_key>`
        
        1. RPM：`requests_per_unit = enforced(rpm_limit, rpm_limit_type, model_has_failures)`
            
        2. TPM：`tokens_per_unit = enforced(tpm_limit, tpm_limit_type, model_has_failures)`
            
        3. 并发：`max_parallel_requests = user_api_key_dict.max_parallel_requests`
            
    2. **User** descriptor：`key="user"`, `value=<user_id>`
        
        1. RPM：`requests_per_unit = user_rpm_limit`
            
        2. TPM：`tokens_per_unit = user_tpm_limit`
            
    3. **Team** descriptor：`key="team"`, `value=<team_id>`
        
        1. RPM：`requests_per_unit = team_rpm_limit`
            
        2. TPM：`tokens_per_unit = team_tpm_limit`
            
    4. **Team Member** descriptor：`key="team_member"`, `value="{team_id}:{user_id}"`
        
    5. **End-User** descriptor：`key="end_user"`, `value=<end_user_id>`
        
    6. **Model-specific（per key）**：`key="model_per_key"`, `value="{api_key}:{requested_model}"`
        
        1. 仅当请求模型在该 key 的 `model_rpm_limit/model_tpm_limit` 配置中出现时才加入
            
        2. RPM：`requests_per_unit = model_rpm_limit[requested_model]`
            
        3. TPM：`tokens_per_unit = model_tpm_limit[requested_model]`
            
    7. **Model-specific（per team）**：`key="model_per_team"`, `value="{team_id}:{requested_model}"`
        
2. team descriptors（补充构造）：
    
    1. `_add_team_model_rate_limit_descriptor_from_metadata` 也可能追加 `model_per_team`（来源同样是 `team_metadata` 的 `model_rpm_limit/model_tpm_limit`）。这一步的效果是“为团队 + 指定模型”增加一条额外闸门；若与基础构造重复命中，会形成重复 descriptor。
        
3. organization descriptors（组织级构造）：
    
    1. **Organization（全局）**：`key="organization"`, `value=<org_id>`
        
        1. RPM：`requests_per_unit = organization_rpm_limit`
            
        2. TPM：`tokens_per_unit = organization_tpm_limit`
            
    2. **Model-specific（per organization）**：`key="model_per_organization"`, `value="{org_id}:{requested_model}"`
        
        1. 仅当请求模型在组织 `organization_metadata.model_rpm_limit/model_tpm_limit` 中出现时才加入
            
        2. RPM：`requests_per_unit = org_model_rpm_limit[requested_model]`
            
        3. TPM：`tokens_per_unit = org_model_tpm_limit[requested_model]`
            
4. Dynamic enforce 规则，及该限额是否生效（**`enforced`****为True生效**）：
    
    1. `should_enforce(limit_type) = (limit_type != "dynamic") OR model_has_failures`
        
    2. `enforced(limit_value) = limit_value if should_enforce(limit_type) else None`
        
    3. 当前仅对 `api_key` 的 `requests_per_unit/tokens_per_unit` 应用该规则；其他维度若配置了限额则始终 enforce。
        

#### 限流检查

- 典型的 descriptor 对象结构如下
    

```Python
RateLimitDescriptor = {
  "key": str,   # 维度名，例如 "api_key" / "user" / "team" / "model_per_key"
  "value": str, # 该维度下的具体身份如 user_id 或 team_id
  "rate_limit": {
    "requests_per_unit": Optional[int],
    "tokens_per_unit": Optional[int],
    "max_parallel_requests": Optional[in`t],
    "window_size": Optional[int],
  } | None
}
```

- 遍历 descriptors 列表中的每个 descriptor 对象，取出 descriptor 进行以下操作：
    

1. **遍历 descriptors 列表 -> 构造的****`keys_to_fetch`****列表用于查询****`key_metadata`****字典用于校验**：
    
    1. 为每个 descriptor 构造`window_key`，使用相同的窗口大小：
        
        1. `window_key = "{<descriptor_key>:<descriptor_value>}:window"`
            
        2. 如 `"{user:7fcbcb9c-9ada-45f6-91e8-0c06481d253d}:window"`
            
    2. 根据该 descriptor 开启了哪些维度的限流，构造对应维度的计数器 key：
        
        1. RPM：`counter_key = "{<descriptor_key>:<descriptor_value>}:requests"`
            
        2. TPM：`counter_key = "{<descriptor_key>:<descriptor_value>}:tokens"`
            
        3. 并行请求：`counter_key = "{<k>:<v>}:max_parallel_requests"`
            
        4. 如 `"{user:7fcbcb9c-9ada-45f6-91e8-0c06481d253d}:tokens"`
            
    3. 把所有要读取/更新的 key 扁平化成一个列表 `keys_to_fetch`，并且严格按照“window_key, counter_key”成对排列：
        
        1. `keys_to_fetch = [window_key_1, counter_key_1, window_key_2, counter_key_2, ...]`
            
        2. 后续所有批量操作都依赖这个顺序（每次步进 2）。
            
    4. 同时为每个 `window_key` 记录元信息 `key_metadata[window_key]`（限额、窗口大小、descriptor 类型），用于最终判定与生成返回结构。
        
        1. 目前实现里会读取 `rate_limit.window_size` 写入元信息，但真正执行 Redis/L1 更新时统一使用 `self.window_size`（也就是全局窗口）；
            
        2. 因此“每个 descriptor 单独 window_size”在这个函数里不会生效，只用于记录/展示。
            

最终形成 1 个列表和 1 个字典

```Python
# keys_to_fetch
[
  window_key, requests_key,
  window_key, tokens_key,
  window_key, max_parallel_key,
  ...
]
# key_metadata
{
  window_key: {
    "requests_limit": Optional[int],
    "tokens_limit": Optional[int],
    "max_parallel_requests_limit": Optional[int],
    "window_size": Optional[int],
    "descriptor_key": str # 维度名，例如 "api_key" / "user" / "team" / "model_per_key"
  },
  ...
}
```

2. **根据****`keys_to_fetch`****读取当前requests、tokens、max parallel requests数据**：
    
    1. 先读本地内存缓存：
        
        1. 若本地内存缓存命中，且任何维度已超限，立即返回 `OVER_LIMIT`，避免再去访问 Redis。
            
    2. 若本地内存缓存**未超限（或未命中）**，访问 Redis读取或读写相关的：
        
        1. 该分支与多实例相关，使用redis在多实例间同步
            
        2. 如果本地缓存未超过限制，就必须检查redis获取最新的数据
            
3. **基于滑动窗口校验是否超限（is_cache_list_over_limit）**：
    

缓动窗口是**以 `window_start` 为起点、长度为 `window_size` 的****时间片**。

对于每个限流计数器（由descriptor平铺后构建的counter），当系统读取到当前时间 `now` 时，会做一次简单判断：

- 若 `window_start` 不存在，或 `now - window_start >= window_size`，则认为**窗口已过期**。
    
- 窗口过期后会**重置**：把 `window_start` 设置为 `now`，`counter` 重置为 1（当前请求），并给两者设置同样的 TTL（`window_size`）。
    

1. 超限判定：
    
    1. **RPM****（requests）**：
        
        - pre-call 通过 `should_rate_limit` 对 `:requests` 计数器做滑动窗口自增与判定；
            
        - 超限条件是 `current_counter > limit`。
            
        - 窗口时间由 `LITELLM_RATE_LIMIT_WINDOW_SIZE` 决定，默认 60 秒。
            
    2. **TPM****（tokens）**：
        
        - pre-call 同样走 `should_rate_limit` 统一校验，但此时窗口内的 `:tokens` 计数只做“占位式自增”（每次请求 +1）。
            
        - 真正的 token 记账发生在请求成功后的回调中：根据响应 `usage` 计算实际 token 数，再用管道/Lua 脚本将 `:tokens` 计数器按真实值增量写入，并保持原有 TTL。
            
    3. **并发（max_parallel_requests）**：
        
        - 也是同一套“窗口计数器”机制，但只对 `api_key` 生效。
            
        - pre-call 中把 `max_parallel_requests` 作为独立计数器加入校验并自增；
            
        - 请求结束时（成功或失败）都会执行 `-1` 回收，从而把计数值作为“当前并发中的请求数”。
            
2. 对 `keys_to_fetch` 中的每一个 `(window_key, counter_key)` 按顺序生成一条 status，并根据status确定 `overall_code`为`OVER_LIMIT`还是`OK`。
    
3. `overall_code`决策逻辑：
    
    1. 从第 1 个 pair 开始逐个判定；一旦发现任意 pair 超限，会将 `overall_code` 置为 `OVER_LIMIT`，但仍会为所有 pair 生成 `statuses` 列表（便于上层定位是哪个维度/哪个维度触发）。
        

Tips:

- `read_only=True` 时不做递增，因此“当前请求是否会导致超限”的语义变成“当前计数是否已超限”；用于观测或预检更合适。
    
- `limit_remaining` 可能为负数（当 `used > limit` 时），上层可以据此计算超额量。
    

3. ## 配额管理
    

- **Best effort** **throughput** **(默认模式)**
    
    - 最佳吞吐量 —— 创建Key时，如果超额分配了 TPM/RPM，不会报错。
        
    - 团队（或组织）有一个总的 `tpm_limit/rpm_limit`，但你仍可以继续给下属 Team/Key 配置更高或累计更高的 `tpm_limit/rpm_limit`；系统不会在创建/更新时拒绝，只在运行时按每个维度的 descriptor 去拦截。
        
- **Guaranteed** **throughput** **(保证****吞吐量****模式)**
    
    - 保证吞吐量 —— 创建Key时，如果超额分配了 TPM/RPM，则会引发错误（同时也会检查特定模型的限制）。用于防止管理员意外地为密钥分配过多的 TPM/RPM 限制。
        
    - 不允许“超额分配”（配置期做 allocation 校验），若超额直接报错；同时会校验 model-specific 限额分配。
        
- **Dynamic (动态模式)**
    
    - 动态调整 —— 如果密钥设置了 TPM（例如 2 TPM）且未出现 429 错误（限流错误），它可以动态地突破所有维度的TPM/RPM限制，如达到3 TPM。
        
    - 在运行时按“上次请求是否出现限流错误”切换是否 enforce（当前仅对 API Key 的 RPM/TPM 生效）。
        
    - `model_has_failures` 通过 router 的 deployment failures（当前分钟）计算得到；若失败数超过阈值则认为模型不健康设置enforce并收紧限流。
        

4. ## 产品集成方案
    

### 用户管理的集成

实现一个限流网关，针对TPS、RPS/并发数量限制。

现有以下2种方案：

- 包含关系:（1）用户组；（2）用户；（3）Key；（4）Key-Models
    
- ~~独立关系：（1） Global-Models；（2）用户组；（3）用户；（4）Key；~~
    

  

```Python
# 方案一
UserGroup
  └── User
        └── Key
              └── Request(model)
# 方案二
Model
UserGroup
  └── User
        └── Key
```

This content is only supported in a Feishu Docs

### Litellm 方案： 基础方案

- **内存****存储和数据库存储**
    

将用量信息存储在内存；将用量限制存储在数据库：

- 请求到达时，访问一次数据库获取：（1）所有维度的约束；（2）模型路由信息；
    
- 将所有维度的约束进行平铺展开，并计算对应的哈希值；
    
- 获取内存中对应约束的时间窗口，并记录用量信息，任一个维度校验失败则不放行请求；
    
    - 内存中的时间窗口数量：约束类型数量 * （用户组数 + 用户数 + Key数量 *（1 + 模型数））
        
    - 即使某个约束失败，仍完整遍历约束，并记录哪些约束不通过
        
    - 返回提示信息“请求不满足[(用户组/用户/Key/Model 的 TPM/RPM 限制).*]，请稍后尝试”
        
- 校验通过后用1个Token占位；
    
- 请求结束并返回后，post_request线程获取usage字段更新用量信息，usage = usage + (实际用量 - 1)
    

  

This content is only supported in a Feishu Docs

复用 Descriptor 结构，将限制列表/字典作为全局变量：

- 校验 key 和查找 route 规则时，连表查看用户配置的RPM、TPM、Max Parallel Requests上限；
    
- 内存缓存滑动窗口内的RPM、TPM、Max Parallel Requests数值；
    
- 采用类似 litellm 的 key 映射规则，进行查找和比对；
    
- 超过滑动窗口部分数据抛弃；
    
- 滑动窗口过期时，重置该 key 的窗口；
    

  

**并发量计算：**

并发校验：

- pre-call：请求到达时，校验并增1计数；
    
- post-call：请求结束（成功/失败）时，减1计数；
    
- 请求到达时，尝试 +1 占位，如果超过限制，则 -1 并返回错误信息；
    

  

**滑动窗口算法**

滑动窗口应用于 TPM 或 RPM 这类单位时间内约束上限的场景：

- 场景1： 约束TPM 500 ，已记录时间窗口已用 450 + (10) Tokens，其中括号内为时间窗口外的Tokens数。
    
    - 新请求到达后，根据当前时间重新计算时间窗口，已用变为 450
        
    - 新请求尝试 +1 占位变为 451，校验通过；
        
    - 新请求结束，更新Usage占用为 100；
        
    - 当前用量为 451 + (100 - 1) = 550；
        
    - 下一请求到达时，因用量超过500，将被阻塞；
        

This content is only supported in a Feishu Docs

### DataBricks 方案：预处理占位及后处理矫正优化

在 litellm 中，任何请求到达后均增加 1 个 Token 进行占位，并在后处理中，将占位的用量修正为实际用量（即 usage = usage + (实际用量 - 1) ）。这引入了以下问题：

- **Litellm 预处理（pre-request）的占位用量与实际用量不符合。在突发流量时，容易突破用量限制。**
    

解决方案：

- **Input** **Tokens 估算** ：尝试从请求中估计的实际 Prompt 消耗的 Tokens 数量。
    
- **Output** **Tokens 估算** ：如果您在请求中提供 `max_tokens` ，使用此值来估算和预留输出令牌容量。
    
- **预准入验证** ：在处理请求之前检查您的请求是否会超出 TPM 或 RPM 限制。如果 `max_tokens` 会导致您超出 TPM 限制，将立即拒绝该请求并返回 429 错误。
    
- **实际用量修正预估用量** ：响应生成后，系统会统计实际输出的令牌数量。 **如果实际使用的令牌少于预留的** **`max_tokens`** **数量，将差额返还到速率限制额度中** 。
    

（参考： [databricks](https://docs.databricks.com/gcp/en/machine-learning/foundation-model-apis/limits#how-limits-are-tracked-and-enforced)）

  

### agentgateway 方案：Token计算方案

在 DataBricks 方案中提供了相对完整的解决方案，用于处理突发流量突破用量限制的问题。但是并没有给出具体的 Input Tokens 估算方案。

- DataBricks **预处理（pre-request）的提示词如何进行估算。**
    

解决方案：

- 通过在 **预处理（pre-request） 阶段调用 `/tokenize` 接口获得提示词的用量；**
    
- 但这引入了一定量的 TTFT 延迟，因为在将请求发送给 vLLM 前需要调用接口确定真实用量信息。
    
- 通过设置 `tokenize: true` 或 `false` 开关，让用户决定是否愿意放弃一定的 TTFT 性能，来换取 RPM 的精确限制；
    
- _在未设置_ `tokenize: true` 或将其设置为 `false` 时，无法计算请求使用的令牌数量。因此，除非速率限制设置为 0 个令牌，否则请求始终会被允许，并在后处理（post-request）时，进行核算（即回退到litellm模式）。
    

（参考：[agentgateway](https://agentgateway.dev/docs/local/latest/configuration/resiliency/rate-limits/)）

  

### krakend方案：滑动窗口 or 令牌桶-如何处理超量惩罚？

在 agentgateway 方案中给出具体的 Input Tokens 估算方法，但是如果关闭开关，回退到 litellm 模式时，依然存在短时突发流量突破限制的风险。

- 如何对超量/超前消费进行惩罚？
    

解决方案：

- 带有负数的令牌桶
    

  

滑动窗口算法实际上是固定窗口算法和漏桶算法的变体。它使用一个动态的时间窗口，并限制该窗口内请求的数量。由于窗口会定期更新，且请求速率理想情况下会均匀分布在一段时间内，因此滑动窗口算法能够提供更精细、更有效的速率限制，从而更好地控制流量方向和发生频率。

![](https://fcncpgawd4we.feishu.cn/space/api/box/stream/download/asynccode/?code=OGZmMjI1N2Q5MjEyNjVhMjk1NTYyZjAzNjQ4ZmY2OTlfeU1xVFBBQkR2bXhOOEt5V3B2aUVyNE9EclVmNVBab3JfVG9rZW46R25qOGJCZW5qbzlQb2d4UW52aGNob21Ebk5nXzE3NzE4MTIyOTY6MTc3MTgxNTg5Nl9WNA)

![](https://fcncpgawd4we.feishu.cn/space/api/box/stream/download/asynccode/?code=ODc5MTRhYTYyYTI0NDIwNWQ3YmY0MDAwYjcxNTliZTNfbWdSdkNVaDF4M2VYZXc2ZUtVWGsxdkZWZ1FOQ2EzaGNfVG9rZW46QnY3dmI3ejZYb0VFTUZ4MTZaTGM5Y2RMblJjXzE3NzE4MTIyOTY6MTc3MTgxNTg5Nl9WNA)

令牌桶算法通过不断生成新令牌来控制数据传输量，并将这些令牌放入令牌桶中。每个请求都需要一个令牌，如果请求者没有任何令牌，则其请求将被拒绝。该算法使系统能够处理不同强度的数据流，并可在给定的时间间隔内控制请求速率。

![](https://fcncpgawd4we.feishu.cn/space/api/box/stream/download/asynccode/?code=YzI1YjA4NGJiZTg3YTQ5ZTgxZTNlNDUxYzkyZmVlMWFfQWlmaExVaWppdU9ObmFvWUxGWDhsemlFM3IyQmlJNXJfVG9rZW46UDQxMGJ5VERNb0pweDF4ZjJxbmM1T2tmbmxjXzE3NzE4MTIyOTY6MTc3MTgxNTg5Nl9WNA)

  

（参考：[krakend](https://www.krakend.io/docs/throttling/token-bucket/) 、 [算法说明](https://www.geeksforgeeks.org/system-design/rate-limiting-algorithms-system-design/)）

  

TODO:

- 与 vllm 联动方案调研
    

## 附录

1. ### hooks 是怎么被注册进回调链的（初始化）
    

2. 代理启动时创建全局 proxy_logging_obj = ProxyLogging(...)
    

3. 见 proxy_server.py:L1301-L1303
    

4. 启动事件里调用 proxy_logging_obj.startup_event(...) ，其内部会：
    
      
    
    2. self._init_litellm_callbacks(llm_router=llm_router)
        
    3. 进而 self._add_proxy_hooks(...) 见 utils.py:L332-L349
        
5. _add_proxy_hooks() 会遍历 PROXY_HOOKS ，实例化每个 hook，并 add_litellm_callback() 加入 litellm.callbacks
    

6. 见 utils.py:L423-L442
    

7. PROXY_HOOKS 字典来自 hooks/ init .py ：
    
      
    
    2. 默认把 "parallel_request_limiter" 绑定到 _PROXY_MaxParallelRequestsHandler_v3 见 init .py:L20-L26
        
    3. 若 LEGACY_MULTI_INSTANCE_RATE_LIMITING=true ，则切回旧版 _PROXY_MaxParallelRequestsHandler 见 init .py:L29-L31 结论： 并发/RPM/TPM 限流 hook（v3）是在启动时被放进 litellm.callbacks 回调链的 ，后续每次请求都会被 ProxyLogging.pre_call_hook() 迭代调用。
        

```Python
# List of all available hooks that can be enabled
PROXY_HOOKS = {
    "max_budget_limiter": _PROXY_MaxBudgetLimiter,
    "parallel_request_limiter": _PROXY_MaxParallelRequestsHandler_v3,
    "cache_control_check": _PROXY_CacheControlCheck,
    "responses_id_security": ResponsesIDSecurity,
    "litellm_skills": SkillsInjectionHook,
}
```

2. ### /v1/chat/completions 请求入口到 hooks 的调用栈（pre-call 限流）
    

#### 主调用栈（到达“限流 hook 的 async_pre_call_hook”）

1. FastAPI 路由入口： chat_completion()
    

2. proxy_server.py:L5534-L5624
    

3. chat_completion() 调 ProxyBaseLLMRequestProcessing.base_process_llm_request(route_type="acompletion")
    

4. 同上 proxy_server.py:L5605-L5624
    

5. base_process_llm_request() 先执行 common_processing_pre_call_logic()
    

6. common_request_processing.py:L714-L729
    

7. common_processing_pre_call_logic() 内部关键点：
    
      
    
    2. 先 litellm.utils.function_setup(...) 建立 logging_obj 并塞进 data["litellm_logging_obj"]
        
    3. 然后调用 proxy_logging_obj.pre_call_hook(user_api_key_dict, data, call_type=route_type)
        
    
       common_request_processing.py:L602-L618
    
8. ProxyLogging.pre_call_hook() 会遍历 litellm.callbacks ，对每个实现了 async_pre_call_hook 的 CustomLogger 执行： await _callback.async_pre_call_hook(user_api_key_dict, cache, data, call_type)
    

9. utils.py:L1205-L1249
    

10. 其中限流 hook（v3）就是 _PROXY_MaxParallelRequestsHandler_v3.async_pre_call_hook()
    

11. parallel_request_limiter_v3.py:L1123-L1209
    

### 3) v3 限流 hook 内部：并发 / RPM / TPM(TPS) 分别在哪里触发

#### A) 并发限制（max_parallel_requests）+ RPM 检查：在 pre-call

#### B) TPM / “TPS”(token per minute) 计数：不在 pre-call 增加，而在成功回调里增加

v3 的设计是： pre-call 阶段主要做 RPM/并发 gate；token 消耗要等到模型响应出来才能知道，因此在 success event 里增量 tokens。

  

对应调用栈（成功路径）：

  

1. 路由仍是 chat_completion() → base_process_llm_request()
    
2. 在 base_process_llm_request() 中，请求会被 route_request(...) 真的发到上游模型（略）
    

3. common_request_processing.py:L748-L756
    

4. 拿到 response 后，框架会调用 proxy_logging_obj.post_call_success_hook(...) （用于让 callbacks 修改 response/headers 等）
    

5. common_request_processing.py:L879
    

6. 其内部会遍历回调并调用 callback.async_post_call_success_hook(...)
    

7. utils.py:L1801-L1809
    

8. v3 限流 hook 的 async_post_call_success_hook() 负责把限流结果写入响应 header（基于 data["litellm_proxy_rate_limit_response"] ）
    

9. parallel_request_limiter_v3.py:L1595-L1642
    

10. 真正的 token 增量 是在 async_log_success_event() （这是 litellm 的 logging callback 生命周期事件，不是 proxy 的 post_call_success_hook）里完成：
    
      
    
    2. 从 response 的 usage 里算出 tokens
        
    3. 对 tokens 相关 keys 执行 INCRBYFLOAT （Redis pipeline/Lua），并对并发计数做 -1
        
    
       parallel_request_limiter_v3.py:L1385-L1542
    

其中关键点：

  

- get_rate_limit_type() 决定用 output/input/total tokens 作为“TPM”口径
    

parallel_request_limiter_v3.py:L1371-L1384

- _get_total_tokens_from_usage() 处理 cached_tokens 的扣除等细节
    

parallel_request_limiter_v3.py:L1238-L1290

  

### 参数

分类位于： `external/litellm/litellm/types/utils.py(253): CallTypes`

|   |   |   |
|---|---|---|
|参数名|类型|说明|
|duration|Optional[str]|指定 Token 的有效时长。支持秒（30s）、分钟（30m）、小时（30h）、天（30d）。|
|key_alias|Optional[str]|用户自定义的 Key 别名。|
|key|Optional[str]|用户自定义的 Key 值；如果未设置，将自动生成一个 16 位唯一的 sk- Key。|
|team_id|Optional[str]|该 Key 所属的团队 ID。|
|user_id|Optional[str]|非功能性参数（将被忽略），原本用于指定 Key 的用户 ID。|
|budget_id|Optional[str]|与 Key 关联的预算 ID，通过 /budget/new 接口创建。|
|models|Optional[list]|该 Key 允许调用的模型列表；为空则表示可调用所有模型。|
|aliases|Optional[dict]|模型别名映射，在 config.yaml 之外追加的别名配置。参考文档：[https://docs.litellm.ai/docs/proxy/virtual_keys#managing-auth---upgradedowngrade-models](https://docs.litellm.ai/docs/proxy/virtual_keys#managing-auth---upgradedowngrade-models)|
|config|Optional[dict]|Key 级别的配置，用于覆盖 config.yaml 中的全局配置。|
|spend|Optional[int]|该 Key 已产生的消费金额，默认值为 0；每次使用 Key 时由 Proxy 自动更新。|
|send_invite_email|Optional[bool]|是否向 user_id 发送包含生成 Key 的邀请邮件。|
|max_budget|Optional[float]|为该 Key 指定最大预算上限。|
|budget_duration|Optional[str]|预算重置周期；到期后预算自动重置。不设置则永不重置。支持 s/m/h/d。|
|max_parallel_requests|Optional[int]|并发请求数限制；当并发请求数超过该值时返回 429 错误。|
|metadata|Optional[dict]|Key 的元数据，用于存储附加信息，例如团队、应用、邮箱等。示例：{"team": "core-infra", "app": "app2", "email": "[xxx@xxx.com](mailto:xxx@xxx.com)"}|
|guardrails|Optional[List[str]]|该 Key 启用的安全护栏（Guardrails）列表。|
|permissions|Optional[dict]|Key 级别权限配置，目前主要用于关闭 PII 脱敏等功能。示例：{"pii": false}|
|model_max_budget|Optional[Dict[str, BudgetConfig]]|模型级别预算配置，例如：{"gpt-4": {"budget_limit": 0.0005, "time_period": "30d"}}；为空或 {} 表示不启用模型级预算。|
|model_rpm_limit|Optional[dict]|模型级 RPM 限制（每分钟请求数）。为空或 {} 表示不限制。|
|model_tpm_limit|Optional[dict]|模型级 TPM 限制（每分钟 Token 数）。为空或 {} 表示不限制。|
|tpm_limit_type|Optional[str]|TPM 限流策略类型：best_effort_throughput、guaranteed_throughput、dynamic。|
|rpm_limit_type|Optional[str]|RPM 限流策略类型：best_effort_throughput、guaranteed_throughput、dynamic。|
|allowed_cache_controls|Optional[list]|允许的 Cache-Control 值列表。示例：["no-cache", "no-store"]|
|blocked|Optional[bool]|是否禁用（封禁）该 Key。|
|rpm_limit|Optional[int]|Key 级别的 RPM 限制（每分钟请求数）。|
|tpm_limit|Optional[int]|Key 级别的 TPM 限制（每分钟 Token 数）。|
|soft_budget|Optional[float]|软预算阈值；达到后会触发 Slack 告警，但不会阻断请求。|
|tags|Optional[List[str]]|Key 标签，用于消费统计或基于标签的路由。|
|enforced_params|Optional[List[str]]|强制参数列表（仅企业版支持），用于要求 LLM 请求必须包含指定参数。|
|allowed_routes|Optional[list]|允许访问的路由列表，支持精确路径或通配符。示例：["/chat/completions", "/embeddings", "/keys/*"]|
|object_permission|Optional[LiteLLM_ObjectPermissionBase]|Key 级对象权限控制，例如允许访问的向量库、Agent、Agent 组等；为空或 {} 表示不限制。|

### CallType 列表

|   |   |   |
|---|---|---|
|类别|CallType|说明|
|通用推理|responses|统一入口：chat / reasoning / tools|
|文本生成（兼容）|completion|老接口 / 兼容逻辑|
|向量化|embedding|RAG / 搜索 / 聚类|
|重排|rerank|检索质量核心|
|搜索|search|联网 / 内部搜索|
|文生图|image_generation||
|语音转文本|transcription|ASR|
|文本转语音|speech|TTS|
|OCR|ocr|图片 → 文本|
|创建 Batch|create_batch / acreate_batch|大规模离线|
|查询 Batch|retrieve_batch / aretrieve_batch||
|取消 Batch|cancel_batch / acancel_batch||
