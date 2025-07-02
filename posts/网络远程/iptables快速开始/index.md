# iptables快速开始


## iptables 核心概念

iptables 是一个功能强大的 Linux 防火墙工具，它基于内核的 Netfilter 框架。想要真正掌握 iptables，理解其核心组件和工作原理是第一步。`iptables` 的世界主要由三个核心概念构成：**表（Tables）**、**链（Chains）** 和 **规则（Rules）**。

> [!NOTE]
> **iptables 与 Netfilter 的关系**
>
> - **Netfilter**：位于 Linux 内核空间，是真正的"防火墙"，它在网络协议栈中设置了多个钩子（hooks），负责执行数据包过滤、网络地址转换（NAT）和数据包修改等底层工作。
> - **iptables**：位于用户空间，是一个命令行工具。它允许系统管理员编写和管理防火墙规则集，然后将这些规则加载到内核的 Netfilter 中，由 Netfilter 来具体实施。
>
> 简单来说，Netfilter 是默默工作的"执行者"，而 iptables 则是我们下达指令的"指挥官"。

`iptables` 的结构层次清晰，可以概括为：

```
iptables -> Tables -> Chains -> Rules
```

- **规则 (Rules)**：是具体的过滤条件和动作。例如，"允许来自 IP `1.2.3.4` 的 `SSH` 连接"就是一条规则。
- **链 (Chains)**：是一系列按顺序排列的规则的集合。当一个数据包到达一个链时，`iptables` 会从上到下依次检查链中的规则，一旦找到匹配的规则，就会执行相应的动作（如 `ACCEPT`, `DROP`），并且通常不再继续检查链中的其他规则。
- **表 (Tables)**：是用于组织不同功能链的容器。每张表都专注于一种特定的数据包处理任务，例如过滤、地址转换等。

>[!NOTE]
>简单地讲，tables由chains组成，而chains又由rules组成。iptables 默认有四个表Filter, NAT, Mangle, Raw。

接下来，我们将深入了解 `iptables` 中最重要的两个构建块：**链**和**表**。

### 五个核心链 (Five Core Chains)

Netfilter 在内核的网络协议栈中定义了五个关键的"挂载点"（hooks），任何网络数据包在内核中穿行时，都必然会经过其中的一个或多个挂载点。这五个挂载点，就对应着 `iptables` 中的五个默认链。

```text
       +------------+    +---------+    +--------+
IN  -> | PREROUTING | -> | ROUTING | -> | INPUT  |
       +------+-----+    +---+-----+    +---+----+
                             |              |
                             v              v
                         +---------+  +---------------+
                         | FORWARD |  | Local Process |
                         +--+------+  +---------------+
                             |              |
                             v              v
       +------------+    +---------+    +--------+
OUT <- |POSTROUTING | <- | ROUTING | <- | OUTPUT |
       +------+-----+    +---+-----+    +---+----+
```

- **`PREROUTING`**: 当数据包从任何网络接口（网卡）进入系统后，这是它遇到的第一个链。
	- 经过 raw, mangle, nat 表中规则的处理然后进行路由判断。
	- 若数据包的目的地址为本机则会进入INPUT链，若数据包的目的地址为其它地址则进入FORWARD链进行转发。
	- 此链在内核进行路由决策之前触发，是执行**目标网络地址转换 (DNAT)** 的理想位置，例如，将公网 IP 的请求转发到内网服务器。

- **`INPUT`**: 如果数据包经过路由判断后，其目标地址是本机，那么它将被发送到 `INPUT` 链。
	- 经过 mangle ,nat, filter 表中规则的处理然后发给 nginx、mysql等上层进程处理 .
	- 这是保护本机服务、**过滤所有进入本机应用程序流量**的关键关卡。

- **`FORWARD`**: 如果数据包的目标地址不是本机，而是需要经由本机转发到另一个网络接口，那么它将流经 `FORWARD` 链。
	- 经过 mangle , filter 表中规则的处理然后进入POSTROUTING链.
	- 当你的 Linux 系统**作为路由器或网关**时，此链是实施访问控制策略的核心。

