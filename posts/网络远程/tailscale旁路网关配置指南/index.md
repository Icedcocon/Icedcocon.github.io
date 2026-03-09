# Tailscale 旁路网关配置指南 - 实现跨局域网设备互访


## 概述

本文详细介绍如何通过 Tailscale 旁路网关实现不同局域网之间的设备互访。这种配置方式特别适合以下场景：

- 需要让局域网内的虚拟机或容器访问 Tailscale 网络中的其他设备
- 希望在不安装 Tailscale 客户端的设备上访问 Tailscale 网络
- 需要将整个子网接入 Tailscale 网络

## 网络拓扑说明

在本指南中，我们使用以下网络拓扑：

- **主网关**: 192.168.2.1（局域网默认网关）
- **Tailscale 宿主机**: 192.168.2.2（运行 Tailscale 服务，Tailscale IP: 100.64.0.3）
- **Debian 虚拟机**: 192.168.2.10（需要访问 Tailscale 网络的客户端）
- **远程 Tailscale 设备**: 100.64.0.4（位于另一个局域网的 Tailscale 节点）

## 配置步骤

### 步骤 1：在 Tailscale 宿主机上启用子网路由

首先，需要在运行 Tailscale 的宿主机（192.168.2.2）上配置子网路由通告，让它成为局域网的旁路网关。

```bash
# 在 192.168.2.2 上执行
tailscale up \
  --accept-routes \
  --accept-dns=false \
  --advertise-exit-node \
  --advertise-routes=192.168.2.0/24 \
  --hostname=NanoPC-T6
```


#### 参数说明

- `--accept-routes`: 接受其他 Tailscale 节点通告的路由
- `--accept-dns=false`: 不接受 Tailscale 的 DNS 配置（如果域名解析失败，可改为 true）
- `--advertise-exit-node`: 将此节点设置为出口节点
- `--advertise-routes=192.168.2.0/24`: 向 Tailscale 网络通告本地子网路由
- `--hostname=NanoPC-T6`: 设置在 Tailscale 网络中的主机名

> [!NOTE]
> `--advertise-routes` 参数告诉 Tailscale 网络："我可以帮助其他节点访问 192.168.2.0/24 这个子网"。这是实现旁路网关的核心配置。

#### 启用 IP 转发

确保 Tailscale 宿主机启用了 IP 转发功能：

```bash
# 检查 IP 转发状态
cat /proc/sys/net/ipv4/ip_forward

# 如果返回 0，需要启用
sudo sysctl -w net.ipv4.ip_forward=1

# 永久启用
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

> [!WARNING]
> IP 转发是路由功能的基础。如果不启用，数据包将无法在不同网络接口之间转发，旁路网关功能将无法工作。

### 步骤 2：在 Headscale 服务器上批准路由


在 Tailscale 宿主机通告路由后，需要在 Headscale 控制服务器上批准这些路由：

```bash
# 查看节点的路由信息（假设节点 ID 为 2）
docker exec -it headscale headscale routes list -i 2

