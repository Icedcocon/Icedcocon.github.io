# 

title: Nginx 快速入门
date: 2025-06-23T09:43:19+08:00
lastmod: 2025-06-23T09:43:19+08:00
draft: false
tags:
  - Nginx
  - Web服务器
  - 网络
categories:
  - 网络
series:
  - 服务器运维
---

## 概述

Nginx（发音为 "engine-x"）是一款高性能的开源 Web 服务器，同时也是一个反向代理服务器、负载均衡器、邮件代理服务器和通用的 TCP/UDP 代理服务器。它由 Igor Sysoev 最初为俄罗斯访问量第二的网站 Rambler.ru 开发，并于 2004 年公开发布。Nginx 以其高并发、低内存消耗和高稳定性而闻名，在全球范围内被广泛应用。

> [!TIP]
> Nginx 的核心优势在于其异步、事件驱动的架构，这使其能够以极高的效率处理大量并发连接。

本文将作为一份快速入门指南，带您了解 Nginx 的版本、安装方法、基本操作以及核心配置文件的结构和常用指令。

## Nginx 版本

Nginx 开源版本主要分为两种：

- **主线版 (Mainline)**：这是最新的版本，包含了最新的功能和正在开发中的实验性模块，但可能存在一些未被发现的 bug。
- **稳定版 (Stable)**：经过长时间测试的版本，稳定性高，不包含实验性新功能，是生产环境的推荐选择。

## Nginx 安装

Nginx 提供了多种安装方式，最常见的是使用操作系统的包管理器或从源码编译安装。

### 通过包管理器安装 (推荐)

这是最简单快捷的安装方式，推荐大多数用户使用。

#### 在 CentOS/RHEL 上安装

你可以使用 `yum` 直接安装，但为了确保安装的是最新版本，推荐使用 Nginx 官方的 yum 仓库。

1.  **安装 `yum-utils`**
    ```bash
    sudo yum install yum-utils
    ```

2.  **添加 Nginx 仓库**
    创建一个新文件 `/etc/yum.repos.d/nginx.repo`。
    ```bash
    sudo vi /etc/yum.repos.d/nginx.repo
    ```
    根据你的需要，将以下内容之一添加到文件中：

    **稳定版 (Stable)**
    ```ini
    [nginx-stable]
    name=nginx stable repo
    baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
    gpgcheck=1
    enabled=1
    gpgkey=https://nginx.org/keys/nginx_signing.key
    module_hotfixes=true
    ```

    **主线版 (Mainline)**
    ```ini
    [nginx-mainline]
    name=nginx mainline repo
    baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
    gpgcheck=1
    enabled=1
    gpgkey=https://nginx.org/keys/nginx_signing.key
    module_hotfixes=true
    ```

3.  **安装 Nginx**
    ```bash
    sudo yum install nginx
    ```

#### 在 Debian/Ubuntu 上安装

你可以使用 `apt` 直接安装，同样地，也推荐使用 Nginx 官方的 apt 仓库。

1.  **安装前置依赖**
    ```bash
    sudo apt install curl gnupg2 ca-certificates lsb-release debian-archive-keyring
    ```

2.  **导入 Nginx 官方签名密钥**
    ```bash
    curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
    | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
    ```

3.  **设置 Nginx 仓库**
    根据你的需要，选择稳定版或主线版。

    **稳定版 (Stable)**
    ```bash
    echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
    http://nginx.org/packages/debian `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
    ```

    **主线版 (Mainline)**
    ```bash
    echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
    http://nginx.org/packages/mainline/debian `lsb_release -cs` nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
    ```

4.  **设置仓库优先级 (可选但推荐)**
    为了优先使用 Nginx 官方仓库的包，可以创建一个优先级配置文件。
    ```bash
    echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
    | sudo tee /etc/apt/preferences.d/99nginx
    ```

5.  **安装 Nginx**
    ```bash
    sudo apt update
    sudo apt install nginx
    ```

### 从源码编译安装 (高级)

从源码编译安装允许你自定义安装目录、添加或移除特定模块，但过程相对繁琐。

#### 1. 安装编译依赖

- **PCRE (Perl Compatible Regular Expressions)**: Nginx 的 rewrite 模块和 location 指令需要 PCRE 的支持。
  ```bash
  # 以 pcre2-10.42 为例
  wget https://github.com/PCRE2Project/pcre2/releases/download/pcre2-10.42/pcre2-10.42.tar.gz
  tar -zxf pcre2-10.42.tar.gz
  cd pcre2-10.42
  ./configure
  make
  sudo make install
  ```

- **zlib**: Nginx 的 gzip 模块需要 zlib 的支持。
  ```bash
  # 以 zlib-1.2.13 为例
  wget http://zlib.net/zlib-1.2.13.tar.gz
  tar -zxf zlib-1.2.13.tar.gz
  cd zlib-1.2.13
  ./configure
  make
  sudo make install
  ```

- **OpenSSL**: Nginx 的 HTTPS (SSL/TLS) 支持需要 OpenSSL。
  ```bash
  # 以 openssl-1.1.1t 为例
  wget http://www.openssl.org/source/openssl-1.1.1t.tar.gz
  tar -zxf openssl-1.1.1t.tar.gz
  cd openssl-1.1.1t
  ./Configure darwin64-x86_64-cc --prefix=/usr
  make
  sudo make install
  ```

#### 2. 下载 Nginx 源码

从 Nginx 官网下载你想要的版本。
```bash
# 下载稳定版
wget https://nginx.org/download/nginx-1.24.0.tar.gz
tar zxf nginx-1.24.0.tar.gz
cd nginx-1.24.0
```

#### 3. 配置与编译

使用 `./configure` 命令进行编译前配置，你可以通过 `--help` 查看所有可用选项。

下面是一个官网的例子，指定了依赖库的源码路径：
```bash
./configure \
   --sbin-path=/usr/local/nginx/nginx \
   --conf-path=/usr/local/nginx/nginx.conf \
   --pid-path=/usr/local/nginx/nginx.pid \
   --with-pcre=../pcre2-10.42 \
   --with-zlib=../zlib-1.2.13 \
   --with-http_ssl_module \
   --with-stream \
   --with-mail=dynamic \
   --add-module=/usr/build/nginx-rtmp-module \
   --add-dynamic-module=/usr/build/3party_module
