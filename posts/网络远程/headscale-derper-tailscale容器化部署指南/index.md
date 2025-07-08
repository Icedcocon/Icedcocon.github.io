# Headscale 与 Derper 容器化部署指南


## 概述

本文将详细介绍如何使用 Docker 容器化部署 Headscale（一个开源的 Tailscale 控制服务器实现）和 Derper（Tailscale 的中继服务器），以搭建一个完全自托管、安全可控的内网穿透网络。

-   **Headscale**: 作为控制平面，负责管理网络中的设备、用户认证、密钥交换和 ACL 规则。
-   **Derper**: 作为数据平面的一部分，当设备之间无法建立直接的点对点（P2P）连接时，提供加密流量中继服务。

## 环境准备

在开始部署之前，请确保您已准备好以下环境：

-   一台具有公网 IP 地址的服务器。
-   服务器已安装 Docker 和 Docker Compose。
-   熟悉基本的 Linux 命令行操作。
-   （可选）一个域名，用于简化 Derper 的证书配置。

### 端口规划

> [!NOTE]
> 本指南将使用以下端口。如果您的环境中这些端口已被占用，可以自行修改并更新所有相关配置文件。请确保服务器防火墙已放行这些端口。

| 服务 | 协议 | 端口 | 用途 |
| --- | --- | --- | --- |
| Headscale | TCP | `8080` | 用于 Tailscale 客户端与 Headscale API 的通信和管理。 |
| Derper | TCP | `30443` | Derper 的 HTTPS 服务端口，用于中继加密流量。 |
| Derper | TCP | `30080` | Derper 的 HTTP 服务端口，主要用于服务健康检查或重定向。 |
| Derper | UDP | `3478` | STUN 服务端口，用于帮助 NAT 后面的设备发现其公网 IP 和端口，以尝试建立 P2P 连接。 |

## 部署 Derper 中继服务器

Derper 服务器的部署分为两种情况：拥有自己的域名和纯 IP 使用。

### 方案一：使用域名部署 (推荐)

当您拥有域名时，Derper 可以利用内置的 Let's Encrypt 支持自动获取和续订 SSL 证书，这是最简单和推荐的方式。

> [!TIP]
> 使用域名可以大大简化 SSL 证书的管理，并提高服务的可信度。DERP 服务器需要公网直接访问，不能位于 NAT 设备或负载均衡器后面。

#### DNS 与防火墙配置