# 启用 192.168.2.0/24 子网路由
docker exec -it headscale headscale routes enable -i 2 -r 192.168.2.0/24
```

> [!TIP]
> 使用 `headscale nodes list` 命令可以查看所有节点及其 ID。路由批准是一个安全机制，防止未授权的子网被意外暴露到 Tailscale 网络中。

### 步骤 3：在主网关上配置静态路由

这是关键步骤。需要告诉主网关：所有发往 Tailscale 网络（100.64.0.0/24）的流量都应该转发到 Tailscale 宿主机。

在主网关（192.168.2.1）的管理界面中添加静态路由：

- **目标网络**: 100.64.0.0/24
- **子网掩码**: 255.255.255.0
- **下一跳网关**: 192.168.2.2

> [!NOTE]
> 不同品牌的路由器配置界面可能不同，但基本原理相同。有些路由器可能将"下一跳网关"称为"网关"或"Gateway"。

#### 原理说明


当局域网内的设备（如 192.168.2.10）尝试访问 Tailscale 网络中的设备（如 100.64.0.4）时：

1. 数据包发送到默认网关 192.168.2.1
2. 主网关查找路由表，发现 100.64.0.0/24 应该转发到 192.168.2.2
3. 数据包被转发到 Tailscale 宿主机 192.168.2.2
4. Tailscale 宿主机通过 Tailscale 隧道将数据包发送到目标设备 100.64.0.4

### 步骤 4：优化 - 在客户端添加静态路由（可选但推荐）

虽然通过主网关的静态路由已经可以工作，但会产生 ICMP 重定向警告，因为数据包经过了不必要的跳转。

#### 问题现象

```bash
root@debain:~# ping 100.64.0.4
PING 100.64.0.4 (100.64.0.4) 56(84) bytes of data.
64 bytes from 100.64.0.4: icmp_seq=1 ttl=63 time=82.3 ms
From 192.168.2.10: icmp_seq=2 Redirect Host(New nexthop: 192.168.2.1)
From 192.168.2.1: icmp_seq=2 Redirect Host(New nexthop: 192.168.2.2)
64 bytes from 100.64.0.4: icmp_seq=2 ttl=63 time=38.9 ms
```

#### 原因分析


虽然连接可以正常工作，但数据包路径不是最优的：

1. Debian 虚拟机 (192.168.2.10) → 主网关 (192.168.2.1)
2. 主网关发现目标在同一子网的 192.168.2.2，发送 ICMP 重定向消息
3. 数据包最终到达 192.168.2.2 → Tailscale → 100.64.0.4

这导致了额外的网络跳转和 ICMP 重定向警告。

#### 优化方案：添加客户端静态路由

在需要访问 Tailscale 网络的客户端（192.168.2.10）上直接添加路由：

```bash
# 在 Debian 虚拟机上执行
sudo ip route add 100.64.0.0/24 via 192.168.2.2

# 验证路由表
ip route show
```

这样数据包会直接从 192.168.2.10 发送到 192.168.2.2，避免经过主网关。

#### 永久配置

**使用 netplan（Ubuntu/Debian）**:

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

添加路由配置：

```yaml
network:
  version: 2
  ethernets:
    eth0:  # 替换为实际网卡名
      dhcp4: true
      routes:
        - to: 100.64.0.0/24
          via: 192.168.2.2
```


应用配置：

```bash
sudo netplan apply
```

**使用传统 /etc/network/interfaces**:

```bash
sudo nano /etc/network/interfaces
```

添加：

```
auto eth0
iface eth0 inet static
    address 192.168.2.10
    netmask 255.255.255.0
    gateway 192.168.2.1
    up ip route add 100.64.0.0/24 via 192.168.2.2
```

> [!TIP]
> 添加客户端静态路由后，数据包路径变为：192.168.2.10 → 192.168.2.2 → Tailscale 网络，减少了一次跳转，提高了网络效率。

### 步骤 5：验证配置

完成所有配置后，进行验证：

```bash
# 在 Debian 虚拟机上测试连接
ping -c 4 100.64.0.4

# 查看路由表
ip route show

# 追踪路由路径
traceroute 100.64.0.4
```

正常情况下，应该看到：

```bash
root@debain:~# ping 100.64.0.4
PING 100.64.0.4 (100.64.0.4) 56(84) bytes of data.
64 bytes from 100.64.0.4: icmp_seq=1 ttl=63 time=38.5 ms
64 bytes from 100.64.0.4: icmp_seq=2 ttl=63 time=38.3 ms
# 没有 Redirect 警告
```


## 常见问题与解决方案

### 问题 1：路由环路和 ICMP 重定向

#### 症状

```bash
From 192.168.2.10: icmp_seq=2 Redirect Host(New nexthop: 192.168.2.1)
From 192.168.2.1: icmp_seq=2 Redirect Host(New nexthop: 192.168.2.10)
From 192.168.2.1 icmp_seq=4 Time to live exceeded
```

#### 原因

- 主网关和 Tailscale 宿主机之间形成路由环路
- 通常是因为静态路由配置错误或 Tailscale 宿主机的默认网关指向了主网关

#### 解决方案

1. 确认主网关的静态路由正确指向 Tailscale 宿主机
2. 在 Tailscale 宿主机上添加明确的路由规则：

```bash
sudo ip route add 100.64.0.0/24 dev tailscale0
```

3. 禁用 ICMP 重定向（临时方案）：

```bash
sudo sysctl -w net.ipv4.conf.all.send_redirects=0
sudo sysctl -w net.ipv4.conf.all.accept_redirects=0
```

### 问题 2：无法 ping 通 Tailscale 网络

#### 检查清单

1. **验证 Tailscale 连接状态**：

```bash
tailscale status
```

2. **检查 IP 转发是否启用**：

```bash
cat /proc/sys/net/ipv4/ip_forward  # 应该返回 1
```


3. **确认路由已在 Headscale 上批准**：

```bash
docker exec -it headscale headscale routes list -i <节点ID>
```

4. **检查防火墙规则**：

```bash
# 查看 iptables 规则
sudo iptables -L -n -v