- **`OUTPUT`**: 从本机应用程序或进程发出的数据包，在路由之前会经过 `OUTPUT` 链。
	- 经过 raw, mangle, nat, filter 表中规则的处理然后进入POSTROUTING链.
	- 你可以用它来**控制本机应用对外的网络访问**，例如，禁止某个程序访问互联网。

- **`POSTROUTING`**: 所有即将离开本机的数据包（无论是从本机发出的，还是被转发的），在发送到网络接口之前都会经过此链。
	- 可在 raw, mangle, nat 表中配置规则.
	- 这是执行**源网络地址转换 (SNAT)** 的最后机会，最常见的用途是让内网中的多台设备共享一个公网 IP 上网（也称为 `MASQUERADE`）。

根据数据包的流向，可以总结为三种主要场景：

-   **入站数据流 (Traffic to the firewall itself)**: `NIC -> PREROUTING -> INPUT -> Local Process`
-   **转发数据流 (Forwarded traffic)**: `NIC -> PREROUTING -> FORWARD -> POSTROUTING -> NIC`
-   **出站数据流 (Traffic from the firewall itself)**: `Local Process -> OUTPUT -> POSTROUTING -> NIC`

> [!TIP]
> **默认策略**
>
> 每个链都有一个默认策略（Policy），通常是 `ACCEPT`。这意味着如果一个数据包没有匹配链中的任何一条规则，它将被默认策略处理。在构建安全的防火墙时，一个最佳实践是**将 `INPUT` 和 `FORWARD` 链的默认策略设置为 `DROP`**，然后明确地 `ACCEPT` 你需要允许的流量。


### 四张功能表 (Four Function Tables)

`iptables` 将功能相近的规则组织在不同的"表"中。一个数据包在流经各个链时，会依次被链上所关联的表中的规则进行处理。默认情况下，`iptables` 有四张主要的表，它们的处理优先级由高到低排列如下：

-   **`raw`** (优先级最高): 主要用于**处理连接跟踪**（Connection Tracking）**的例外情况**。通过在 `raw` 表中对数据包打上 `NOTRACK` 标记，可以让 `iptables` 不再追踪此连接的状态，这在处理大量数据包（如负载均衡）时可以提升性能。它作用于 `PREROUTING` 和 `OUTPUT` 链。

-   **`mangle`**: 这是一张"魔法"表，专门用于**修改数据包的 IP 头部信息**。你可以用它来改变服务质量（TOS）、生存时间（TTL），或者给数据包打上一个内核标记（MARK），以便后续其他工具（如 `tc` 流量控制）或 `iptables` 自身其他规则可以根据这个标记进行匹配和处理。它几乎可以作用于所有链。

-   **`nat`** (Network Address Translation): 顾名思义，这张表专门用于**网络地址转换**。
    -   在 `PREROUTING` 链上进行 `DNAT` (目标地址转换)，常用于端口转发。
    -   在 `POSTROUTING` 链上进行 `SNAT` (源地址转换)，常用于共享上网。
    -   `OUTPUT` 链也支持 `DNAT`，用于改变本机发出数据包的目标地址。

-   **`filter`** (优先级最低，但最常用): 这是 `iptables` 的**默认表**，也是**防火墙功能**的核心所在。它负责对数据包进行**过滤**，即做出 `ACCEPT` (接受)、`DROP` (直接丢弃)、`REJECT` (拒绝并返回错误消息) 的裁决。它作用于 `INPUT`、`FORWARD` 和 `OUTPUT` 链。

下表清晰地展示了哪些链存在于哪些表中：

| Table    | `PREROUTING` | `INPUT` | `FORWARD` | `OUTPUT` | `POSTROUTING` |
| :------- | :----------: | :-----: | :-------: | :------: | :-----------: |
| `raw`    |      ✅      |         |           |    ✅    |               |
| `mangle` |      ✅      |    ✅   |     ✅    |    ✅    |       ✅      |
| `nat`    |      ✅      |    ✅   |           |    ✅    |       ✅      |
| `filter` |              |    ✅   |     ✅    |    ✅    |               |

