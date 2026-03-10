# Tailscale自建DERP中继服务器完整指南


## 概述

DERP (Designated Encrypted Relay for Packets) 是 Tailscale 的中继服务器，用于在无法建立直接点对点连接时转发加密流量。本指南将详细介绍如何部署自己的 DERP 服务器，以及如何配置 Tailscale 客户端使用自定义 DERP 服务器。

### 什么是 DERP？

DERP 服务器在 Tailscale 网络中扮演两个关键角色：

1. **协商和建立连接**：帮助设备之间协商并建立直接连接（或中继连接）
2. **流量中继**：当直接连接不可能时（由于严格的 NAT、防火墙等原因），作为最后的备用方案中继流量

> [!NOTE]
> 大多数 Tailscale 设备之间的连接只使用 DERP 服务器来建立初始连接，之后会切换到直接的点对点连接。只有在无法建立直接连接时，DERP 才会持续中继流量。

### 为什么要自建 DERP 服务器？

虽然 Tailscale 官方在全球部署了大量 DERP 服务器，但在某些情况下自建 DERP 服务器仍然有意义：

- **性能优化**：官方 DERP 服务器对长时间流量传输有 QoS 限制，自建服务器可以获得更好的持续性能
- **地理位置**：在官方服务器覆盖不佳的地区（如中国大陆）部署更近的中继节点
- **网络隔离**：在完全隔离的内网环境中部署私有 DERP 服务器
- **合规要求**：某些组织需要确保流量不经过第三方服务器

