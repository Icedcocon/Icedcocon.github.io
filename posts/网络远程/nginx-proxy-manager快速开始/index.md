# Nginx Proxy Manager 快速上手指南


Nginx Proxy Manager (NPM) 是一款强大的 Nginx 图形化管理工具，它提供了简洁的 Web UI，让反向代理、SSL 证书管理（包括自动续签）等操作变得直观、高效，尤其适合不熟悉 Nginx 配置的开发者和运维人员。

# 部署方式

官方推荐使用 Docker 进行部署，这极大地简化了安装、升级和迁移过程。本节将介绍从基础到高级的多种部署方案。

## 基础部署 (使用 SQLite)

这是最简单、最快速的部署方式，使用内置的 SQLite 数据库存储配置，适合大多数个人和开发场景。

1.  创建 `docker-compose.yml` 文件：

```yaml
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    network_mode: "host"
    # ports:
    #   - '80:80'
    #   - '443:443'
    #   - '81:81'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```
    
> [!NOTE]
> - **镜像版本**: 示例中使用 `latest` 标签。为保证生产环境稳定性，建议将 `latest` 替换为具体的版本号，如 `2.12.1`。
> - **端口映射**: `ports` 配置将容器的 `80`, `443`, `81` 端口映射到主机。请确保主机防火墙已放行这些端口。
> - **持久化存储**: `volumes` 配置用于持久化存储 NPM 的配置数据和 SSL 证书，**至关重要，请勿随意修改或删除**。
> - **Host 网络模式**: 对于内网穿透等特殊场景，可将 `ports` 配置替换为 `network_mode: "host"`，使容器直接使用主机网络，简化网络配置。

2.  启动服务：

```bash
docker-compose up -d
```

## 高级部署 (使用外部数据库)

对于生产环境或需要更高稳定性和扩展性的场景，推荐使用独立的数据库服务（如 MariaDB/MySQL 或 PostgreSQL）。

> [!WARNING]
> 当同时设置了 MySQL/PostgreSQL 和 SQLite 的环境变量时，NPM 会优先使用外部数据库。

### 使用 MariaDB / MySQL

1.  **环境要求**:
    -   MySQL >= v5.7.8
    -   MariaDB >= v10.2.7

2.  创建 `docker-compose.yml` 文件，将 NPM 和 MariaDB 数据库容器一同编排：

```yaml
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
      - '81:81'
    environment:
      DB_MYSQL_HOST: "db"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "npm"
      DB_MYSQL_PASSWORD: "npm"
      DB_MYSQL_NAME: "npm"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    depends_on:
      - db

  db:
    image: 'jc21/mariadb-aria:latest'
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: 'npm' # 请修改为更安全的密码
      MYSQL_DATABASE: 'npm'
      MYSQL_USER: 'npm'
      MYSQL_PASSWORD: 'npm' # 请修改为更安全的密码
    volumes:
      - ./mysql:/var/lib/mysql
```

### 使用 PostgreSQL

1.  创建 `docker-compose.yml` 文件：

```yaml
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
      - '81:81'
    environment:
      DB_POSTGRES_HOST: 'db'
      DB_POSTGRES_PORT: '5432'
      DB_POSTGRES_USER: 'npm'
      DB_POSTGRES_PASSWORD: 'npmpass' # 请修改为更安全的密码
      DB_POSTGRES_NAME: 'npm'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    depends_on:
      - db
      
  db:
    image: 'postgres:latest'
    restart: unless-stopped
    environment:
      POSTGRES_USER: 'npm'
      POSTGRES_PASSWORD: 'npmpass' # 请修改为更安全的密码
      POSTGRES_DB: 'npm'
    volumes:
      - ./postgres:/var/lib/postgresql/data
```

## 在 ARM 设备 (如树莓派) 上部署

NPM 的 Docker 镜像原生支持 `amd64`, `arm64`, `armv7` 架构，因此你可以在树莓派等 ARM 设备上直接使用上述部署命令，无需任何特殊配置。

> [!TIP]
> `jc21/mariadb-aria:latest` 镜像在部分 ARM 设备上可能存在兼容性问题。如果遇到，可以尝试将其替换为 `yobasystems/alpine-mariadb:latest` 镜像。

## 持久化目录结构解析

通过 `volumes` 挂载 `./data` 和 `./letsencrypt` 目录是确保 NPM 服务配置和 SSL 证书在容器重启或更新后不丢失的关键。首次启动后，你的项目目录将生成如下结构：