```

编译并安装：
```bash
make
sudo make install
```
> [!NOTE]
> 编译时 `--with-pcre=../pcre2-10.42` 指向的是解压后的源码目录，而不是安装路径。

**常用配置参数说明：**

| 参数 (`Parameter`)      | 说明 (`Description`)                                 |
| ----------------------- | ---------------------------------------------------- |
| `--prefix=`             | 指定安装目录                                         |
| `--sbin-path=`          | 指定 Nginx 可执行文件路径                            |
| `--conf-path=`          | 指定配置文件路径                                     |
| `--pid-path=`           | 指定 pid 文件路径                                    |
| `--error-log-path=`     | 指定错误日志文件路径                                 |
| `--http-log-path=`      | 指定 HTTP 访问日志文件路径                           |
| `--user=`               | 指定运行 Nginx worker 进程的用户                     |
| `--group=`              | 指定运行 Nginx worker 进程的用户组                   |
| `--with-pcre=`          | 指定 PCRE 库的源码路径                               |
| `--with-pcre-jit`       | 开启 PCRE 的 JIT (Just-in-time compilation) 支持       |
| `--with-zlib=`          | 指定 zlib 库的源码路径                               |
| `--with-http_ssl_module`| 启用 HTTPS 支持模块                                  |

## 基本操作

以下是一些管理 Nginx 服务的常用命令。

```bash
# 启动 Nginx
sudo nginx

# 停止 Nginx (快速关闭)
sudo nginx -s stop

# 优雅地停止 Nginx (处理完当前请求后再关闭)
sudo nginx -s quit

# 重新加载配置文件 (服务不中断)
sudo nginx -s reload

# 检查配置文件语法是否正确
sudo nginx -t

# 查看 Nginx 版本及编译参数
sudo nginx -V
```

安装后，你可以通过访问服务器 IP 或 `127.0.0.1` 来验证 Nginx 是否成功运行。
```bash
curl -I 127.0.0.1
```
如果看到类似 `Server: nginx/...` 的响应头，说明 Nginx 已成功启动。

## Nginx 配置文件详解

> [!TIP]
> Nginx 的强大之处在于其高度灵活的配置文件。默认配置文件通常位于 `/etc/nginx/nginx.conf` (包管理器安装) 或你编译时指定的路径。
> 使用 `nginx -t` 命令可以查看配置文件的加载路径并检查语法。

### 配置文件结构

Nginx 配置文件由一系列**指令 (directive)** 组成。指令由名称和参数构成，以分号 `;` 结尾。为了便于组织，指令被放在称为**块 (block)** 的上下文中，块由大括号 `{}` 界定。

一个典型的配置文件结构如下：

```nginx
# main (全局) 块
user  nginx;
worker_processes  1;

events {
    # events 块
    worker_connections 1024;
}

http {
    # http 块

    server {
        # server 块

        location / {
            # location 块
        }
    }

    server {
        # 另一个 server 块
    }
}
```

### `main` (全局) 块

这部分配置影响 Nginx 的全局行为。

```nginx
# 指定运行 Nginx worker 进程的用户和用户组。
# 如果注释掉或设为 nobody，则所有用户都可以运行。
user nginx;

# 指定 Nginx 启动的 worker 进程数量。通常设为 CPU 核心数或 "auto"。
worker_processes auto;

# 错误日志的存放路径和日志级别 (debug, info, notice, warn, error, crit)。
error_log /var/log/nginx/error.log warn;