# 如果需要，允许转发
sudo iptables -A FORWARD -i tailscale0 -j ACCEPT
sudo iptables -A FORWARD -o tailscale0 -j ACCEPT
```

5. **验证路由表**：

```bash
# 在客户端上
ip route show | grep 100.64.0.0

# 在 Tailscale 宿主机上
ip route show
```

### 问题 3：DNS 解析失败

如果在使用 Tailscale 网络时遇到 DNS 解析问题：

```bash
# 重新配置 Tailscale，启用 DNS
tailscale up \
  --accept-routes \
  --accept-dns=true \
  --advertise-routes=192.168.2.0/24 \
  --hostname=NanoPC-T6
```

> [!NOTE]
> `--accept-dns=false` 会拒绝 Tailscale 的 DNS 配置。如果需要使用 Tailscale 的 MagicDNS 功能（通过主机名访问设备），应该设置为 `true`。

## 高级配置

### 配置多个子网路由


如果 Tailscale 宿主机连接到多个网络，可以通告多个子网：

```bash
tailscale up \
  --accept-routes \
  --advertise-routes=192.168.2.0/24,10.0.0.0/24 \
  --hostname=multi-subnet-gateway
```

然后在 Headscale 上分别批准每个路由：

```bash
docker exec -it headscale headscale routes enable -i 2 -r 192.168.2.0/24
docker exec -it headscale headscale routes enable -i 2 -r 10.0.0.0/24
```

### 使用 ACL 控制访问

为了增强安全性，可以在 Headscale 中配置 ACL 规则，限制哪些设备可以访问子网：

编辑 Headscale 配置文件中的 ACL 部分：

```yaml
acls:
  - action: accept
    src:
      - "group:admins"
    dst:
      - "192.168.2.0/24:*"
  
  - action: accept
    src:
      - "tag:trusted"
    dst:
      - "192.168.2.0/24:22,80,443"
```

> [!WARNING]
> ACL 规则从上到下匹配，第一个匹配的规则将被应用。确保将更具体的规则放在前面，通用规则放在后面。

### 配置 NAT 和源地址转换

在某些网络环境中，可能需要启用 SNAT（源地址转换）：

```bash
tailscale up \
  --accept-routes \
  --advertise-routes=192.168.2.0/24 \
  --snat-subnet-routes=true \
  --hostname=NanoPC-T6
```


> [!TIP]
> `--snat-subnet-routes=true` 使 Tailscale 对转发的流量执行源地址转换。这在目标网络不知道如何路由回 Tailscale 网络时特别有用。

## 网络原理详解

### 数据包流向分析

让我们详细分析一个数据包从 Debian 虚拟机（192.168.2.10）到远程 Tailscale 设备（100.64.0.4）的完整流程：

#### 未优化配置（通过主网关）

```
192.168.2.10 (源)
    ↓ [查找路由表，使用默认网关]
192.168.2.1 (主网关)
    ↓ [查找静态路由: 100.64.0.0/24 → 192.168.2.2]
    ↓ [发送 ICMP 重定向: 建议直接联系 192.168.2.2]
192.168.2.2 (Tailscale 宿主机)
    ↓ [通过 tailscale0 接口]
Tailscale 网络 (加密隧道)
    ↓ [可能通过 DERP 中继或直连]
100.64.0.4 (目标)
```

#### 优化配置（客户端静态路由）

```
192.168.2.10 (源)
    ↓ [查找路由表: 100.64.0.0/24 via 192.168.2.2]
192.168.2.2 (Tailscale 宿主机)
    ↓ [通过 tailscale0 接口]
Tailscale 网络 (加密隧道)
    ↓