```
.
├── data
│   ├── access
│   ├── custom_ssl
│   ├── database.sqlite
│   ├── keys.json
│   ├── letsencrypt-acme-challenge
│   ├── logs
│   └── nginx
├── docker-compose.yaml
└── letsencrypt
    ├── accounts
    ├── archive
    ├── credentials
    ├── live
    ├── renewal
    └── renewal-hooks
```

-   **`./data`**: 存储 NPM 的核心应用数据。
    -   `database.sqlite`: 在使用 SQLite 数据库时，这是最重要的文件，包含了你所有的用户、代理主机、证书等配置。
    -   `nginx/`: NPM 根据你在 UI 上的配置自动生成的底层 Nginx 配置文件。排查问题时，可以查看这里的文件来理解实际的 Nginx 规则。
    -   `logs/`: Nginx 的访问日志和错误日志，按代理主机和日期分割，是监控和审计的重要依据。
-   **`./letsencrypt`**: 存储所有由 Let's Encrypt 管理的 SSL 证书数据。
    -   `archive/`: 存放所有历史版本的证书文件。每次成功续期，新的证书和私钥都会保存在这里。
    -   `live/`: **这是最核心的目录**。它通过符号链接（快捷方式）指向 `archive/` 目录中当前最新、有效的证书文件。**外部应用应始终读取此目录下的证书**，以确保在证书续期后能无缝切换到新证书。

> [!IMPORTANT]
> 务必将这两个目录加入版本控制的忽略列表（如 `.gitignore`），并定期备份，尤其是 `data` 目录和 `letsencrypt` 目录。

# 初始配置与登录

服务首次启动时，NPM 会进行初始化，包括生成 JWT 密钥、创建数据库表结构和设置默认管理员。此过程可能需要几分钟。

服务启动后，通过 `http://<服务器IP>:81` 访问管理后台。

- **默认管理员账号**: `admin@example.com`
- **默认密码**: `changeme`

首次登录后，系统会强制要求修改管理员信息和密码。

> [!TIP]
> 你可以在 `docker-compose.yml` 文件中通过环境变量预设初始管理员信息，避免手动修改：
> ```yaml
> services:
>   app:
>     # ... other settings
>     environment:
>       INITIAL_ADMIN_EMAIL: my@example.com
>       INITIAL_ADMIN_PASSWORD: mypassword1
> ```

# 核心功能：配置反向代理

1.  登录 NPM 后台，点击 `Hosts` -> `Proxy Hosts`。
2.  点击 `Add Proxy Host` 添加新的代理规则。
3.  在 `Details` 标签页中填写：
    - **Domain Names**: 你的域名，如 `app.yourdomain.com`。
    - **Scheme**: `http` 或 `https`，取决于后端服务协议。
    - **Forward Hostname / IP**: 后端服务的 IP 地址或容器名，如 `192.168.1.10` 或 `my-app-container`。
    - **Forward Port**: 后端服务的端口，如 `8080`。
    - **Enable `Websockets Support`**: 如果后端服务使用 WebSocket（如 Jupyter, Portainer），请开启此选项。
4.  保存配置即可生效。

# SSL 证书自动化（以阿里云 DNS 为例）

NPM 深度集成了 Let's Encrypt，支持通过 DNS Challenge 方式全自动申请和续签 SSL 证书，无需公网 IP 或开放端口。

## 准备阿里云 AccessKey

1.  登录阿里云控制台，点击右上角头像下拉菜单中的 AccessKey，提示使用RAM用户操作
2.  创建 RAM 用户并授予 `AliyunDNSFullAccess` （和 `AliyunDomainFullAccess` ？）权限。
3.  为该用户创建 AccessKey，并记录 **AccessKey ID** 和 **AccessKey Secret**。

> [!TIP]
> 遵循最小权限原则，建议为 NPM 单独创建专用的、权限受限的 RAM 用户。

## 在 NPM 中配置证书申请

1.  在 NPM 后台，进入 `SSL Certificates` 页面，点击 `Add SSL Certificate` -> `Let's Encrypt`。
2.  在 **Domain Names** 中输入需要申请证书的域名（支持泛域名，如 `*.yourdomain.com`）。
3.  开启 **Use a DNS Challenge**。
4.  在 **DNS Provider** 列表中选择 `Aliyun`。
5.  将准备好的 **AccessKey ID** 和 **AccessKey Secret** 填入 `Credentials`。
6.  点击 `Save`，NPM 将自动通过阿里云 DNS API 完成域名验证并申请证书。

