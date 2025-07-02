# iptables 端口屏蔽策略详解


本文档详细解析一套用于自动化管理 `iptables` 端口白名单的脚本，旨在建立一个安全的防火墙基线策略。该策略默认屏蔽一系列预定义端口，并仅对信任的 IP 地址（白名单）开放访问权限，是提升服务器安全、收缩攻击面的重要实践。

## 核心组件

自动化流程由以下关键文件组成：

*   `setup.sh`: 主执行脚本，负责调用核心配置脚本。
*   `config_port_filter.sh`: 核心的防火墙规则配置脚本。
*   `iptables`: 初始的 `iptables` 配置文件模板。
*   `../config/ips`: (非脚本) 存储白名单 IP 地址或网段的文本文件。

## 执行入口

部署防火墙策略的入口是 `setup.sh` 脚本。其核心操作是通过以下指令完成：

```bash
ips=$(cat ../config/ips)
bash -x ./config_port_filter.sh add $ips 10.244.0.0/16 10.10.0.0/16 172.17.0.0/16 127.0.0.1
```
1.  首先，脚本从 `../config/ips` 文件中读取用户定义的白名单 IP 列表。
2.  随后，将该列表及一系列硬编码的 IP/CIDR（通常用于内部服务或容器网络）作为参数，调用核心脚本 `config_port_filter.sh`，执行防火墙规则的 `add`（添加）操作。