1.  **DNS 配置**: 为您的域名设置一条 A 或 AAAA 记录，指向您服务器的公网 IP。例如，将 `derp.yourdomain.com` 指向 `YOUR_SERVER_IP`。
2.  **防火墙配置**: 根据我们的端口规划，需要开放以下端口：
    - TCP `30080` - HTTP 服务 (用于健康检查和 Let's Encrypt 验证)
    - TCP `30443` - HTTPS 服务 (DERP 主要服务端口)
    - UDP `3478` - STUN 服务 (用于 NAT 穿透)

#### 使用 Docker Compose 部署

标准 DERP 服务通常使用端口 80(HTTP)、443(HTTPS) 和 3478(STUN)。但由于我们的环境中 80 和 443 端口已被 nginx-proxy-manager 占用，我们需要使用非标准端口并配置适当的参数。

创建 `docker-compose.yaml` 文件：

```yaml
version: '3.8'
services:
  derper:
    image: fredliang/derper:v1.84.2
    container_name: derper
    restart: always
    ports:
      - "30443:30443/tcp"   # 自定义 HTTPS 端口
      - "3478:3478/udp"     # STUN 服务
    volumes:
      - /root/nginx-proxy-manager/letsencrypt:/app/certs/letsencrypt:ro
    command:
      - "/app/derper"
      - "--hostname=derp.yourdomain.com"
      - "--certmode=manual"
      - "--certdir=/app/certs/letsencrypt/live/npm-6"
      - "-a=:30443"
      - "--http-port=30080"
      # - "--verify-clients" # 需要部署 tailnetd
```

> [!NOTE]
> 请将 `derp.yourdomain.com` 替换为您自己的域名。Tailscale 官方说明中提到 HTTP 端口不能自定义，但实际上 `--http-port` 参数允许您更改 HTTP 服务端口，这对于 Let's Encrypt 验证很重要。

> [!WARNING]
> Derper 需要使用域名对应的 `.key` 和 `.crt` 文件。您需要在证书目录中为您的域名创建符号链接，例如:
> ```bash
> cd /root/nginx-proxy-manager/letsencrypt/live/npm-6
> ln -s privkey.pem yourdomain.com.key
> ln -s fullchain.pem yourdomain.com.crt
> ```
> 请确保链接名称与您在 `--hostname` 参数中指定的域名完全匹配。

#### 启动服务

配置完成后，启动 DERP 服务：

```bash
docker-compose up -d
```

> [!WARNING]
> **安全加固**: 默认情况下，您的 Derper 服务器是公开的，任何知道您服务器地址的人都可以使用。为防止滥用，强烈建议将其加入您的 Tailscale 网络，并在 `command` 中添加 `--verify-clients` 参数。这会强制 Derper 验证所有连接客户端都属于您的 tailnet。

### 方案二：使用 IP 地址部署

在某些情况下，您可能没有可用的域名，或者希望直接使用 IP 地址部署 Derper 服务。这种方法需要修改 Tailscale 的源码并编译自定义的 Derper 镜像，以绕过域名验证要求。

> [!NOTE]
> IP 地址部署适合测试环境或临时使用。对于生产环境，推荐使用域名方案，以便更好地管理 SSL 证书和提高服务可靠性。

#### 准备源码

首先，我们需要克隆修改版的 Derper 仓库并初始化子模块：

```bash
# 仓库克隆
git clone https://mirror.ghproxy.com/https://github.com/yangchuansheng/ip_derper.git
cd ip_derper
git submodule update --init --recursive
vim tailscale/cmd/derper/cert.go
```

#### 修改源码

需要修改 Tailscale 的证书验证逻辑，以允许使用 IP 地址作为主机名：

```go
# 注释掉以下3行代码，以允许 IP 地址作为主机名
func (m *manualCertManager) getCertificate(hi *tls.ClientHelloInfo) (*tls.Certificate, error) {
    //if hi.ServerName != m.hostname {
    //    return nil, fmt.Errorf("cert mismatch with hostname: %q", hi.ServerName)
    //}
    ....
}
```

> [!TIP]
> 修改源码是为了绕过 Tailscale 默认的主机名验证，使系统能够接受自签名证书与 IP 地址的组合。这对内网环境部署特别有用。

#### 创建 Dockerfile

以下是自定义 Derper 服务器的 Dockerfile，它将从修改后的源码构建 Derper 程序：

```dockerfile
# 多阶段构建: 第一阶段编译 Derper
FROM golang:latest AS builder

LABEL org.opencontainers.image.source https://github.com/yangchuansheng/ip_derper

WORKDIR /app

ADD tailscale /app/tailscale

# 构建修改版 derper
RUN cd /app/tailscale/cmd/derper && \
    CGO_ENABLED=0 /usr/local/go/bin/go build -buildvcs=false -ldflags "-s -w" -o /app/derper && \
    cd /app && \
    rm -rf /app/tailscale

# 第二阶段: 创建运行环境
FROM ubuntu:20.04
WORKDIR /app

# 环境变量配置
ENV DERP_ADDR :30443
ENV DERP_HTTP_PORT 30080
ENV STUN_PORT 3478
ENV DERP_HOST=127.0.0.1
ENV DERP_CERTS=/app/certs/
ENV DERP_STUN true
ENV DERP_VERIFY_CLIENTS false

# 安装必要依赖
RUN apt-get update && \
    apt-get install -y openssl curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

COPY build_cert.sh /app/
COPY --from=builder /app/derper /app/derper

# 启动命令: 生成自签名证书并启动 derper 服务
CMD bash /app/build_cert.sh $DERP_HOST $DERP_CERTS /app/san.conf && \
    /app/derper -hostname=$DERP_HOST \
    -certmode=manual \
    -certdir=$DERP_CERTS \
    -stun=$DERP_STUN \
    -stun-port $STUN_PORT \
    -a=$DERP_ADDR \
    -http-port=$DERP_HTTP_PORT \
    -verify-clients=$DERP_VERIFY_CLIENTS
```

> [!WARNING]
> 默认配置中 `DERP_VERIFY_CLIENTS` 设置为 `false`，这意味着任何人都可以使用您的 Derper 服务。在生产环境中，建议设置为 `true` 并将此服务器专门用于您自己的 Tailscale 网络。

#### 证书生成脚本

Dockerfile 中引用了 `build_cert.sh` 脚本，该脚本负责为 Derper 服务器生成自签名 SSL 证书。以下是脚本内容及其功能说明：

```bash
#!/bin/bash

CERT_HOST=$1    # 证书的主机名/IP地址
CERT_DIR=$2     # 证书存储目录
CONF_FILE=$3    # OpenSSL配置文件路径

# 创建OpenSSL配置文件，包含SAN(Subject Alternative Name)设置
echo "[req]
default_bits  = 2048
distinguished_name = req_distinguished_name
req_extensions = req_ext
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
countryName = XX
stateOrProvinceName = N/A
localityName = N/A
organizationName = Self-signed certificate
commonName = $CERT_HOST: Self-signed certificate

[req_ext]
subjectAltName = @alt_names

[v3_req]
subjectAltName = @alt_names

[alt_names]
IP.1 = $CERT_HOST
" > "$CONF_FILE"

# 创建证书目录并生成自签名证书，有效期730天(2年)
mkdir -p "$CERT_DIR"
openssl req -x509 -nodes -days 730 -newkey rsa:2048 -keyout "$CERT_DIR/$CERT_HOST.key" -out "$CERT_DIR/$CERT_HOST.crt" -config "$CONF_FILE"
```

> [!TIP]
> 此脚本最关键的部分是在证书的 SAN 扩展中添加 IP 地址。这使得证书可以用于直接通过 IP 地址访问的 HTTPS 服务，避免了常见的证书错误。在 Derper 容器启动时，此脚本会被自动执行，为您的服务器 IP 生成自签名证书。

#### 构建和运行

最后，构建 Docker 镜像并启动 Derper 服务器：

```bash
# 编译镜像
docker build -t ip_derper:1.76.1 .

# 启动容器
docker run \
    --restart always \
    --net host \
    --name derper \
    -d ip_derper:1.76.1 # 或使用预构建镜像: ghcr.io/yangchuansheng/ip_derper
```

> [!TIP]
> 使用 `--net host` 参数可以让容器直接使用宿主机网络，简化网络配置，但可能存在安全隐患。对于生产环境，考虑使用端口映射而非 host 网络模式。

#### 验证部署

完成部署后，需要进行以下操作：

```bash
# 确保服务器防火墙放行以下端口
# - TCP 30443: HTTPS Derper 服务
# - TCP 30080: HTTP 服务端口 
# - UDP 3478: STUN 协议端口

# 验证服务是否正常运行
# 访问 https://<您的服务器IP>:30443 应显示 "This is a Tailscale DERP server."
```

成功部署后，您可以在 Headscale 配置中添加此 Derper 服务器，供您的 Tailscale 网络使用。

## Headscale 部署

Headscale 是 Tailscale 控制服务器的开源实现，负责管理整个网络的设备注册、认证和配置。本节将详细介绍如何使用 Docker 部署 Headscale 服务，并配置与前面部署的 DERP 中继服务器集成。

### 准备工作

在开始部署前，我们需要创建必要的目录结构并获取 Headscale 的源代码：

```bash
# 创建配置和数据目录
mkdir -p container-config
mkdir -p container-data/data
mkdir -p web

# 克隆特定版本的 Headscale 仓库
git clone -b v0.26.1 https://ghproxy.net/https://github.com/juanfont/headscale.git headscale-repo
# 复制示例配置文件
cp headscale-repo/config-example.yaml container-config/config.yaml
cd headscale-repo
```

### 配置文件准备

#### 初始化脚本

在 Headscale 仓库中创建 `init.sh` 脚本，用于容器启动时初始化服务：

```bash
#!/bin/sh

# 检查是否存在自定义 Caddyfile
if [ ! -f /data/Caddyfile ]; then
  echo "未检测到 Caddyfile，使用默认配置"
  cp /staging/Caddyfile /data/Caddyfile
fi

# 启动 Caddy 服务
echo "启动 Caddy 服务..."
/usr/sbin/caddy run --adapter caddyfile --config /data/Caddyfile 2>&1 | sed 's/^/[Caddy] /' &

# 增加等待时间，确保Caddy完全启动
echo "等待Caddy启动完成..."
sleep 5

# 启动 Headscale
echo "启动 Headscale 服务..."
/bin/headscale serve 2>&1 | sed 's/^/[Headscale] /' &

# 等待所有后台进程
wait
```

#### Caddy 配置

创建 `Caddyfile` 文件，用于配置 Headscale 的 Web 界面和 API 代理：

- 1. **使用 HTTP 部署时**

```
{
        skip_install_trust
        auto_https disable_redirects
        http_port {$HTTP_PORT}
        https_port {$HTTPS_PORT}
}

# HTTP 服务配置
:{$HTTP_PORT} {
        # Web UI路径处理
        handle_path /web/* {
                uri strip_prefix /web
                file_server {
                        root /web
                }
        }

        # 其他所有请求代理到headscale
        handle {
                reverse_proxy http://headscale:8080
        }
}
```

- 1. **使用 HTTPS 部署时**

```
{
        skip_install_trust
        auto_https disable_redirects
        http_port {$HTTP_PORT}
        https_port {$HTTPS_PORT}
}

# HTTP 服务配置
:{$HTTP_PORT} {
        # 专门处理 /web 路径
        handle_path /web/* {
                uri strip_prefix /web
                file_server {
                        root /web
                }
        }
}

# HTTPS 服务配置（使用IP时）
:{$HTTPS_PORT} {
        tls /etc/ssl/certs/letsencrypt/live/npm-6/fullchain.pem /etc/ssl/certs/letsencrypt/live/npm-6/privkey.pem
        reverse_proxy http://headscale:8080
}
```

#### DERP 配置

根据您的 DERP 服务器部署情况，创建适当的 `derper.json` 配置文件：

##### 使用域名的 DERP 配置

如果您使用域名部署了 DERP 服务器，请使用以下配置：

```json
{
    "Regions": {
        "901": {
            "RegionID": 901,
            "RegionCode": "My Region",
            "RegionName": "My Region DERP Server",
            "Nodes": [
                {
                    "Name": "derp1",
                    "RegionID": 901,
                    "DERPPort": 30443,
                    "HostName": "derp.yourdomain.com",
                    "IPv4": "YOUR_SERVER_IP",
                    "STUNPort": 3478,
                    "InsecureForTests": false
                }
            ]
        }
    }
}
```

##### 使用 IP 地址的 DERP 配置

如果您使用 IP 地址部署了 DERP 服务器，请使用以下配置：

```json
{
    "Regions": {
        "901": {
            "RegionID": 901,
            "RegionCode": "My Region",
            "RegionName": "My Region DERP Server",
            "Nodes": [
                {
                    "Name": "derp1",
                    "RegionID": 901,
                    "DERPPort": 30443,
                    "HostName": "YOUR_SERVER_IP",
                    "IPv4": "YOUR_SERVER_IP",
                    "STUNPort": 3478,
                    "InsecureForTests": true
                }
            ]
        }
    }
}
```

> [!WARNING]
> `InsecureForTests` 参数仅在使用自签名证书或 IP 地址配置时设置为 `true`。在生产环境中使用域名和有效证书时，应设置为 `false` 以确保安全。

### 构建 Headscale 镜像

在 `headscale-repo` 目录下创建 `Dockerfile`：

```dockerfile
# 多阶段构建: 第一阶段编译 Headscale
FROM golang:1.24-bookworm AS build
ARG VERSION=dev
ENV GOPATH=/go
WORKDIR /go/src/headscale

# 安装必要依赖
RUN apt-get update \
  && apt-get install --no-install-recommends --yes less jq \
  && rm -rf /var/lib/apt/lists/* \
  && apt-get clean
RUN mkdir -p /var/run/headscale

# 复制并下载依赖
COPY go.mod go.sum /go/src/headscale/
RUN go mod download

# 复制源码并编译
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go install -ldflags="-s -w -X github.com/juanfont/headscale/cmd/headscale/cli.Version=$VERSION" -a ./cmd/headscale && test -e /go/bin/headscale

# 第二阶段: 创建运行环境
FROM alpine:latest

# 配置 Caddy 服务端口
ENV HTTP_PORT="80"
ENV HTTPS_PORT="443"

# 设置工作目录
WORKDIR /staging
WORKDIR /web
WORKDIR /data

# 复制配置文件
COPY ./Caddyfile /staging/Caddyfile
COPY ./derper.json /web/derper.json
COPY ./init.sh /staging/init.sh

# 安装 Caddy 服务器
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories \
    && apk add --no-cache caddy

# 从构建阶段复制编译好的二进制文件
COPY --from=build /go/bin/headscale /bin/headscale
ENV TZ=Asia/Shanghai

# 创建运行目录
RUN mkdir -p /var/run/headscale

# 暴露 API 端口
EXPOSE 8080/tcp

# 设置启动命令
CMD ["/bin/sh", "/staging/init.sh"]
```

执行构建命令：

```bash
docker build -t headscale/headscale:v0.26.1 .
```

### Headscale 配置文件

编辑 `container-config/config.yaml` 文件，根据您的环境进行配置：

```yaml
# 基本配置
server_url: http://<PUBLIC_ENDPOINT>:8080  # 替换为您的公网 IP 或域名
listen_addr: 0.0.0.0:8080  # 监听所有网络接口

# TLS 配置
tls_cert_path: ""
tls_key_path: "/etc/ssl/certs/letsencrypt/live/npm-6/privkey.pem"


# DERP 配置
derp:
  server:
    enabled: false  # 默认禁用内置的 DERP 服务器
  urls:
    - http://localhost/web/derper.json
  paths: []
```

### 部署与启动

创建 `docker-compose.yaml` 文件：

```yaml
version: '3'

services:
  headscale:
    image: headscale/headscale:v0.26.1
    container_name: headscale
    restart: unless-stopped
    environment:
      - TZ=Asia/Shanghai
      - HTTPS_PORT=443
      - HTTP_PORT=80
    volumes:
      - ./container-config:/etc/headscale
      - ./container-data/data:/var/lib/headscale
      - ./Caddyfile:/data/Caddyfile
      - ./derper.json:/web/derper.json
      - /root/nginx-proxy-manager/letsencrypt:/etc/ssl/certs/letsencrypt:ro
    ports:
      - "8080:443"
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
    security_opt:
      - no-new-privileges:true
```

最后，启动 Headscale 服务：

```bash
docker-compose up -d
```

### 验证部署

启动成功后，您可以通过以下方式验证部署：

```bash
# 查看容器运行状态
docker ps | grep headscale

# 查看容器日志
docker logs headscale
```

如果一切正常，您现在应该有一个功能完整的 Headscale 服务器，可以开始添加客户端并构建您的 Tailscale 网络了。

> [!TIP]
> Headscale 部署成功后，您可以通过 `headscale namespaces create default` 命令创建默认命名空间，然后使用 `headscale -n default preauthkeys create --reusable --expiration 24h` 生成预授权密钥，方便客户端连接。

## 配置 Tailscale 客户端

完成 Headscale 服务器部署后，需要配置各种设备上的 Tailscale 客户端以连接到您的私有控制服务器。以下是针对不同平台的详细配置指南。

可以使用以下指令安装 tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### Windows 客户端配置

Windows 用户可以按照以下步骤配置 Tailscale 客户端：

1. 从 [Tailscale 官网](https://tailscale.com/download) 下载并安装 Windows 客户端
2. 打开命令提示符 (CMD) 并输入以下命令：

```bash
# 使用您的 Headscale 服务器地址登录
tailscale login --login-server http://公网IP:8080
```

3. 完成登录后，可以通过以下方式注册节点：
   - 在 Headscale UI 的设备页面中填入生成的 `nodekey:...` 信息
   - 或使用 Headscale 服务器上的命令行注册节点：

```bash
docker exec -it headscale headscale nodes register --user private --key nodekey:bd3f70
```

### Linux 客户端配置

Linux 设备的配置步骤如下：

```bash
# 使用您的 Headscale 服务器地址登录
tailscale login --login-server http://公网IP:8080

# 在 Headscale UI 或通过命令行注册节点
docker exec -it headscale headscale nodes register --user private --key nodekey:bd3f70

# 可选：强制重新认证
tailscale up --force-reauth --login-server=http://公网IP:8080
```

### Android 客户端配置

安装后打开应用，点击 `Get start`，然后选择 `Sign in with others`。应用会自动跳转到浏览器，复制最后一行命令到 Headscale 服务器执行（将 `NAMESPACE` 替换为您的命名空间）。成功后设备会自动连接到网络。

### Tailscale 客户端常用参数说明

Tailscale 客户端提供了丰富的参数选项，用于控制连接行为、网络配置和安全设置。下面是 `tailscale login` 和 `tailscale up` 命令中最常用的参数详解：

#### `tailscale login` 常用参数

```bash
# 指定控制服务器地址（用于自托管 Headscale）
--login-server value       # 控制服务器的基础 URL（默认为 https://controlplane.tailscale.com）

# DNS 配置
--accept-dns=<bool>        # 是否接受从控制面板下发的 DNS 配置（默认为 true）

# 路由相关
--accept-routes=<bool>     # 是否接受其他 Tailscale 节点通告的路由（默认为 false）
--advertise-routes value   # 向其他节点通告路由（用逗号分隔，如 "10.0.0.0/8,192.168.0.0/24"）
--advertise-exit-node      # 将该节点设置为 tailnet 互联网流量的出口节点

# 身份与安全
--auth-key value           # 节点授权密钥，可用于自动注册
--hostname value           # 使用自定义主机名替代操作系统提供的主机名
--shields-up=<bool>        # 阻止入站连接（默认为 false）
```

#### `tailscale up` 常用参数

```bash
# 所有 tailscale login 的参数也适用于 tailscale up
# 另外，tailscale up 特有的重要参数包括：

# 认证相关
--force-reauth=<bool>      # 强制重新认证（警告：会断开 Tailscale 连接，不要在远程 SSH/RDP 会话中使用）

# 出口节点配置
--exit-node value          # 指定用于互联网流量的出口节点（IP 或基本名称）
--exit-node-allow-lan-access=<bool> # 使用出口节点时允许直接访问本地网络（默认为 false）

# 其他选项
--reset=<bool>             # 将未指定的设置重置为默认值（默认为 false）
--ssh=<bool>               # 运行 SSH 服务器，按 tailnet 管理员声明的策略允许访问（默认为 false）
--netfilter-mode value     # 网络过滤模式（可选 on、nodivert、off，默认为 on）
--stateful-filtering=<bool> # 对转发的数据包应用有状态过滤（子网路由器、出口节点等）（默认为 false）
```

> [!TIP]
> 当设置子网路由时，`--advertise-routes` 参数指定要分享的本地网络，而其他节点需要使用 `--accept-routes` 参数才能访问这些网络。出口节点功能需要 `--advertise-exit-node` 参数来启用。

### 启用子网路由

如果您需要将某个客户端作为子网路由器，允许其他 Tailscale 节点访问该客户端所在网络的其他设备，可以按照以下步骤配置：

1. **在子网路由器客户端上**：

```bash
tailscale up \
--advertise-routes 192.168.1.0/24 \
--advertise-exit-node \
--reset \
--force-reauth \
--accept-dns=true \
--accept-routes=true \
--snat-subnet-routes=true \
--exit-node-allow-lan-access=true \
--hostname=NanoPC-T6 \
--login-server=http://<公网IP地址>:8080
```

> [!NOTE]
> - 将 `192.168.1.0/24` 替换为您要分享的实际本地网络 IP 范围
> - `--snat-subnet-routes=true` 参数使路由器对转发的流量执行源地址转换，这在多数网络环境中是必要的
> - `--hostname` 参数可帮助您在 Tailscale 网络中轻松识别路由器设备
> - `--accept-dns=true` 当域名解析失败时请开启该选项。

2. **在 Headscale 服务器上启用路由**：

```bash
# 查看路由信息（-i 参数后是节点ID）
docker exec -it headscale headscale routes list -i 2

# 启用特定路由（-r 参数后是路由，不是路由索引）
docker exec -it headscale headscale routes enable -i 2 -r 192.168.1.0/24
```

3. **在其他需要访问该子网的客户端上**：

```bash
tailscale up \
--accept-routes \
--exit-node-allow-lan-access=true \
--force-reauth \
--login-server=http://<公网IP地址>:8080
```

> [!TIP]
> - `--accept-routes` 参数使客户端能够接收并使用子网路由
> - `--exit-node-allow-lan-access=true` 允许在使用出口节点时保留对本地网络的访问
> - 如果路由不起作用，可能需要在 Linux 路由器上启用 IP 转发：`echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf && sudo sysctl -p`

### 配置出口节点

出口节点是一个特殊的 Tailscale 节点，可以将网络流量从 Tailscale 网络路由到互联网。这对于在不安全的网络环境中保护您的连接或绕过区域性网络限制非常有用。

#### 1. 设置出口节点

在您选择作为出口节点的设备上执行：

```bash
tailscale up \
--advertise-exit-node \
--hostname=exit-node \
--login-server=http://<公网IP地址>:8080
```

> [!NOTE]
> - 出口节点应该具有良好的网络连接和足够的带宽
> - 为了更好的安全性，建议在服务器而不是个人计算机上设置出口节点
> - 在某些地区，提供出口节点服务可能需要遵守当地法规

#### 2. 在 Headscale 服务器上批准路由规则

路由需要在 Headscale 服务器上得到批准：

```bash
# 查看节点信息，确认哪个节点提供了出口节点服务
docker exec -it headscale headscale nodes list-routes

# 启用节点的特定路由
docker exec -it headscale headscale nodes approve-routes -i 1 -r 192.168.10.0/24
```

#### 3. 在客户端使用出口节点

要通过出口节点路由流量，在客户端执行：

```bash
# 查看可用的出口节点
tailscale status

# 使用特定的出口节点（通过其 Tailscale IP 或主机名）
tailscale up \
--exit-node=exit-node \
--exit-node-allow-lan-access=true \
--login-server=http://<公网IP地址>:8080

# 或者通过 Tailscale IP 地址指定
# tailscale up --exit-node=100.x.y.z --login-server=http://<公网IP地址>:8080

# 停止使用出口节点
# tailscale up --exit-node= --login-server=http://<公网IP地址>:8080
```

> [!TIP]
> - 使用 `--exit-node-allow-lan-access=true` 可以在启用出口节点的同时继续访问本地网络设备
> - 可以通过访问 https://ipinfo.io 或类似服务来验证您的流量是否确实通过出口节点
> - 如需临时使用出口节点，可以使用 CLI 命令而非永久配置：`tailscale set --exit-node=100.x.y.z`

## Headscale 常用命令参考

以下是一些管理 Headscale 网络常用的命令：

### 命名空间管理

```bash
# 查看所有命名空间
headscale users list

# 创建命名空间
headscale users create user

# 删除命名空间
headscale users destroy user

# 重命名命名空间
headscale users rename user newuser
```

### 节点管理

```bash
# 列出所有节点
headscale node list

# 列出所有节点，同时显示标签信息
headscale node ls -t

# 查看特定命名空间下的节点
headscale -u user node ls

# 根据 ID 删除节点
headscale node delete -i<ID>  # 如 headscale nodes delete -i=2

# 为节点添加标签
headscale node tag -i=2 -t=tag:test  # 给 ID 为 2 的节点设置标签 tag:test

# 添加/注册节点
headscale nodes register --user private --key nodekey:bd3f70
```

### 路由管理

```bash
# 列出节点的所有路由信息
docker exec -it headscale headscale nodes list-routes  # 列出节点 3 的路由

# 启用节点的特定路由
docker exec -it headscale headscale nodes approve-routes -i 1 -r 192.168.10.0/24
```

### 预授权密钥管理

```bash
# 查看命名空间的预授权密钥
headscale -u private preauthkeys list

# 创建预授权密钥（24小时有效期）
headscale preauthkeys create -e 24h -n default
```

### API 密钥管理

```bash
# 创建 API 密钥
# 注意：创建时需要记录完整值，后续只能查询前缀
headscale apikeys create  # 值类似于 zs3NTt7G0w.pDWtOtaVx_mN9SzoM24Y02y6tfDzz5uysRHVxwJc1o4

# 以 JSON 格式查询 API 密钥
headscale apikeys list -o=json
```

> [!TIP]
> API 密钥用于客户端和 Headscale 之间的 HTTP 认证。使用时需要在 HTTP 请求头部设置 `Authorization: Bearer YOUR_API_KEY`。