# Nginx 主进程的 PID 文件存放路径。
pid /var/run/nginx.pid;
```

### `events` 块

`events` 块用于配置 Nginx 的网络连接处理机制。

```nginx
events {
    # 指定使用的网络 I/O 模型。Linux 下推荐使用 epoll，FreeBSD 下推荐使用 kqueue。
    # use epoll;
    
    # 每个 worker 进程允许的最大并发连接数。
    # 理论上服务器的最大连接数 = worker_processes * worker_connections。
    worker_connections 1024;
}
```

### `http` 块

`http` 块是 Nginx 配置中最核心和最复杂的部分，它定义了 HTTP 协议相关的设置。一个 `http` 块可以包含多个 `server` 块。

```nginx
http {
    # 使用 include 指令可以引入其他配置文件，增强可维护性。
    include       /etc/nginx/mime.types;
    # 如果根据文件扩展名无法确定 MIME 类型，则使用此默认类型。
    default_type  application/octet-stream;

    # 定义访问日志的格式。
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    # 指定访问日志的存放路径，并使用上面定义的 main 格式。
    access_log  /var/log/nginx/access.log  main;

    # 开启高效文件传输模式，它会调用 sendfile() 系统调用来直接在内核空间传递文件。
    sendfile        on;
    
    # 保持长连接的超时时间，单位是秒。
    keepalive_timeout  65;

    # 开启 Gzip 压缩，可以显著减少传输的数据量。
    # gzip on;

    # upstream 指令用于定义一组后端服务器，常用于反向代理和负载均衡。
    upstream my_backend {
        # ip_hash: 根据客户端 IP 的哈希值分配服务器，确保同一客户端的请求总是被发送到同一台服务器。
        ip_hash; 
        
        # weight: 设置服务器的权重，权重越高的服务器被分配到的请求越多。
        server 192.168.50.11:80 weight=3;
        server 192.168.50.12:80;
        server 192.168.50.13:80;
    }

    # 引入其他 server 配置文件，通常在 /etc/nginx/conf.d/ 目录下。
    include /etc/nginx/conf.d/*.conf;
}
```

### `server` 块

`server` 块定义了一个**虚拟主机**，用于处理特定域名或 IP 地址的请求。

```nginx
server {
    # 监听的 IP 和端口。
    # listen 80; # 监听所有 IPv4 地址的 80 端口
    # listen [::]:80; # 监听所有 IPv6 地址的 80 端口
    listen 80;

    # 指定虚拟主机的域名，可以有多个，支持通配符和正则表达式。
    # server_name example.org www.example.org; # 精确匹配
    # server_name *.example.org;             # 通配符匹配
    # server_name ~^www\d+\.example\.net$;   # 正则匹配
    server_name localhost;

    # location 块，用于匹配请求的 URI。
    location / {
        # ... 见下一节 ...
    }
    
    # 定义错误页面。当发生 500, 502, 503, 504 错误时，内部重定向到 /50x.html。
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

### `location` 块

`location` 块是 `server` 块的一部分，它根据请求的 URI (Uniform Resource Identifier) 来决定如何处理请求，是 Nginx 配置中至关重要的一环。

`location` 匹配规则有不同的优先级，语法如下：
`location [modifier] /uri/ { ... }`

| 修饰符 (`modifier`) | 含义                                                         |
| ------------------- | ------------------------------------------------------------ |
| (无)                | 前缀匹配。匹配以 `/uri/` 开头的请求。                          |
| `=`                 | 精确匹配。URI 必须与 `/uri/` 完全相同。                      |
| `^~`                | "非正则"前缀匹配。如果此项匹配成功，则停止搜索其他 `location`。 |
| `~`                 | 正则表达式匹配 (区分大小写)。                                  |
| `~*`                | 正则表达式匹配 (不区分大小写)。                                |

**匹配顺序：**
1.  `=` 精确匹配
2.  `^~` "非正则"前缀匹配
3.  正则表达式匹配 (按文件中出现的顺序)
4.  (无) 前缀匹配 (匹配最长者)

```nginx
location / {
    # 指定静态文件的根目录。
    root /usr/share/nginx/html;

    # 指定默认页面。如果请求的是一个目录，Nginx 会依次查找这些文件。
    index index.html index.htm;
}

# 示例：
location = / {
    # 只匹配根目录的请求，优先级最高。
}

location ^~ /images/ {
    # 匹配任何以 /images/ 开头的请求，且停止后续的正则匹配。
    # 适合用于静态资源目录。
}

location ~* \.(gif|jpg|jpeg)$ {
    # 匹配任何以 .gif, .jpg, 或 .jpeg 结尾的请求 (不区分大小写)。
    # 可用于设置图片缓存策略。
    expires 30d;
}
```

## 总结

本文档提供了 Nginx 的快速入门指南，涵盖了从安装到核心配置的各个方面。通过理解 `http`, `server`, 和 `location` 等核心块的作用，你就可以开始构建和管理自己的 Web 服务了。Nginx 的功能远不止于此，深入学习反向代理、负载均衡、缓存和安全设置将为你打开一扇新的大门。希望这篇指南能为你打下坚实的基础。