> [!WARNING]
> **重要注意事项**
>
> 1.  操作 `iptables` 命令时，可以使用 `-t <table_name>` 参数来明确指定要操作的表。如果**省略 `-t` 参数，`iptables` 会默认操作 `filter` 表**。
> 2.  表的处理是有严格优先级的：`raw` > `mangle` > `nat` > `filter`。当一个数据包流经一个链（如 `PREROUTING`）时，它会先被该链上的 `raw` 表规则处理，然后是 `mangle` 表，最后是 `nat` 表。


## iptables 命令详解

理解了核心概念后，我们来看看如何使用 `iptables` 命令来管理防火墙规则。

### 命令基本语法

`iptables` 命令遵循一个通用的结构，熟悉这个结构是高效使用它的第一步。

```sh
iptables [-t table] COMMAND [chain] [matches] [-j target]
```

- **`-t <table>`**: 指定要操作的表，如 `filter`, `nat`, `mangle`, `raw`。如果省略此参数，`iptables` 将**默认操作 `filter` 表**。
- **`COMMAND`**: 指定要执行的操作，是命令的核心部分，例如 `-A` (追加规则)、`-D` (删除规则)、`-L` (列出规则)等。
- **`[chain]`**: 指定规则所作用的链，例如 `INPUT`, `OUTPUT`, `FORWARD`。
- **`[matches]`**: 定义规则的匹配条件。一条规则可以包含零个或多个匹配条件，例如 `-p tcp --dport 22` 用于匹配目标端口为 22 的 TCP 协议包。
- **`-j <target>`**: "jump" 的缩写，指定当数据包满足所有匹配条件后，应该执行的动作（Target），例如 `ACCEPT`、`DROP` 或跳转到另一个自定义链。

下面我们将详细介绍主要的命令、匹配参数和目标动作。

### 主要命令 (Commands)

这些命令定义了对规则和链本身的管理操作。

| 命令 (Command) | 长格式 | 描述 |
| :--- | :--- | :--- |
| `-A` | `--append` | 在指定链的末尾追加一条新规则。 |
| `-I [num]` | `--insert [num]` | 在指定链的开头（不写 `num`）或指定编号 `num` 处插入一条新规则。 |
| `-D [num]` | `--delete [num]` | 从指定链中删除一条规则，可以按规则内容或编号 `num` 精准删除。 |
| `-R [num]` | `--replace [num]` | 替换指定链中的某条规则，按编号 `num` 匹配。 |
| `-L` | `--list` | 列出指定链或所有链的规则。 |
| `-F` | `--flush` | 清空指定链或所有链的规则。 |
| `-P` | `--policy` | 为指定链设置默认策略 (`ACCEPT`, `DROP`)。 |
| `-N` | `--new-chain` | 创建一个新的用户自定义链。 |
| `-X` | `--delete-chain` | 删除一个用户自定义链（该链必须为空且未被任何规则引用）。 |
| `-Z` | `--zero` | 将指定链中所有规则的数据包和字节计数器清零。 |
| `-v` | `--verbose` | 与 `-L` 配合使用，显示更详细的信息（如计数器、网络接口等）。 |
| `-n` | `--numeric` | 与 `-L` 配合使用，以数字形式显示 IP 地址和端口号，不做 DNS 解析，速度更快。 |

### 通用及扩展匹配参数 (Matches)

匹配参数是规则的灵魂，它们定义了"什么样的"数据包会被我们的规则捕捉到。

#### 通用匹配

这些是最基础和常用的匹配参数。

| 参数 (Match) | 长格式 | 描述 |
| :--- | :--- | :--- |
| `-p <protocol>` | `--protocol` | 匹配指定的协议，如 `tcp`, `udp`, `icmp`, `all`。 |
| `-s <address>` | `--source` | 匹配来源 IP 地址或子网（如 `192.168.1.100`, `192.168.1.0/24`）。 |
| `-d <address>` | `--destination` | 匹配目标 IP 地址或子网。 |
| `-i <interface>` | `--in-interface` | 匹配数据包进入的网卡接口，如 `eth0`。（只适用于 `INPUT`, `FORWARD`, `PREROUTING` 链） |
| `-o <interface>` | `--out-interface`| 匹配数据包流出的网卡接口。（只适用于 `OUTPUT`, `FORWARD`, `POSTROUTING` 链） |