> [!NOTE]
> 证书和密钥文件默认保存在 `./letsencrypt` 目录下，可按需用于其他服务。

自动续期

- NPM 会在证书到期前自动执行续期流程，无需人工干预。
- 续期成功后，NPM 会自动重载 Nginx 服务以应用新证书。

> [!WARNING]
> 请确保持久化的 AccessKey 持续有效，若 AccessKey 失效或权限变更，将导致自动续期失败。

## 与其他项目共享 SSL 证书

NPM 强大的证书自动化功能可以服务于其他需要 HTTPS 的项目（如 Caddy, Traefik, 或其他自建服务）。核心思路是让其他项目能够访问 NPM 管理的 `letsencrypt` 目录。

### 共享证书目录

最直接的方式是通过 Docker 的 `volumes` 功能，将 NPM 的 `letsencrypt` 目录以**只读**模式挂载到其他服务的容器中。

例如，在另一个项目的 `docker-compose.yml` 中：
```yaml
services:
  my-other-app:
    image: some-app-image
    # ... other settings
    volumes:
      # 将 NPM 的 letsencrypt 目录挂载到容器内的 /etc/ssl/certs 目录
      # 注意：/path/to/your/npm/letsencrypt 应替换为实际的绝对路径
      # :ro 表示只读挂载，增强安全性
      - /path/to/your/npm/letsencrypt:/etc/ssl/certs:ro
```
在应用中，你只需将证书路径配置为容器内的 `/etc/ssl/certs/live/<your-domain>/fullchain.pem` 和 `/etc/ssl/certs/live/<your-domain>/privkey.pem`。

### 自动应用新证书

当 NPM 成功续期证书后，`letsencrypt/live` 目录下的符号链接会自动更新指向新文件。但正在运行的服务不会自动感知这一变化，需要重新加载配置才能应用新证书。

推荐使用 `renewal-hooks` (续期挂钩) 机制实现全自动化：

1.  在 NPM 项目的 `./letsencrypt/renewal-hooks/deploy` 目录下创建一个脚本，例如 `reload-services.sh`。

2.  赋予脚本执行权限：
    ```bash
    chmod +x ./letsencrypt/renewal-hooks/deploy/reload-services.sh
    ```

3.  编辑脚本内容，添加重载其他服务的命令。例如，重启另一个 Docker Compose 项目中的 `my-other-app` 服务：
    ```sh
    #!/bin/sh

    # 切换到其他项目的目录
    cd /path/to/other/project/

    # 使用 docker-compose 重启指定服务以应用新证书
    docker-compose restart my-other-app

    # 如果是单个容器，也可以直接用 docker restart
    # docker restart <other-container-name>
    ```

每次 NPM 为证书成功续期后，都会自动执行此脚本，从而实现其他依赖服务的无缝、及时更新。

# 进阶技巧与常见问题

自定义 Nginx 配置

NPM 允许在 Web UI 中为每个代理主机添加自定义 Nginx 配置，以满足特殊需求。

- **位置**: `Proxy Hosts` -> `Edit` -> `Advanced`
- **示例**: 设置根路径 `/` 永久重定向到 `/dashboard/`。
  ```nginx
  location / {
    return 301 /dashboard/;
  }
  ```

## 常见问题排查

### 登录时出现 502 Bad Gateway

部分网络环境下，NPM 容器可能因无法访问 `https://ip-ranges.amazonaws.com/ip-ranges.json` 而启动失败。

- **现象**: 访问后台报 502，查看容器日志 (`docker logs <container_name>`) 卡在 `Fetching...ip-ranges.json`。
- **解决方案**: 执行以下命令进入容器并移除该网络请求，然后重启容器。
  ```bash
  # 将 <container_name> 替换为你的 NPM app 容器名或ID
  docker exec <container_name> sed -i 's/\.then(internalIpRanges\.fetch)//g' /app/index.js
  docker restart <container_name>
  ```

---

参考资料：
- [Nginx Proxy Manager 官方部署文档](https://nginxproxymanager.com/setup/)
- [acme.sh 阿里云DNS自动申请证书说明](https://github.com/acmesh-official/acme.sh/wiki/%E8%AF%B4%E6%98%8E)
- [【玩转Docker】反向代理神器：Nginx Proxy Manager](https://www.golovin.cn/post/11)