> [!WARNING]
> 自建 DERP 服务器是一项高级操作，需要持续维护。在大多数情况下，使用 Tailscale 官方的 DERP 服务器就足够了。如果主要问题是中继连接性能，建议优先考虑配置 [Peer Relays](https://tailscale.com/kb/1257/peer-relays)（对等中继）。

## 环境准备

### 服务器要求

- 具有公网 IP 地址的服务器（VPS 或云主机）
- 操作系统：Linux（推荐 Ubuntu 20.04+ 或 Debian 11+）
- 已安装 Docker 和 Docker Compose
- （推荐）一个域名，用于 SSL 证书配置

### 端口规划

DERP 服务器需要开放以下端口：

| 协议  | 端口    | 用途                                    |
| --- | ----- | ------------------------------------- |
| TCP | 443   | HTTPS 服务（DERP 主要服务端口）                |
| TCP | 80    | HTTP 服务（用于 Let's Encrypt 证书验证和健康检查） |
| UDP | 3478  | STUN 服务（用于 NAT 穿透和 IP 发现）             |

> [!TIP]
> 如果您的环境中 80 和 443 端口已被占用，可以使用非标准端口，但需要相应调整配置。

## 方案一：使用域名部署（推荐）

使用域名部署是最简单和推荐的方式，DERP 可以利用 Let's Encrypt 自动获取和续订 SSL 证书。

### 1. DNS 配置

为您的域名添加 A 记录，指向服务器的公网 IP：

```bash
# 示例：将 derp.yourdomain.com 指向您的服务器 IP
derp.yourdomain.com.  A  YOUR_SERVER_IP
```

### 2. 防火墙配置

确保服务器防火墙开放必要端口：

```bash
# UFW 防火墙示例
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 3478/udp

# iptables 示例
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 3478 -j ACCEPT
```

### 3. Docker Compose 部署

创建 `docker-compose.yaml` 文件：

```yaml
version: '3.8'

services:
  derper:
    image: fredliang/derper:latest
    container_name: derper
    restart: unless-stopped
    environment:
      - DERP_DOMAIN=derp.yourdomain.com
      - DERP_CERT_MODE=letsencrypt
      - DERP_ADDR=:443
      - DERP_HTTP_PORT=80
      - DERP_VERIFY_CLIENTS=false  # 设置为 true 以限制只有您的 tailnet 可以使用
    ports:
      - "80:80/tcp"
      - "443:443/tcp"
      - "3478:3478/udp"
    volumes:
      - ./derper-certs:/app/certs
    cap_add:
      - NET_BIND_SERVICE
```

> [!IMPORTANT]
> 将 `derp.yourdomain.com` 替换为您自己的域名。

### 4. 启用客户端验证（可选但推荐）

为了防止您的 DERP 服务器被滥用，强烈建议启用客户端验证：

```yaml
version: '3.8'

services:
  derper:
    image: fredliang/derper:latest
    container_name: derper
    restart: unless-stopped
    environment:
      - DERP_DOMAIN=derp.yourdomain.com
      - DERP_CERT_MODE=letsencrypt
      - DERP_VERIFY_CLIENTS=true  # 启用客户端验证
    ports:
      - "80:80/tcp"
      - "443:443/tcp"
      - "3478:3478/udp"
    volumes:
      - ./derper-certs:/app/certs
      - /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock  # 挂载 Tailscale socket
```

> [!NOTE]
> 启用 `DERP_VERIFY_CLIENTS=true` 后，DERP 服务器会通过挂载的 Tailscale socket 验证客户端是否属于您的 tailnet。这需要在 DERP 服务器上也安装并运行 Tailscale 客户端。

### 5. 启动服务

```bash
docker-compose up -d
```

### 6. 验证部署

访问 `https://derp.yourdomain.com`，如果看到 "This is a Tailscale DERP server." 消息，说明部署成功。

## 方案二：使用 IP 地址部署

如果没有域名，可以使用 IP 地址部署，但需要使用自签名证书。

### 1. 创建自签名证书

```bash
#!/bin/bash
# generate-cert.sh

SERVER_IP="YOUR_SERVER_IP"
CERT_DIR="./certs"

mkdir -p "$CERT_DIR"

# 创建 OpenSSL 配置文件
cat > "$CERT_DIR/openssl.cnf" <<EOF
[req]
default_bits = 2048
distinguished_name = req_distinguished_name
req_extensions = req_ext
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
countryName = XX
stateOrProvinceName = N/A
localityName = N/A
organizationName = Self-signed certificate
commonName = $SERVER_IP

[req_ext]
subjectAltName = @alt_names

[v3_req]
subjectAltName = @alt_names

[alt_names]
IP.1 = $SERVER_IP
EOF

# 生成自签名证书（有效期 5 年）
openssl req -x509 -nodes -days 1825 \
  -newkey rsa:2048 \
  -keyout "$CERT_DIR/$SERVER_IP.key" \
  -out "$CERT_DIR/$SERVER_IP.crt" \
  -config "$CERT_DIR/openssl.cnf"

echo "证书已生成在 $CERT_DIR 目录"
```

运行脚本：

```bash
chmod +x generate-cert.sh
./generate-cert.sh
```

### 2. Docker Compose 配置

```yaml
version: '3.8'

services:
  derper:
    image: fredliang/derper:latest
    container_name: derper
    restart: unless-stopped
    environment:
      - DERP_DOMAIN=YOUR_SERVER_IP
      - DERP_CERT_MODE=manual
      - DERP_CERT_DIR=/app/certs
      - DERP_ADDR=:443
      - DERP_HTTP_PORT=80
    ports:
      - "80:80/tcp"
      - "443:443/tcp"
      - "3478:3478/udp"
    volumes:
      - ./certs:/app/certs:ro
```

### 3. 启动服务

```bash
docker-compose up -d
```

## 配置 Tailscale 使用自定义 DERP 服务器

部署完 DERP 服务器后，需要在 Tailscale 管理面板中配置自定义 DERP 映射。

### 1. 编辑 ACL 策略文件

登录 [Tailscale 管理面板](https://login.tailscale.com/admin/acls)，在 ACL 策略文件中添加 `derpMap` 配置：

#### 使用域名的配置

```json
{
  "acls": [
    // 您现有的 ACL 规则...
  ],
  
  "derpMap": {
    "OmitDefaultRegions": false,  // false: 保留官方 DERP 服务器; true: 只使用自定义 DERP
    "Regions": {
      "900": {  // 自定义区域 ID，使用 900-999 范围
        "RegionID": 900,
        "RegionCode": "custom-derp",
        "RegionName": "Custom DERP Server",
        "Nodes": [
          {
            "Name": "1",
            "RegionID": 900,
            "HostName": "derp.yourdomain.com",
            "IPv4": "YOUR_SERVER_IP",
            "IPv6": "YOUR_SERVER_IPV6",  // 可选
            "STUNPort": 3478,
            "DERPPort": 443,
            "InsecureForTests": false  // 使用有效证书时设为 false
          }
        ]
      }
    }
  }
}
```

#### 使用 IP 地址的配置

```json
{
  "derpMap": {
    "OmitDefaultRegions": false,
    "Regions": {
      "900": {
        "RegionID": 900,
        "RegionCode": "custom-derp-ip",
        "RegionName": "Custom DERP Server (IP)",
        "Nodes": [
          {
            "Name": "1",
            "RegionID": 900,
            "HostName": "YOUR_SERVER_IP",
            "IPv4": "YOUR_SERVER_IP",
            "STUNPort": 3478,
            "DERPPort": 443,
            "InsecureForTests": true  // 使用自签名证书时必须设为 true
          }
        ]
      }
    }
  }
}
```

### 2. DERP Map 配置参数说明

| 参数                   | 类型      | 说明                                                |
| -------------------- | ------- | ------------------------------------------------- |
| `OmitDefaultRegions` | Boolean | `false`: 保留官方 DERP 服务器<br>`true`: 只使用自定义 DERP 服务器 |
| `RegionID`           | Integer | 区域 ID，自定义区域使用 900-999 范围                          |
| `RegionCode`         | String  | 区域代码，用于标识区域                                       |
| `RegionName`         | String  | 区域名称，显示在客户端中                                      |
| `Name`               | String  | 节点名称                                              |
| `HostName`           | String  | DERP 服务器的域名或 IP 地址                                |
| `IPv4`               | String  | 服务器的 IPv4 地址                                      |
| `IPv6`               | String  | （可选）服务器的 IPv6 地址                                  |
| `STUNPort`           | Integer | STUN 服务端口，通常为 3478                                |
| `DERPPort`           | Integer | DERP 服务端口，通常为 443                                 |
| `InsecureForTests`   | Boolean | 使用自签名证书时设为 `true`，有效证书时设为 `false`                 |

### 3. 禁用特定官方 DERP 区域

如果您想禁用某个官方 DERP 区域，可以将其 `RegionID` 设置为 `null`：

```json
{
  "derpMap": {
    "Regions": {
      "1": null,  // 禁用纽约区域
      "2": null   // 禁用旧金山区域
    }
  }
}
```

### 4. 查看官方 DERP 映射

您可以通过以下方式查看 Tailscale 官方的 DERP 映射：

```bash
# 使用 curl 获取
curl https://controlplane.tailscale.com/derpmap/default

# 使用 jq 格式化输出
curl -s https://controlplane.tailscale.com/derpmap/default | jq '.Regions | to_entries | .[] | {id: .key, code: .value.RegionCode, name: .value.RegionName}'
```

### 5. 验证自定义 DERP 配置

在 Tailscale 客户端上运行以下命令验证配置：

```bash
# 查看网络状态和 DERP 连接
tailscale netcheck

# 查看当前使用的 DERP 服务器
tailscale status
```

输出中应该能看到您的自定义 DERP 服务器及其延迟信息。

## Tailscale Up 参数完整说明

在配置 Tailscale 客户端时，`tailscale up` 命令提供了丰富的参数选项。以下是所有参数的详细说明和应用场景。

### 基础认证参数

#### --auth-key

- **作用**：提供认证密钥，自动将节点添加到网络，无需浏览器交互认证
- **格式**：`--auth-key=tskey-auth-xxxxx` 或 `--auth-key=file:/path/to/keyfile`
- **应用场景**：
  - 自动化部署（CI/CD、容器编排）
  - 无头设备（服务器、路由器、IoT 设备）
  - 批量设备配置
  - 脚本化安装
- **示例**：
  ```bash
  tailscale up --auth-key=tskey-auth-xxxxx
  # 或从文件读取
  tailscale up --auth-key=file:/etc/tailscale/authkey
  ```

#### --force-reauth

- **作用**：强制重新认证，会断开当前 Tailscale 连接
- **应用场景**：
  - 切换账户
  - 解决认证问题
  - 更新设备所有者
- **警告**：不要在 SSH/RDP 远程连接时使用，会导致连接断开
- **示例**：
  ```bash
  tailscale up --force-reauth
  ```

#### --qr / --qr-format

- **作用**：生成登录 URL 的二维码
- **格式**：`--qr` 或 `--qr-format=small|large`
- **应用场景**：
  - 移动设备快速登录
  - 无法复制粘贴 URL 的环境
  - 演示和培训场景
- **示例**：
  ```bash
  tailscale up --qr
  tailscale up --qr --qr-format=large
  ```

### DNS 配置

#### --accept-dns

- **作用**：接受管理面板的 DNS 配置（默认 true）
- **应用场景**：
  - 使用 MagicDNS 功能（通过机器名访问设备）
  - 统一管理 DNS 解析
  - 设为 false：保留本地 DNS 设置，适合有特殊 DNS 需求的环境
- **示例**：
  ```bash
  tailscale up --accept-dns=true   # 使用 Tailscale DNS
  tailscale up --accept-dns=false  # 保留本地 DNS
  ```

#### --hostname

- **作用**：自定义设备主机名，覆盖操作系统提供的名称
- **应用场景**：
  - 统一命名规范（如 prod-web-01）
  - 避免主机名冲突
  - 更友好的 MagicDNS 名称
- **注意**：会改变 MagicDNS 中使用的机器名
- **示例**：
  ```bash
  tailscale up --hostname=prod-web-01
  ```

### 路由和网络访问

#### --accept-routes

- **作用**：接受其他节点广播的子网路由
- **默认值**：
  - Windows/iOS/Android/macOS：默认 true
  - Linux/BSD：默认 false
- **应用场景**：
  - 访问远程局域网资源（如办公室内网）
  - 通过子网路由器访问私有网络
  - 多站点网络互联
- **示例**：
  ```bash
  tailscale up --accept-routes
  ```

#### --advertise-routes

- **作用**：将本机可访问的子网路由广播给整个网络（子网路由器功能）
- **格式**：逗号分隔的 CIDR 列表
- **应用场景**：
  - 让其他设备访问本地局域网
  - 连接多个物理网络
  - 作为网关设备
- **注意**：需要在管理面板批准，并启用 IP 转发
- **示例**：
  ```bash
  # 广播单个子网
  tailscale up --advertise-routes=192.168.1.0/24
  
  # 广播多个子网
  tailscale up --advertise-routes=10.0.0.0/8,192.168.0.0/24
  
  # 清除路由广播
  tailscale up --advertise-routes=
  ```

### 出口节点（Exit Node）

#### --advertise-exit-node

- **作用**：将本机作为出口节点，为其他设备提供互联网流量代理
- **应用场景**：
  - 提供 VPN 出口（类似传统 VPN 服务器）
  - 让移动设备通过家庭网络上网
  - 绕过地理限制
  - 在不受信任的 WiFi 上保护流量
- **注意**：需要在管理面板批准
- **示例**：
  ```bash
  tailscale up --advertise-exit-node
  ```

#### --exit-node

- **作用**：使用指定的出口节点路由所有互联网流量
- **格式**：IP 地址、机器名或 `auto:any`
- **应用场景**：
  - 在公共 WiFi 上保护隐私
  - 访问特定地区的服务
  - 统一出口 IP 地址
- **示例**：
  ```bash
  # 使用特定出口节点（通过 IP）
  tailscale up --exit-node=100.64.0.1
  
  # 使用特定出口节点（通过主机名）
  tailscale up --exit-node=exit-node-name
  
  # 自动选择最佳出口节点
  tailscale up --exit-node=auto:any
  
  # 禁用出口节点
  tailscale up --exit-node=
  ```

#### --exit-node-allow-lan-access

- **作用**：使用出口节点时仍可访问本地局域网
- **应用场景**：
  - 使用出口节点的同时访问本地打印机
  - 访问本地 NAS 或文件共享
  - 保持本地设备连接（智能家居等）
- **示例**：
  ```bash
  tailscale up --exit-node=exit-node-name --exit-node-allow-lan-access
  ```

### 访问控制和安全

#### --advertise-tags

- **作用**：为设备分配 ACL 标签，基于标签管理访问权限
- **格式**：逗号分隔的标签列表，每个标签必须以 `tag:` 开头
- **应用场景**：
  - 按角色分组设备（如 `tag:server`, `tag:dev`）
  - 自动化访问控制
  - 环境隔离（生产/测试）
  - 批量权限管理
- **注意**：必须在 ACL 的 TagOwners 中被授权
- **示例**：
  ```bash
  tailscale up --advertise-tags=tag:server,tag:prod,tag:web
  ```

#### --shields-up

- **作用**：阻止来自 Tailscale 网络的入站连接
- **应用场景**：
  - 个人设备只需主动连接其他设备
  - 笔记本电脑等移动设备
  - 提高安全性，减少攻击面
- **类似**：防火墙的"隐身模式"
- **示例**：
  ```bash
  tailscale up --shields-up
  ```

#### --ssh

- **作用**：启动 Tailscale SSH 服务器，根据 ACL 策略允许访问
- **应用场景**：
  - 无需管理 SSH 密钥
  - 基于身份的 SSH 访问
  - 统一的访问审计
  - 零信任 SSH 访问
- **示例**：
  ```bash
  tailscale up --ssh
  ```

### 高级配置

#### --accept-risk

- **作用**：跳过风险确认提示
- **可选值**：
  - `lose-ssh`：接受可能失去 SSH 访问的风险
  - `mac-app-connector`：macOS 应用连接器风险
  - `linux-strict-rp-filter`：Linux 严格反向路径过滤风险
  - `all`：接受所有风险
- **应用场景**：自动化脚本、无人值守部署
- **示例**：
  ```bash
  tailscale up --accept-risk=lose-ssh
  ```

#### --advertise-connector

- **作用**：将节点广播为应用连接器
- **应用场景**：企业级应用访问控制
- **示例**：
  ```bash
  tailscale up --advertise-connector
  ```

#### --login-server

- **作用**：指定控制服务器 URL（默认 Tailscale 官方服务器）
- **应用场景**：
  - 使用 Headscale 自建控制服务器
  - 私有部署
  - 企业内网环境
- **示例**：
  ```bash
  tailscale up --login-server=https://headscale.example.com
  ```

#### --operator

- **作用**：允许指定的 Unix 用户无需 sudo 操作 tailscaled
- **应用场景**：
  - 非 root 用户管理 Tailscale
  - 服务账户自动化
  - 安全最佳实践
- **示例**：
  ```bash
  tailscale up --operator=myuser
  ```

#### --reset

- **作用**：将未指定的设置重置为默认值
- **应用场景**：
  - 清理之前的配置
  - 确保干净的配置状态
  - 避免遗留设置影响
- **示例**：
  ```bash
  tailscale up --reset --advertise-routes=192.168.1.0/24
  ```

#### --timeout

- **作用**：等待 tailscaled 进入运行状态的最大时间
- **默认值**：0s（永久等待）
- **应用场景**：
  - 脚本中避免无限等待
  - CI/CD 管道超时控制
- **示例**：
  ```bash
  tailscale up --timeout=30s
  ```

#### --json

- **作用**：以 JSON 格式输出（格式可能变化）
- **应用场景**：
  - 脚本解析输出
  - 自动化工具集成
  - 日志记录
- **示例**：
  ```bash
  tailscale up --json
  ```

### Linux 特定参数

#### --netfilter-mode

- **作用**：控制防火墙自动配置程度
- **可选值**：
  - `on`：完全管理（默认）
  - `nodivert`：创建规则但不自动调用
  - `off`：不管理防火墙
- **应用场景**：自定义防火墙规则、特殊网络环境
- **示例**：
  ```bash
  tailscale up --netfilter-mode=nodivert
  ```

#### --snat-subnet-routes

- **作用**：对广播的本地路由进行源 NAT
- **默认值**：true
- **应用场景**：子网路由器配置、NAT 穿透
- **示例**：
  ```bash
  tailscale up --advertise-routes=192.168.1.0/24 --snat-subnet-routes=true
  ```

#### --stateful-filtering

- **作用**：为子网路由器和出口节点启用状态防火墙
- **默认值**：false
- **应用场景**：增强安全性、防止未授权的入站连接
- **示例**：
  ```bash
  tailscale up --advertise-exit-node --stateful-filtering
  ```

### Windows 特定参数

#### --unattended

- **作用**：用户注销后保持 Tailscale 运行
- **应用场景**：
  - Windows 服务器
  - 无人值守系统
  - 后台服务
- **示例**：
  ```bash
  tailscale up --unattended
  ```

## 常见配置场景示例

### 场景 1：配置子网路由器

将设备配置为子网路由器，允许其他 Tailscale 节点访问本地网络：

```bash
# 在路由器设备上
tailscale up \
  --advertise-routes=192.168.1.0/24 \
  --snat-subnet-routes=true \
  --accept-dns=true \
  --hostname=home-router

# 在管理面板批准路由后，在客户端设备上
tailscale up --accept-routes
```

### 场景 2：配置出口节点

将设备配置为出口节点，为其他设备提供互联网访问：

```bash
# 在出口节点设备上
tailscale up \
  --advertise-exit-node \
  --hostname=exit-node-home

# 在客户端设备上使用出口节点
tailscale up \
  --exit-node=exit-node-home \
  --exit-node-allow-lan-access
```

### 场景 3：自动化部署（无头服务器）

使用认证密钥自动部署 Tailscale：

```bash
tailscale up \
  --auth-key=tskey-auth-xxxxx \
  --advertise-tags=tag:server,tag:prod \
  --hostname=prod-web-01 \
  --accept-dns=false \
  --shields-up
```

### 场景 4：同时配置子网路由和出口节点

将设备同时配置为子网路由器和出口节点：

```bash
tailscale up \
  --advertise-routes=192.168.1.0/24 \
  --advertise-exit-node \
  --snat-subnet-routes=true \
  --accept-dns=false \
  --hostname=gateway-node
```

### 场景 5：使用 Headscale 自建控制服务器

连接到自建的 Headscale 控制服务器：

```bash
tailscale up \
  --login-server=https://headscale.example.com \
  --auth-key=xxxxx \
  --accept-routes \
  --hostname=my-device
```

### 场景 6：个人移动设备（只出站）

配置笔记本电脑等移动设备，只允许主动连接其他设备：

```bash
tailscale up \
  --shields-up \
  --accept-routes \
  --accept-dns=true
```

## 重要提示和最佳实践

### 参数持久化

> [!WARNING]
> `tailscale up` 的参数不会持久化。每次运行 `tailscale up` 都需要指定完整的参数集。如果忘记指定之前使用的参数，这些设置会被重置为默认值。

从 Tailscale v1.8 开始，如果您忘记指定之前的参数，CLI 会发出警告并提供包含所有现有参数的命令。

### 清空参数

要清空之前设置的参数，传递空值即可：

```bash
# 清空路由广播
tailscale up --advertise-routes=

# 清空标签
tailscale up --advertise-tags=

# 禁用出口节点
tailscale up --exit-node=
```

### 使用配置文件（推荐）

对于需要持久化配置的场景，建议使用 Tailscale 配置文件而不是命令行参数。创建 `/etc/tailscale/tailscaled.json`：

```json
{
  "version": "alpha0",
  "serverURL": "https://controlplane.tailscale.com",
  "authKey": "file:/etc/tailscale/authkey",
  "hostname": "my-device",
  "acceptDNS": true,
  "acceptRoutes": true,
  "advertiseRoutes": ["192.168.1.0/24"],
  "exitNode": "",
  "shieldsUp": false,
  "runSSHServer": false
}
```

然后使用配置文件启动：

```bash
tailscaled --config=/etc/tailscale/tailscaled.json
```

### 安全建议

1. **启用客户端验证**：如果自建 DERP 服务器，务必启用 `DERP_VERIFY_CLIENTS=true`
2. **使用标签管理权限**：通过 ACL 标签而不是用户来管理设备权限
3. **限制子网路由**：只广播必要的子网，避免过度暴露网络
4. **定期更新**：保持 Tailscale 客户端和 DERP 服务器更新到最新版本
5. **监控日志**：定期检查 DERP 服务器和客户端日志，发现异常访问

### 性能优化

1. **优先使用直连**：DERP 只应作为备用方案，确保网络配置允许 UDP 打洞
2. **选择最近的 DERP**：自建 DERP 服务器应部署在地理位置最优的位置
3. **启用 IPv6**：如果可能，启用 IPv6 可以提高连接成功率
4. **考虑 Peer Relays**：对于频繁的中继连接，Peer Relays 通常比 DERP 性能更好

## 故障排查

### DERP 服务器无法访问

```bash
# 检查端口是否开放
nc -zv derp.yourdomain.com 443
nc -zu derp.yourdomain.com 3478

# 检查证书
openssl s_client -connect derp.yourdomain.com:443 -servername derp.yourdomain.com

# 查看 Docker 日志
docker logs derper
```

### 客户端无法连接到自定义 DERP

```bash
# 检查网络连接状态
tailscale netcheck

# 查看详细状态
tailscale status --json

# 强制刷新 DERP 映射
tailscale debug derp
```

### 路由不生效

```bash
# 检查 IP 转发是否启用（Linux）
sysctl net.ipv4.ip_forward
# 如果返回 0，启用它
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# 检查路由是否被批准（在管理面板或使用 CLI）
tailscale status
```

## 参考资源

- [Tailscale 官方文档](https://tailscale.com/kb/)
- [DERP 服务器文档](https://tailscale.com/kb/1232/derp-servers)
- [自定义 DERP 服务器](https://tailscale.com/kb/1118/custom-derp-servers)
- [Tailscale ACL 配置](https://tailscale.com/kb/1018/acls)
- [Headscale 项目](https://github.com/juanfont/headscale)
- [Tailscale GitHub 仓库](https://github.com/tailscale/tailscale)

## 总结

自建 DERP 服务器可以为特定场景提供更好的性能和控制，但需要权衡维护成本。本指南涵盖了从部署到配置的完整流程，以及 `tailscale up` 命令的所有参数说明。根据您的实际需求选择合适的部署方案，并遵循最佳实践以确保网络的安全性和稳定性。