100.64.0.4 (目标)
```

### 为什么需要主网关静态路由？

即使在客户端添加了静态路由，主网关的静态路由仍然重要：

1. **其他设备的访问**: 局域网中其他没有配置静态路由的设备仍需要通过主网关访问 Tailscale 网络
2. **DHCP 客户端**: 动态获取 IP 的设备通常只知道默认网关
3. **网络一致性**: 保持网络配置的一致性和可预测性


### ICMP 重定向机制

ICMP 重定向是路由器的一种优化机制：

1. 当路由器收到一个数据包时，发现最佳路径是将数据包发回到同一个网络接口
2. 路由器会发送 ICMP 重定向消息给源主机，告诉它："你应该直接联系 X.X.X.X，而不是通过我"
3. 源主机收到重定向后，会临时更新其路由缓存

虽然这不会影响连接，但会产生额外的网络开销和警告消息。

## 最佳实践建议

### 1. 分层路由策略

- **主网关**: 配置静态路由作为兜底方案
- **关键客户端**: 添加客户端静态路由以优化性能
- **DHCP 选项**: 如果路由器支持，可以通过 DHCP 推送静态路由

### 2. 安全加固

```bash
# 在 Tailscale 宿主机上禁用不必要的服务
sudo ufw enable
sudo ufw allow from 192.168.2.0/24 to any port 22

# 限制 Tailscale 访问
tailscale up \
  --accept-routes \
  --advertise-routes=192.168.2.0/24 \
  --shields-up  # 阻止入站连接
```

### 3. 监控和日志

```bash
# 监控 Tailscale 连接状态
watch -n 5 tailscale status

# 查看路由表变化
watch -n 5 ip route show

# 监控网络流量
sudo tcpdump -i tailscale0 -n
```


### 4. 性能优化

```bash
# 调整 TCP 参数以提高性能
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
sudo sysctl -w net.core.rmem_max=16777216
sudo sysctl -w net.core.wmem_max=16777216

# 永久配置
cat << EOF | sudo tee -a /etc/sysctl.conf
net.ipv4.tcp_congestion_control=bbr
net.core.rmem_max=16777216
net.core.wmem_max=16777216
EOF
```

### 5. 备份和恢复

定期备份 Tailscale 配置：

```bash
# 导出 Tailscale 状态
tailscale status --json > tailscale-status-backup.json

# 备份路由配置
ip route show > route-backup.txt

# 备份 Headscale 数据
docker exec headscale headscale nodes list -o json > headscale-nodes-backup.json
```

## 故障排查工具

### 网络诊断命令

```bash
# 1. 检查网络连通性
ping -c 4 100.64.0.4

# 2. 追踪路由路径
traceroute 100.64.0.4
# 或使用 mtr 获得更详细的信息
mtr -r -c 10 100.64.0.4

# 3. 检查端口连通性
nc -zv 100.64.0.4 22

# 4. 查看 Tailscale 日志
sudo journalctl -u tailscaled -f

# 5. 测试 DNS 解析
nslookup hostname.tailnet-name.ts.net
```


### Tailscale 调试命令

```bash
# 查看详细状态
tailscale status --json | jq

# 查看网络映射
tailscale netcheck

# 查看 DERP 延迟
tailscale netcheck --verbose

# 重置连接
tailscale down
tailscale up --reset

# 查看 Tailscale 接口信息
ip addr show tailscale0
```

### 抓包分析

```bash
# 在 Tailscale 宿主机上抓包
sudo tcpdump -i tailscale0 -w tailscale-traffic.pcap

# 抓取特定 IP 的流量
sudo tcpdump -i any host 100.64.0.4 -w specific-host.pcap

# 实时查看流量
sudo tcpdump -i tailscale0 -n -v
```

## 总结

通过本指南，您已经学会了如何配置 Tailscale 旁路网关，实现跨局域网设备互访。关键要点包括：

1. **在 Tailscale 宿主机上通告子网路由** - 使用 `--advertise-routes` 参数
2. **在 Headscale 服务器上批准路由** - 安全控制机制
3. **在主网关配置静态路由** - 确保所有设备都能访问 Tailscale 网络
4. **（可选）在客户端添加静态路由** - 优化网络性能，避免 ICMP 重定向
5. **启用 IP 转发** - 路由功能的基础

这种配置方式的优势在于：

- 无需在每个设备上安装 Tailscale 客户端
- 集中管理和控制网络访问
- 适合虚拟机、容器和嵌入式设备
- 保持网络拓扑的简洁性

> [!TIP]
> 如果您管理多个局域网，可以在每个网络中部署一个 Tailscale 旁路网关，实现全网互联。配合 Headscale 的 ACL 规则，可以精确控制不同网络之间的访问权限。

