# RustDesk 快速开始


本文将引导您快速搭建一套带有 Web 用户管理界面的 RustDesk 远程桌面服务。我们将使用 `lejianwen/rustdesk-server-s6` 镜像，该镜像集成了 RustDesk 服务器和 Web API，可以方便地进行用户管理。

> [!NOTE]
> 部署前，请确保您的服务器已经安装了 Docker 和 Docker Compose，并拥有一个公网 IP 地址。

## 服务端部署

### 1. 创建并编辑 `docker-compose.yml`

首先，创建一个目录用于存放 RustDesk 的数据和配置文件，例如 `mkdir -p /data/rustdesk && cd /data/rustdesk`。

然后，在该目录下创建一个 `docker-compose.yml` 文件，并填入以下内容：

```yaml
version: '3'

networks:
  rustdesk-net:
    external: false

services:
  rustdesk:
    image: lejianwen/rustdesk-server-s6:latest
    container_name: rustdesk-server
    restart: unless-stopped
    ports:
      - "21114:21114" # Web API
      - "21115:21115" # TCP, for ID server
      - "21116:21116" # TCP, for ID server
      - "21116:21116/udp" # UDP, for ID server
      - "21117:21117" # TCP, for relay server
      - "21118:21118" # TCP, for web client
      - "21119:21119" # TCP, for web client
    environment:
      - "TZ=Asia/Shanghai"
      - "RELAY=your_server_ip:21117"
      - "MUST_LOGIN=Y"
      - "RUSTDESK_API_RUSTDESK_ID_SERVER=your_server_ip:21116"
      - "RUSTDESK_API_RUSTDESK_RELAY_SERVER=your_server_ip:21117"
      - "RUSTDESK_API_RUSTDESK_API_SERVER=http://your_server_ip:21114"
      - "RUSTDESK_API_KEY_FILE=/data/id_ed25519.pub"
      # 请务必将 'your_jwt_secret' 替换为一个足够复杂的随机字符串
      - "RUSTDESK_API_JWT_KEY=your_jwt_secret"
    volumes:
      - ./server:/data
      - ./api:/app/data
    networks:
      - rustdesk-net
```

> [!WARNING]
> 在启动容器前，请务必将文件中的所有 `your_server_ip` 替换成您服务器的公网 IP 地址，并将 `your_jwt_secret` 替换为一个强随机字符串作为 JWT 密钥。

### 2. 启动服务

在 `docker-compose.yml` 文件所在目录执行以下命令启动服务：

```bash
docker-compose up -d
```

服务启动后，会在 `server` 目录下自动生成密钥文件 `id_ed25519.pub`。

### 3. 获取公钥

执行以下命令查看并复制您的公钥（Key），后续客户端配置需要用到。

```bash
cat /data/rustdesk/server/id_ed25519.pub
```

## 登录管理界面

服务启动成功后，即可通过浏览器访问 Web 管理界面。

> [!TIP]
> Web 管理后台地址为: `http://your_server_ip:21114`

此版本的 RustDesk 服务默认没有管理员账户，第一个注册的用户将自动成为管理员。请访问管理后台并立即注册您的管理员账户。

### 重置管理员密码

如果您忘记了管理员密码，可以通过以下命令进入容器重置。将 `<你的新密码>` 部分替换为您要设置的新密码。

```bash
docker-compose exec rustdesk ./apimain reset-admin-pwd <你的新密码>
```

## 客户端配置

最后，在您的 RustDesk 客户端上，配置连接信息以使用您自己的服务器。

> [!WARNING]
> 由于我们设置了 `MUST_LOGIN=Y`，客户端 **必须** 将上一步获取到的公钥（`id_ed25519.pub` 的内容）配置到 Key 字段，否则无法对客户端进行远程。

1.  点击 ID 右侧的 `...` 菜单，选择 `ID/中继服务器`。
2.  在 **ID 服务器** 字段填入您的服务器 IP 地址（例如 `your_server_ip`）。
3.  **中继服务器** 字段留空，它会自动使用 ID 服务器的地址和默认端口 `21117`。
4.  将您在上一步获取到的公钥（Key）内容粘贴到 **Key** 字段。
5.  点击 **确定** 保存设置。

配置完成后，您可以在客户端主界面点击 `设置` -> `账户`，使用您在 Web 界面注册的用户名和密码进行登录，之后便可正常使用。