#### 扩展匹配

`iptables` 通过模块（`-m` 或 `--match`）提供了强大的扩展匹配能力，这里列举一些最常见的模块。

| 模块/参数 | 描述 | 示例 |
| :--- | :--- | :--- |
| **`tcp`** (需 `-p tcp`) | | |
| `--sport <port>` | 匹配 TCP 源端口。 | `--sport 8080` |
| `--dport <port>` | 匹配 TCP 目标端口。 | `--dport 443` |
| `--tcp-flags <mask> <comp>`| 匹配指定的 TCP 标志位。| `--tcp-flags SYN,ACK,FIN,RST SYN` |
| **`udp`** (需 `-p udp`) | | |
| `--sport <port>` | 匹配 UDP 源端口。 | `--sport 53` |
| `--dport <port>` | 匹配 UDP 目标端口。 | `--dport 53` |
| **`multiport`** | 匹配多个离散的端口（最多15个）。 | `-m multiport --dports 22,80,443` |
| **`state`** | 匹配连接的状态，这是实现状态防火墙的核心。 | `-m state --state NEW,ESTABLISHED` |
| `NEW` | 新建的连接请求。 | |
| `ESTABLISHED` | 已经建立并正在通信的连接。 | |
| `RELATED` | 与已有连接相关的连接（如 FTP 数据连接）。 | |
| `INVALID` | 无法识别或格式不正确的无效包。 | |
| **`limit`** | 限制匹配速率，常用于防止日志刷屏或简单的DoS攻击。 | `-m limit --limit 5/min` |
| **`connlimit`**| 限制来自单个 IP 的并发连接数。| `-m connlimit --connlimit-above 10` |
| **`icmp`** (需 `-p icmp`) | | |
| `--icmp-type <type>` | 匹配 ICMP 类型。 | `--icmp-type 8` (Echo Request, ping 请求) |

> [!NOTE]
> **规则的执行顺序**
>
> `iptables` 规则的执行是**严格按顺序**的。当一个数据包到达一个链时，它会从该链的第一条规则开始向下匹配：
>
> 1.  **找到匹配**：一旦找到匹配的规则，`iptables` 就会执行该规则指定的动作 (`-j target`)。
> 2.  **决定后续**：执行动作后，数据包的命运就由这个动作决定了。
>     -   像 `ACCEPT` 或 `DROP` 这样的**终结性动作**会立即停止在该链中的匹配，数据包的命运就此决定。
>     -   像 `LOG` 这样的**非终结性动作**执行后，数据包会继续匹配链中的下一条规则。
>     -   如果匹配了一个**自定义链**，数据包会进入该自定义链中继续匹配，完成后再返回主链。
> 3.  **无匹配**：如果数据包遍历完整个链都没有找到任何匹配的规则，它将被该链的**默认策略 (`Policy`)** 处理。

### 目标动作 (Targets)

`-j` 或 `--jump` 参数指定了当数据包匹配规则时，应该执行什么动作。

- **`ACCEPT`**: 允许数据包通过。这是一个终结性动作。

- **`DROP`**: 悄悄地丢弃数据包，不给来源地址任何回应。这是一个终结性动作。

> [!TIP]
> **`DROP` vs `REJECT`**
> `DROP` 更加安全，因为它不会暴露防火墙的存在。攻击者无法判断请求是被防火墙丢弃了还是目标主机不存在。而 `REJECT` 会返回一个错误消息（如 `icmp port-unreachable`），这会告诉对方这里有一台主机，只是端口不开放。在生产环境中，通常推荐使用 `DROP`。

- **`REJECT`**: 拒绝数据包，并返回一个错误消息。这是一个终结性动作。
    ```bash
    # 拒绝访问 22 端口，并返回一个 ICMP 错误
    iptables -A INPUT -p tcp --dport 22 -j REJECT --reject-with icmp-port-unreachable
    ```

