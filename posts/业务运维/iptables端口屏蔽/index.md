# iptables 端口屏蔽策略详解


本文档旨在详细解析用于服务器 `iptables` 防火墙端口屏蔽的一系列自动化脚本。这些脚本的目标是建立一个安全的防火墙基线，默认屏蔽一系列已知风险或非必要的服务端口，同时仅对信任的 IP 地址（白名单）开放访问权限。这对于维护服务器安全、减少攻击面至关重要。

## 核心组件

整个自动化流程由以下几个关键文件组成：

*   `setup.sh`: 主执行脚本，负责环境初始化和调用配置脚本。
*   `config_port_filter.sh`: 核心的防火墙规则配置脚本，负责添加或删除规则。
*   `iptables`: 初始的 `iptables` 配置文件模板。
*   `../config/ips`: (非脚本) 存储需要加入白名单的 IP 地址或网段的文本文件。

## 执行流程

部署防火墙策略的入口是 `setup.sh` 脚本，其执行流程如下：

1.  **环境准备**: 首先，脚本会清理旧的 `iptables` 服务并安装指定版本的 `rpm` 包，以确保环境的一致性。
    ```bash
    yum remove iptables-services -y
    yum localinstall ./rpm/*.rpm -y
    ```

> [!NOTE]
> 这一步表明该脚本是为特定的、基于 RPM 的 Linux 发行版（如 CentOS/RHEL）定制的。

2.  **重置配置**: 脚本会删除系统默认的 `iptables` 配置文件，并替换为一个基础模板 (`iptables` 文件)。该模板默认接受所有链（INPUT, FORWARD, OUTPUT）的流量。
    ```bash
    rm -rf /etc/sysconfig/iptables
    cp iptables /etc/sysconfig/iptables
    chmod 600 /etc/sysconfig/iptables
    ```

3.  **调用核心脚本**: `setup.sh` 从 `../config/ips` 文件中读取 IP 地址列表，并连同一系列硬编码的 IP/CIDR（如 `10.244.0.0/16`, `127.0.0.1` 等，通常用于内部服务或 Kubernetes/Docker 网络），作为参数传递给 `config_port_filter.sh` 脚本来添加防火墙规则。
    ```bash
    ips=$(cat ../config/ips)
    bash -x ./config_port_filter.sh add $ips 10.244.0.0/16 10.10.0.0/16 172.17.0.0/16 127.0.0.1
    ```

## 防火墙策略详解

核心逻辑封装在 `config_port_filter.sh` 中，它采用了一种 **“白名单优先，默认拒绝”** 的安全策略。

1.  **受管端口**: 脚本内置了一个需要进行访问控制的端口列表（`port_list`）及其对应的协议（`protocl_list`）。

2.  **规则生成**: 对于列表中的每一个端口/协议对，`add_input_action` 函数执行以下操作：
    *   **创建白名单规则**: 遍历所有传入的 IP 地址/CIDR，为每一个 IP 生成一条 `ACCEPT` 规则，并将其 **插入（-I）** 到 `INPUT` 链的定义中。这意味着来自这些白名单 IP 的流量会最先被匹配并接受。
        ```iptables
        # Example rule for whitelisted IP 10.10.0.10 on port 6443
        -I INPUT -s 10.10.0.10 -p tcp --dport 6443 -j ACCEPT
        ```
    *   **创建默认拒绝规则**: 在所有白名单规则添加完毕后，为该端口 **追加（-A）** 一条 `REJECT` 或 `DROP` 规则到 `INPUT` 链的定义中。
        *   对于 TCP 端口，使用 `REJECT --reject-with tcp-reset`。
        *   对于 UDP 端口，使用 `DROP`。

        ```iptables
        # Example default-deny rule for port 6443
        -A INPUT -p tcp --dport 6443 -j REJECT --reject-with tcp-reset
        ```

> [!TIP]
> **为何 TCP 用 `REJECT` 而 UDP 用 `DROP`?**
> *   **TCP `REJECT`**: 当流量被拒绝时，防火墙会返回一个 `tcp-reset` 包。这能让客户端立即知道连接被拒绝，从而快速失败，避免了不必要的等待和重试。这是一种更“友好”的拒绝方式。
> *   **UDP `DROP`**: UDP 是无连接的协议。`DROP` 会静默丢弃数据包，不给任何响应。这可以防止端口扫描工具轻易地探测到端口的存在，从而提高安全性。

最终，对于每一个受管端口，`/etc/sysconfig/iptables` 文件中 `INPUT` 链的规则顺序会是这样：
```
-I INPUT -s <白名单IP_1> -p <协议> --dport <端口> -j ACCEPT
-I INPUT -s <白名单IP_2> -p <协议> --dport <端口> -j ACCEPT
...
-A INPUT -p <协议> --dport <端口> -j REJECT/DROP  # 拒绝所有其他 IP
```
由于 `iptables` 规则是按顺序匹配的，这样的设计确保了只有白名单流量被放行，其他所有流量都会被最后的规则拦截。

## 脚本用法

可以直接调用 `config_port_filter.sh` 来动态管理白名单。

*   **添加白名单**:
    ```bash
    bash config_port_filter.sh add <IP_1> <IP_2> ... <CIDR_n>
    ```
*   **删除白名单**:
    ```bash
    bash config_port_filter.sh delete <IP_1> <IP_2> ... <CIDR_n>
    ```

> [!WARNING]
> **`delete` 功能存在缺陷!**
> 脚本中的 `delete_input_action` 函数存在严重问题。它使用一个宽泛的 `sed` 命令来删除规则，这可能误删其他规则。更重要的是，它 **只删除了 `ACCEPT` 规则，而没有删除末尾的 `REJECT/DROP` 规则**。
>
> 这会导致一个危险的后果：当一个 IP 从白名单中删除后，由于 `REJECT/DROP` 规则依然存在，该端口会对 **所有 IP**（包括之前在白名单中的 IP）都关闭。
>
> 在修复此问题前，请谨慎使用 `delete` 功能。推荐手动编辑 `/etc/sysconfig/iptables` 文件来删除规则，或者重新执行 `add` 命令（不包含要删除的 IP）来覆盖整个配置。