> [!NOTE] 默认 IP/CIDR 含义解析
>
> 在 Kubernetes 集群环境中，这几个硬编码的 IP 地址和 CIDR 网段通常对应以下核心组件的默认网络范围。将它们加入白名单是为了确保集群内部的基础通信（如 Pod 间通信、节点与 Pod 通信）不会被防火墙阻断。
>
> *   `10.244.0.0/16`: **Pod 网段 (Pod CIDR)**.
>     *   这通常是 [Flannel](https://github.com/flannel-io/flannel) 网络插件为 Pod 分配 IP 地址的默认网段。集群中的每个 Pod 都会从这个地址池中获得一个唯一的 IP。允许此网段是为了保证不同节点上的 Pod 可以互相通信。
>     *   对于 [Calico](https://www.tigera.io/project-calico/)，默认的 Pod 网段通常是 `192.168.0.0/16`，但这个值是完全可以自定义的。此处的 `10.244.0.0/16` 表明环境很可能使用了 Flannel，或者是自定义了 Pod 网段的 Calico。
>
> *   `10.10.0.0/16`: **服务网段 (Service CIDR)**.
>     *   这是 Kubernetes `Service` 使用的虚拟 IP (ClusterIP) 地址范围。当 Pod 访问一个 Service 时，流量的目标 IP 会是这个范围内的地址，然后由 `kube-proxy` 组件通过 `iptables` 或 `IPVS` 规则将其转发到后端真实的 Pod。允许此网段对于集群内部的服务发现和负载均衡至关重要。
>
> *   `172.17.0.0/16`: **Docker 默认网桥 (docker0)**.
>     *   这是 Docker Daemon 在宿主机上创建的默认 `docker0` 网桥的 IP 地址范围。如果 Kubernetes 的容器运行时是 Docker，节点上的容器（包括一些系统 Pod）可能会通过这个网桥与宿主机或其他容器通信。虽然在现代的 CNI 插件（如 Calico, Flannel）管理下，Pod 间通信主要走 CNI 创建的虚拟网络，但保留这个白名单可以兼容一些边缘情况，或确保节点上非 Pod 化的 Docker 容器能与集群服务正常交互。
>
> *   `127.0.0.1`: **本地回环地址 (Loopback Address)**.
>     *   代表 `localhost`，即节点自身。许多运行在节点上的代理或守护进程（如 `kubelet`, `kube-proxy`）需要通过该地址与本地监听的服务（例如 API Server 的健康检查端点）通信。将其加入白名单是保证节点自身组件正常工作的基本要求。

> [!TIP] 如何确认实际的网络 CIDR？
>
> 上述 `10.244.0.0/16` 和 `10.10.0.0/16` 是 `kubeadm` 安装环境下的常见默认值，但实际生产环境中的配置可能不同。在 `kubectl` 无法使用的情况下，您可以通过直接检查主节点上的配置文件来确认准确的网段：
>
> *   **Pod 网段 (Cluster CIDR)**:
>     *   **文件**: `/etc/kubernetes/manifests/kube-controller-manager.yaml`
>     *   **查找**: 在 `command` 列表中寻找 `--cluster-cidr` 参数。
>
> *   **服务网段 (Service CIDR)**:
>     *   **文件**: `/etc/kubernetes/manifests/kube-apiserver.yaml`
>     *   **查找**: 在 `command` 列表中寻找 `--service-cluster-ip-range` 参数。
>
> *   **Docker 网段**:
>     *   **命令**: 在任意节点上运行 `ip addr show docker0` 查看 `docker0` 网桥的 IP。
>     *   **文件**: 检查 `/etc/docker/daemon.json` 文件中的 `bip` 配置。
>
> 对于非 `kubeadm` 安装的集群，这些配置可能位于 systemd 服务文件 (如 `/etc/systemd/system/kube-apiserver.service`) 中。在调整防火墙规则前，务必确认您环境中的实际网络配置。

## 核心逻辑：`config_port_filter.sh` 详解

脚本 `config_port_filter.sh` 承载了防火墙策略的核心逻辑，其设计遵循 **"白名单优先，默认拒绝"** 的安全原则。

### 1. 受管端口定义

脚本在顶部通过两个数组 `port_list` 和 `protocl_list` 硬编码了需要进行访问控制的端口及其协议。所有后续操作都将围绕这个列表展开。

### 2. 白名单规则生成 (`add_input_action` 函数)

当执行 `add` 操作时，函数 `add_input_action` 负责生成规则：

它遍历所有受管端口和所有传入的白名单 IP，为每个组合生成一条 `ACCEPT` 规则，并使用 `-I INPUT` 将其 **插入** 到 `iptables` 配置文件中 `INPUT` 链的定义处。使用 `-I` (insert) 至关重要，因为它能确保白名单规则置于 `INPUT` 链的顶部，被优先匹配。

```iptables
# 为白名单 IP 10.10.0.10 访问端口 6443 生成的规则示例
-I INPUT -s 10.10.0.10 -p tcp --dport 6443 -j ACCEPT
```

### 3. 默认拒绝规则

在为某个端口添加完所有白名单 IP 的 `ACCEPT` 规则后，脚本会紧接着为该端口 **追加** 一条默认的拒绝规则。

*   对于 TCP 协议，追加 `REJECT --reject-with tcp-reset` 规则。
*   对于 UDP 协议，追加 `DROP` 规则。

```iptables
# 针对端口 6443 的默认拒绝规则示例
-A INPUT -p tcp --dport 6443 -j REJECT --reject-with tcp-reset
```

> [!TIP]
> **为何 TCP 用 `REJECT` 而 UDP 用 `DROP`?**
> *   **TCP `REJECT`**: 当流量被拒绝时，防火墙会返回一个 `tcp-reset` 包。这能让合法的、但未在白名单中的客户端立即知道连接被拒，从而快速失败，避免不必要的等待超时。
> *   **UDP `DROP`**: UDP 是无连接协议。`DROP` 会静默丢弃数据包，不返回任何响应。这使得端口扫描工具难以确认端口是否开放，从而在一定程度上隐藏服务，提升安全性。

这种设计使得 `INPUT` 链中每个受管端口的规则集都遵循以下模式，确保了只有白名单流量被放行：
```
-I INPUT -s <白名单IP_1> ... -j ACCEPT
-I INPUT -s <白名单IP_2> ... -j ACCEPT
...
-A INPUT -p <协议> --dport <端口> -j REJECT/DROP  # 拒绝所有其他 IP
```

### 4. 规则持久化与生效
脚本通过 `sed` 命令直接修改 `/etc/sysconfig/iptables` 文件，完成规则的持久化。之后，通过 `systemctl restart iptables.service` 命令重启防火墙服务，使新规则立即生效。

> [!TIP]
> **`/etc/sysconfig/iptables` 详解**
>
> `iptables` 服务在启动时，会读取 `/etc/sysconfig/iptables` 文件（在像 CentOS/RHEL 这样的系统中），并使用 `iptables-restore` 命令将其中定义的规则加载到内核中，从而激活防火墙策略。
>
> 附录中提供的 `iptables` 文件内容是一个非常基础的模板，我们来逐行分析：
>
> ```text
> *filter
> :INPUT ACCEPT [0:0]
> :FORWARD ACCEPT [0:0]
> :OUTPUT ACCEPT [0:0]
> COMMIT
> ```
>
> *   `*filter`: 声明接下来的规则属于 `filter` 表。`filter` 表是 `iptables` 的默认表，专门负责数据包过滤。
> *   `:INPUT ACCEPT [0:0]`: 定义 `INPUT` 链（处理访问本机流量）的默认策略为 `ACCEPT`（接受）。这意味着，如果一个数据包没有匹配到任何具体规则，它将被无条件放行。
> *   `:FORWARD ACCEPT [0:0]`: 定义 `FORWARD` 链（处理流经本机流量）的默认策略为 `ACCEPT`。
> *   `:OUTPUT ACCEPT [0:0]`: 定义 `OUTPUT` 链（处理本机发出流量）的默认策略为 `ACCEPT`。
> *   `COMMIT`: 提交 `filter` 表的规则定义。
>
> **默认逻辑总结**
>
> 这份初始配置文件的逻辑是 **完全开放，不做任何限制**。所有入站、出站和转发的流量都会被默认允许。
>
> **与脚本的关联**
>
> 脚本正是在这个"裸奔"的模板之上进行工作的。它并 **不** 修改默认的 `ACCEPT` 策略，而是：
> 1.  **插入白名单规则**：使用 `-I INPUT` 将白名单 IP 的 `ACCEPT` 规则插入到 `INPUT` 链的 **最顶部**。
> 2.  **追加拒绝规则**：在所有白名单规则之后，使用 `-A INPUT` 追加一条针对特定端口的 `REJECT/DROP` 规则。
>
> 由于 `iptables` 规则是自上而下匹配的，来自白名单的流量会优先匹配 `ACCEPT` 规则而被放行。而其他所有访问受管端口的流量都会被后续的 `REJECT/DROP` 规则拦截。通过这种方式，脚本将一个完全开放的防火墙，转变成了一个仅对受管端口执行严格白名单策略的、更安全的状态。

> [!NOTE]
> **规则顺序为何如此重要？**
>
> 读者可能会注意到，脚本通过 `sed` 命令将所有新规则都插入到了 `:OUTPUT ACCEPT` 这一行之后。这个位置的选择和规则的顺序（`-I` vs `-A`）是保证策略正确执行的关键。
>
> **1. `iptables` 文件结构与注入点**
>
> `/etc/sysconfig/iptables` 文件本身不是脚本，而是一个供 `iptables-restore` 命令使用的数据文件。其基本结构必须遵循：
> 1.  以 `*<table_name>` 开始 (例如 `*filter`)。
> 2.  定义链的默认策略 (例如 `:INPUT ACCEPT`)。
> 3.  列出要添加的具体规则 (例如 `-A INPUT ...`)。
> 4.  以 `COMMIT` 结束。
>
> 脚本选择在 `:OUTPUT ACCEPT` 之后追加规则，是因为这是默认链定义的 **最后一个稳定锚点**。将规则注入到这里，可以确保它们被正确地放置在规则区域，且在 `COMMIT` 命令之前，`iptables-restore` 能够正确解析它们。
>
> **2. `-I` (插入) 与 `-A` (追加) 的逻辑**
>
> 当 `iptables-restore` 加载这些规则时，`-I` 和 `-A` 决定了规则在内核中 **实际生效的顺序**，这至关重要：
>
> *   **`-I INPUT ...`**: `-I` (Insert) 会将规则插入到 `INPUT` 链的 **最顶端** (位置 1)。脚本为所有白名单 IP 生成的都是 `-I` 规则。这意味着无论文件中这些规则的顺序如何，它们都会被优先加入链头。当一个数据包进入时，会 **最先** 匹配这些白名单规则。一旦匹配成功（例如，源 IP 在白名单中），数据包就会被 `ACCEPT`，处理流程结束。
>
> *   **`-A INPUT ...`**: `-A` (Append) 会将规则追加到 `INPUT` 链的 **末尾**。脚本为每个端口生成的 `REJECT/DROP` 规则都使用 `-A`。
>
> **组合效果：**
>
> 对于受管的某个端口（如 6443），最终在内核中的 `INPUT` 链规则会形成如下逻辑顺序：
>
> 1.  `-I INPUT -s <白名单IP_1> ... -j ACCEPT`  (最先检查)
> 2.  `-I INPUT -s <白名单IP_2> ... -j ACCEPT`  (其次检查)
> 3.  ... (其他无关规则) ...
> 4.  `-A INPUT -p tcp --dport 6443 -j REJECT` (在所有规则之后检查)
> 5.  `INPUT` 链的默认策略 `ACCEPT` (最后防线)
>
> 这种 "顶部插入白名单，底部追加黑名单" 的策略，精确地实现了 **"白名单优先通过，其他全部拒绝"** 的目标，而不会影响到其他端口（它们会穿过所有这些规则，最终被默认的 `ACCEPT` 策略放行）。

## 动态管理

可以直接调用 `config_port_filter.sh` 脚本来动态管理白名单 IP，而无需重新运行主脚本。

*   **添加白名单**:
    ```bash
    bash config_port_filter.sh add <IP_1> <IP_2> ... <CIDR_n>
    ```
*   **删除白名单**:
    ```bash
    bash config_port_filter.sh delete <IP_1> <IP_2> ... <CIDR_n>
    ```

> [!WARNING]
> **`delete` 功能存在严重设计缺陷**
> 脚本中的 `delete_input_action` 函数存在问题。它使用一个宽泛的 `sed` 命令来删除规则，这可能误删其他规则。更重要的是，它 **仅删除了 `ACCEPT` 规则，而没有移除相应的 `REJECT/DROP` 规则**。
>
> 这会导致一个危险的副作用：当一个 IP 从白名单中被删除后，由于末尾的 `REJECT/DROP` 规则依然存在，该端口会对 **所有 IP**（包括之前在白名单中的 IP）关闭。
>
> **在修复此缺陷前，请避免使用 `delete` 功能。** 推荐通过重新执行 `add` 命令（使用更新后的完整白名单）来覆盖现有配置，或手动编辑 `/etc/sysconfig/iptables` 文件进行精确管理。

## 附录

### `config_port_filter.sh`

```bash
#!/bin/bash

# 定义需要进行访问控制的端口列表。
# 例如，这里我们管理 25 (SMTP) 和 80 (HTTP) 端口。
port_list=(
25
80
)

# 为 port_list 中的每个端口定义对应的协议。
# 数组下标需要与 port_list 一一对应。
protocl_list=(
tcp
tcp
)

# 函数：添加防火墙规则
add_input_action(){
    # 遍历 port_list 中的每一个端口。
    for (( i = 0; i < ${#port_list[@]}; i++ )); do
        # 遍历所有作为参数传入的白名单 IP 地址或 CIDR。
        # 注意：循环从 j=1 开始，因为 param_list[0] 是操作指令 ('add' 或 'delete')。
        for (( j = 1; j < ${#param_list[@]}; j++ )); do
            # 获取 IP 地址或 CIDR。
            clusterpod_cidr=${param_list[$j]}
			# 检查 /etc/sysconfig/iptables 文件中是否已存在针对此 IP 和端口的 ACCEPT 规则。
			check_ip_exist=$(cat /etc/sysconfig/iptables | grep -w "${clusterpod_cidr} " | grep ${protocl_list[$i]} | grep -w ${port_list[$i]})
			# 如果规则不存在，则添加它。
			if [ -z "${check_ip_exist}" ];then
			    # 在 :OUTPUT ACCEPT 链定义之后插入 ACCEPT 规则。
			    # 这条规则允许来自指定 IP/CIDR 的流量访问指定的端口和协议。
                # 示例：sed -i "/:OUTPUT ACCEPT/a\-I INPUT -s 1.2.3.4 -p tcp --dport 80 -j ACCEPT" /etc/sysconfig/iptables
			    sed -i "/:OUTPUT ACCEPT/a\-I INPUT -s ${clusterpod_cidr} -p ${protocl_list[$i]} --dport ${port_list[$i]} -j ACCEPT"   /etc/sysconfig/iptables
			fi
        done
		
        # 为某个端口添加完所有白名单IP的 ACCEPT 规则后，追加一条默认的拒绝/丢弃规则。
        if [[ ${protocl_list[$i]} == "tcp" ]]; then
            # 对于 TCP 协议，检查是否已存在针对此端口的 REJECT 规则。
            check_tcp_exist=$(cat /etc/sysconfig/iptables | grep reject-with | grep tcp | grep -w ${port_list[$i]})
			# 如果 REJECT 规则不存在，则添加它。
			if [ -z "${check_tcp_exist}" ];then
			    # 追加一条规则，拒绝所有其他访问此 TCP 端口的流量，并返回 tcp-reset。
                # 示例：sed -i "/:OUTPUT ACCEPT/a\-A INPUT -p tcp --dport 80 -j REJECT --reject-with tcp-reset" /etc/sysconfig/iptables
			    sed -i "/:OUTPUT ACCEPT/a\-A INPUT -p ${protocl_list[$i]} --dport ${port_list[$i]} -j REJECT --reject-with tcp-reset"   /etc/sysconfig/iptables
			fi
        else
            # 对于其他协议（此处假定为 UDP），检查是否已存在 DROP 规则。
            check_udp_exist=$(cat /etc/sysconfig/iptables | grep DROP | grep udp | grep -w ${port_list[$i]})
			# 如果 DROP 规则不存在，则添加它。
			if [ -z "${check_udp_exist}" ];then
                # 追加一条规则，静默丢弃所有其他访问此 UDP 端口的流量。
                # 示例：sed -i "/:OUTPUT ACCEPT/a\-A INPUT -p udp --dport 53 -j DROP" /etc/sysconfig/iptables
                sed -i "/:OUTPUT ACCEPT/a\-A INPUT -p ${protocl_list[$i]} --dport ${port_list[$i]} -j DROP"   /etc/sysconfig/iptables
			fi
        fi
    done
    # 重启并启用 iptables 服务，使新规则生效。
	systemctl restart iptables.service
	systemctl enable iptables.service
}

# 函数：删除防火墙规则
delete_input_action(){
    # 遍历 port_list 中的每一个端口。
    for (( i = 0; i < ${#port_list[@]}; i++ )); do
        # 遍历所有作为参数传入的 IP 地址或 CIDR。
        for (( j = 1; j < ${#param_list[@]}; j++ )); do
            # 获取 IP 地址或 CIDR。
            clusterpod_cidr=${param_list[$j]}
            # 检查是否存在针对此 IP 和端口的 ACCEPT 规则。
            check_ip_exist=$(cat /etc/sysconfig/iptables | grep -w "${clusterpod_cidr} " | grep ${protocl_list[$i]} | grep -w ${port_list[$i]})
			# 如果规则存在，则删除它。
			if [ ! -z "${check_ip_exist}" ];then
			    # 此处的 sed 命令存在风险，它会删除任何包含该 IP/CIDR 和 "-p" 的行，可能误删其他规则。
                # 示例：要删除IP为 1.2.3.4 的规则，命令会是 sed -i "/1.2.3.4 \-p/d" /etc/sysconfig/iptables
                # 这可能意外删除该IP的其他端口规则。
			    sed -i "/${clusterpod_cidr} \-p/d"   /etc/sysconfig/iptables
            fi
        done
    done
    # 重启并启用 iptables 服务以应用变更。
	systemctl restart iptables.service
    systemctl enable iptables.service
}

# 将所有命令行参数存储到一个数组中。
param_list=($*)

# 检查参数数量是否少于2 (必须包含 'add'/'delete' 操作 和至少一个 IP)。
if [[ $# -lt 2 ]]; then
    # 如果参数不足，打印用法信息。
    echo "例如: bash config_port_filter.sh add 10.233.0.0/16 100.2.126.0/24"
    echo "例如: bash config_port_filter.sh delete 10.233.0.0/16 100.2.126.0/24"
    # 以错误码退出。
    exit 1
fi

# 第一个参数必须是 'add' 或 'delete'。
if [[ "$1" != "add" && "$1" != "delete" ]]; then
    # 如果操作无效，打印错误信息。
    echo "$0: 第一个参数 '$1' 必须是 'add' 或 'delete'"
    # 以错误码退出。
    exit 1
fi

# 根据第一个参数调用相应的函数。
if [ "$1" == "add" ]; then
    add_input_action
else
    delete_input_action
fi
```

### `iptables`

```text
# 这是一个 iptables 服务的示例配置文件。
# 它可以被手动编辑，也可以由 system-config-firewall 等工具管理。
# 不应在此处修改默认配置来添加额外的服务。
# 'filter' 表的开始，这是防火墙规则的默认表。
*filter
# 定义 INPUT 链策略。ACCEPT 表示默认允许所有传入流量。[0:0] 是数据包和字节计数器。
:INPUT ACCEPT [0:0]
# 定义 FORWARD 链策略。ACCEPT 表示默认允许所有转发流量。
:FORWARD ACCEPT [0:0]
# 定义 OUTPUT 链策略。ACCEPT 表示默认允许所有传出流量。
:OUTPUT ACCEPT [0:0]
# 提交 'filter' 表中的规则。
COMMIT
```