- **`LOG`**: 记录数据包信息到内核日志（可通过 `dmesg` 或 `syslog` 查看），然后将数据包传递给链中的下一条规则。这是一个非终结性动作，常用于调试规则。
    ```bash
    # 记录所有尝试访问 22 端口的数据包，并添加 "SSH-Attemp:" 的前缀
    iptables -A INPUT -p tcp --dport 22 -j LOG --log-prefix "SSH-Attempt: "
    ```

- **`SNAT`**: **源地址转换**，用于 `nat` 表的 `POSTROUTING` 链。它会修改数据包的源 IP 地址，通常用于内网主机通过一个公网 IP 上网。
    ```bash
    # 将从 eth0 出去的数据包的源 IP 改为 1.2.3.4
    iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 1.2.3.4
    ```

- **`DNAT`**: **目标地址转换**，用于 `nat` 表的 `PREROUTING` 和 `OUTPUT` 链。它会修改数据包的目标 IP 地址，通常用于**端口转发**。
    ```bash
    # 将访问本机公网 IP 80 端口的流量转发到内网服务器 192.168.1.100 的 80 端口
    iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:80
    ```

- **`MASQUERADE`**: `SNAT` 的一个特例。它会自动获取出口网卡的 IP 地址作为源地址，特别适用于动态 IP 地址（如家庭宽带拨号上网）的场景。仅用于 `POSTROUTING` 链。
    ```bash
    # 动态伪装从 ppp0 接口（拨号上网）出去的流量
    iptables -t nat -A POSTROUTING -o ppp0 -j MASQUERADE
    ```
- **`REDIRECT`**: 在 `nat` 表中，将数据包重定向到本机自身的另一个端口，常用于**透明代理**。
    ```bash
    # 将本机的 80 端口流量重定向到 8080 端口
    iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080
    ```
- **`MARK`**: 在 `mangle` 表中使用，给数据包打上一个特殊的内核标记，供后续的 `iptables` 规则或其他网络工具（如 `tc` 流量控制）使用。

### 常见命令

以下是一些最常用的 `iptables` 命令，掌握这些命令可以帮助你快速管理防火墙规则：

#### 基础管理命令

```bash
# 查看当前所有规则（以数字形式显示IP和端口）
iptables –L
iptables -L -n -v

# 查看指定表的规则（例如 nat 表）
iptables -t nat -L -n -v

# 清空所有规则
iptables -F

# 清空指定表的规则
iptables -t nat -F

# 设置指定链的默认策略（丢弃、接受等）
iptables -P INPUT DROP 
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
```

#### 规则管理命令

```bash
# 添加规则到链末尾
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT 

# 在指定位置插入规则（例如在INPUT链的第2个位置）
iptables -I INPUT 2 -p tcp --dport 443 -j ACCEPT
iptables -I INPUT 2 -i eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT 

# 删除指定规则（通过规则内容）
iptables -D INPUT -p tcp --dport 80 -j ACCEPT

# 删除指定位置的规则（例如删除INPUT链的第2条规则）
iptables -D INPUT 2

# 替换指定位置的规则
iptables -R INPUT 2 -p tcp --dport 8443 -j ACCEPT
```

#### 规则持久化

```bash
# 保存当前规则到文件
iptables-save > /etc/iptables/rules.v4

# 从文件恢复规则
iptables-restore < /etc/iptables/rules.v4

# 在系统启动时自动加载规则
# 在 /etc/network/if-pre-up.d/ 目录下创建脚本
cat > /etc/network/if-pre-up.d/iptables << 'EOF'
#!/bin/sh
iptables-restore < /etc/iptables/rules.v4
EOF
chmod +x /etc/network/if-pre-up.d/iptables
```

### 如何正确配置iptables

配置 `iptables` 时，应该遵循以下最佳实践：

#### 基础配置步骤

1. **设置默认策略**
```bash
# 设置默认策略为 DROP
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP
```

2. **允许已建立的连接**
```bash
# 允许已建立的连接通过
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

3. **允许本地回环接口**
```bash
# 允许本地回环接口的所有流量
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
```

4. **配置常用服务**
```bash
# 允许新建 SSH 连接
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -j ACCEPT

# 允许新建 HTTP 和 HTTPS
iptables -A INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -m state --state NEW -j ACCEPT

# 允许 DNS 查询
iptables -A INPUT -p udp --dport 53 -j ACCEPT
iptables -A OUTPUT -p udp --dport 53 -j ACCEPT
```

5. **配置出站规则**
```bash
# 允许所有出站流量
iptables -A OUTPUT -j ACCEPT
```

#### 高级配置示例

```bash
# 允许特定 IP 访问
iptables -A INPUT -s 192.168.1.100 -j ACCEPT

# 允许特定网段访问
iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT

# 限制连接速率
iptables -A INPUT -p tcp --dport 80 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT

# 记录被拒绝的连接
iptables -A INPUT -j LOG --log-prefix "IPTables-Dropped: " --log-level 4
```

### 使用iptables抵抗常见攻击

#### 防御 SYN 洪水攻击

```bash
# 创建自定义链
iptables -N syn-flood

# 限制 SYN 请求速率
iptables -A syn-flood -m limit --limit 1/s --limit-burst 4 -j RETURN
iptables -A syn-flood -j DROP

# 将 SYN 请求转发到自定义链
iptables -A INPUT -p tcp --syn -j syn-flood

# 限制单个 IP 的并发连接数
iptables -A INPUT -p tcp --syn -m connlimit --connlimit-above 15 -j DROP
```

#### 防御 SSH 暴力破解

```bash
# 使用 recent 模块记录 SSH 连接尝试
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --set --name SSH

# 限制 SSH 连接尝试次数
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 300 --hitcount 3 --name SSH -j DROP
```

#### 防御 DDoS 攻击

```bash
# 限制单个 IP 的并发连接数
iptables -A INPUT -p tcp --dport 80 -m connlimit --connlimit-above 30 -j DROP

# 限制 ICMP 请求速率
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
```

#### 防御端口扫描

```bash
# 记录端口扫描尝试
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j LOG --log-prefix "Port Scan: "

# 丢弃无效的 TCP 标志组合
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
```

### iptables 规则的备份与恢复

#### 手动备份与恢复

```bash
# 备份当前规则
iptables-save > /etc/iptables/rules.v4.backup

# 恢复规则
iptables-restore < /etc/iptables/rules.v4.backup
```

#### 自动备份脚本

```bash
#!/bin/bash
# 创建备份目录
BACKUP_DIR="/var/backups/iptables"
mkdir -p $BACKUP_DIR

# 生成备份文件名（包含时间戳）
BACKUP_FILE="$BACKUP_DIR/iptables-$(date +%Y%m%d-%H%M%S).rules"

# 备份规则
iptables-save > $BACKUP_FILE

# 保留最近 7 天的备份
find $BACKUP_DIR -name "iptables-*.rules" -mtime +7 -delete
```

#### 使用 systemd 服务自动备份

```bash
# 创建服务文件
cat > /etc/systemd/system/iptables-backup.service << 'EOF'
[Unit]
Description=Backup iptables rules
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/iptables-save > /etc/iptables/rules.v4
ExecStart=/bin/cp /etc/iptables/rules.v4 /var/backups/iptables/iptables-$(date +%Y%m%d-%H%M%S).rules

[Install]
WantedBy=multi-user.target
EOF

# 创建定时器
cat > /etc/systemd/system/iptables-backup.timer << 'EOF'
[Unit]
Description=Backup iptables rules daily

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
EOF

# 启用服务
systemctl enable iptables-backup.timer
systemctl start iptables-backup.timer
```

#### 规则迁移

```bash
# 导出规则
iptables-save > rules.v4

# 在新系统上恢复规则
iptables-restore < rules.v4

# 验证规则是否恢复成功
iptables -L -n -v
```


## iptables-save 命令详解

## iptables-restore 命令详解
