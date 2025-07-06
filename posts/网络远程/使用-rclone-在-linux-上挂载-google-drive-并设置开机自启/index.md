# 使用 rclone 在 Linux 上挂载 Google Drive 并设置开机自启


`rclone` 被誉为"云存储的瑞士军刀"，它是一个强大的命令行工具，用于管理、同步、挂载多种云存储服务。本文将详细介绍如何在 Ubuntu 22.04 系统上安装和配置 `rclone`，将 Google Drive 挂载到本地文件系统，并最终通过 `systemd` 服务实现开机自动挂载。

这种方法不仅可以将云端海量存储空间无缝地集成到本地，还能为你的应用（如 Plex, Jellyfin, Emby 等媒体服务器）提供稳定可靠的数据来源。

> [!TIP]
> 本文内容同样适用于 Debian、CentOS 等其他主流 Linux 发行版，只需将包管理命令（如 `apt`）替换为相应的命令（如 `yum`, `dnf`）即可。

---

## 第一步：创建 Google Drive API 凭据

为了让 `rclone` 有权限访问你的 Google Drive，你需要先在 Google API Console 创建一个专属的 API 凭据。

> [!NOTE]
> 此步骤参考了 [阿段的笔记 - Linux使用Rclone挂载 Google Drive 教程](https://aduan.cc/archives/20/) 中的方法。

1.  **登录 Google API Console**
    访问 [Google API Console](https://console.cloud.google.com/apis)，使用你的 Google 账号登录。

2.  **创建新项目**
    点击页面顶部的项目选择器，然后点击 "New Project"。输入一个你喜欢的项目名称（例如 `rclone-project`），然后点击 "CREATE"。

3.  **启用 Google Drive API**
    确保你选择了刚刚创建的项目，然后在顶部的搜索框中搜索 `Google Drive API` 并进入，点击 "ENABLE" 启用它。

4.  **配置 OAuth 同意屏幕**
    -   在左侧导航栏中，点击 "OAuth consent screen"。
    -   选择用户类型（User Type）为 **"External"**（外部），然后点击 "CREATE"。
    -   填写应用信息：
        -   **App name**: 随便填写，例如 `rclone`。
        -   **User support email**: 选择你的邮箱地址。
        -   **Developer contact information**: 填写你的邮箱地址。
    -   点击 "SAVE AND CONTINUE"，后续步骤直接使用默认设置，一直点击"CONTINUE"直到完成。
    -   最后，返回 OAuth 同意屏幕，点击 **"PUBLISH APP"** 来发布应用。

5.  **创建 OAuth 客户端 ID**
    -   在左侧导航栏中，点击 "Credentials"。
    -   点击页面上方的 "+ CREATE CREDENTIALS"，选择 **"OAuth client ID"**。
    -   在 "Application type" 中，选择 **"Desktop app"**（桌面应用）。
    -   名称可以保持默认，点击 "CREATE"。

6.  **保存凭据**
    创建成功后，你会看到一个弹窗，里面包含了你的 **Client ID** 和 **Client Secret**。请务必将这两串字符复制并妥善保存下来，后续配置 `rclone` 时会用到。

---

## 第二步：在 Ubuntu 上安装和配置 rclone

### 1. 安装 rclone

rclone 官方提供了一键安装脚本，非常方便。在你的 Ubuntu 终端中执行以下命令：

```bash
curl https://rclone.org/install.sh | sudo bash
```

### 2. 配置 rclone

安装完成后，运行配置命令：

```bash
rclone config
```

接下来，程序会引导你进行交互式配置。请按照以下步骤操作：

-   `No remotes found, make a new one?`
    输入 `n` 并回车，表示创建一个新的远程连接（New remote）。

-   `name>`
    为你的远程连接起个名字，例如 `GoogleDrive`。

-   `Storage>`
    会列出所有支持的云服务商。找到 `drive` (Google Drive) 对应的序号，输入该序号并回车。

-   `client_id>`
    粘贴你在第一步中保存的 **Client ID**。

-   `client_secret>`
    粘贴你在第一步中保存的 **Client Secret**。

-   `scope>`
    选择 `rclone` 的访问权限。通常我们选择 `1` (Full access all files)，输入 `1` 并回车。

-   `root_folder_id>` 和 `service_account_file>`
    直接按回车跳过。

-   `Edit advanced config?`
    输入 `n` 并回车，我们不需要进行高级配置。

-   `Use auto config?`
    因为我们是在没有图形界面的服务器上操作，所以这里必须选择 `n`。输入 `n` 并回车。

-   **获取授权 Token**
    选择 `n` 之后，`rclone` 会在终端里生成一段认证 URL 和一条命令。它会提示你：
    1.  在任何一台有浏览器的电脑上，安装 `rclone`。
    2.  打开那台电脑的终端（Windows 的 CMD 或 PowerShell，macOS 的 Terminal）。
    3.  运行 `rclone` 在你服务器终端上提示的那条 `rclone authorize ...` 命令。

    浏览器会自动打开一个 Google 登录和授权页面。登录并授权后，回到你本地电脑的终端，它会生成一长串 `token`。

-   **粘贴 Token**
    将本地电脑终端上生成的整段 `token` 结果（包含花括号 `{...}`）完整地复制下来，然后粘贴到你 Ubuntu 服务器的终端里，并回车。

-   `Configure this as a team drive?`
    如果你要挂载的是团队盘（Shared Drive），输入 `y` 并选择对应的盘。如果是个人盘，输入 `n`。

-   `Yes this is OK?`
    确认配置无误，输入 `y` 并回车。

最后输入 `q` 退出配置。至此，`rclone` 已成功连接到你的 Google Drive。

---

## 第三步：测试手动挂载

在配置自启动前，务必先手动挂载一次，以验证配置是否正确。

1.  **安装 FUSE（关键步骤）**
    FUSE (Filesystem in Userspace) 是 `rclone` 实现挂载功能的底层依赖。**如果缺少 FUSE，挂载命令将直接失败**，并提示 `fusermount3: executable file not found` 错误。
    ```bash
    sudo apt update && sudo apt install -y fuse3
    ```

2.  **允许其他用户访问（重要）**
    为了让系统服务（通常以 `root` 用户运行）或其他非 root 用户能访问挂载点，需要修改 FUSE 的配置。
    ```bash
    sudo vim /etc/fuse.conf
    ```
    找到 `#user_allow_other` 这一行，去掉行首的 `#` 保存退出。如果文件是空的，直接添加一行 `user_allow_other` 即可。

3.  **创建挂载点**
    创建一个空目录作为挂载点。建议使用 `/mnt` 目录下的子目录。
    ```bash
    sudo mkdir -p /mnt/GoogleDrive
    ```

4.  **执行挂载命令**
    ```bash
    rclone mount GoogleDrive:/ /mnt/GoogleDrive --allow-other --vfs-cache-mode writes
    ```
> [!WARNING]
> **此命令会阻塞当前终端**
> 该命令默认在前台运行，会持续输出日志并占用当前终端会话。在验证挂载或执行其他命令前，您需要 **新开一个终端窗口**。测试完成后，可以回到此终端按 `Ctrl+C` 来停止挂载。
    
    -   `GoogleDrive:/`: `rclone` 配置的远程名称和要挂载的路径（`/` 代表根目录）。
    -   `/mnt/GoogleDrive`: 本地挂载点目录。
    -   `--allow-other`: 允许其他用户访问挂载的文件，对于 systemd 服务和 Docker 等应用是必需的。
    -   `--vfs-cache-mode writes`: VFS 缓存模式。`writes` 模式可以显著提高读写性能和稳定性，强烈推荐使用。
    -   `--daemon` (可选): 添加此参数可以让 `rclone` 在后台运行，从而不会阻塞当前终端。虽然方便，但对于最终的开机自启，我们强烈推荐使用 `systemd` 进行管理，而不是依赖此参数。

5.  **验证挂载**
    打开一个新的终端窗口，执行 `df -h`。如果看到类似下面的输出，说明挂载成功。
    ```
    Filesystem      Size  Used Avail Use% Mounted on
    GoogleDrive:/   15G   4.9G   11G  33% /mnt/GoogleDrive
    ```

6.  **卸载**
    要卸载挂载盘，只需在运行挂载命令的终端按 `Ctrl+C`，或者在新终端执行：
    ```bash
    fusermount3 -u /mnt/GoogleDrive
    ```
> [!NOTE]
> `rclone` 较新版本默认使用 `fuse3`，因此卸载命令是 `fusermount3`。如果您的系统环境较旧，可能需要使用 `fusermount`。

---

## 第四步：配置 systemd 实现开机自动挂载

使用 `systemd` 服务来管理 `rclone` 挂载是目前最稳定、最可靠的方法。它比 `--daemon` 参数或 `cron` 任务更具优势，因为它能确保在网络准备就绪后才执行挂载，并且能自动重启失败的进程。

1.  **移动 rclone 配置文件（推荐）**
为了让 `systemd` 服务能稳定地找到配置文件，建议将其移动到一个标准的系统位置。
```bash
sudo mkdir -p /etc/rclone
sudo mv ~/.config/rclone/rclone.conf /etc/rclone/rclone.conf
```

2.  **创建 systemd 服务文件**
```bash
sudo vim /etc/systemd/system/rclone.service
```将以下内容复制粘贴到文件中。

```ini
[Unit]
Description=Rclone Mount for Google Drive
AssertPathIsDirectory=/mnt/GoogleDrive
After=network-online.target

[Service]
Type=simple
User=root
Group=root
ExecStart=/usr/bin/rclone mount GoogleDrive:/ /mnt/GoogleDrive \
    --config=/etc/rclone/rclone.conf \
    --allow-other \
    --vfs-cache-mode writes \
    --cache-dir=/tmp/rclone-cache \
    --log-file=/var/log/rclone.log \
    --log-level INFO
ExecStop=/usr/bin/fusermount3 -u /mnt/GoogleDrive
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> [!TIP]
> **参数解释**:
> - `[Unit]`: 定义了服务的依赖关系，`After=network-online.target` 确保在网络连接后才启动。
> - `[Service]`: 定义了服务的核心行为。
>   - `User`, `Group`: 指定以哪个用户和组的身份运行命令。使用 `root` 是最简单直接的方式。
>   - `ExecStart`: 核心挂载命令。这里我们添加了 `--config` 指定配置文件路径，并用 `--log-file` 记录日志，方便排查问题。
>   - `ExecStop`: 定义停止服务时执行的卸载命令。
>   - `Restart=on-failure`: 如果进程意外退出，`systemd` 会自动尝试重启它。
> - `[Install]`: 定义了如何安装这个服务，`WantedBy=multi-user.target` 表示它应该在多用户模式下启用。

---

## 第五步：管理和验证自启动服务

1.  **重新加载 systemd 配置**
    ```bash
    sudo systemctl daemon-reload
    ```

2.  **设置开机自启**
    ```bash
    sudo systemctl enable rclone.service
    ```

3.  **立即启动服务进行测试**
    ```bash
    sudo systemctl start rclone.service
    ```

4.  **检查服务状态**
    ```bash
    sudo systemctl status rclone.service
    ```
    如果看到绿色的 `active (running)`，恭喜你，服务已成功运行！

5.  **检查日志**
    如果服务启动失败，可以通过以下命令查看日志来定位问题：
    ```bash
    # 查看 systemd 的日志
    sudo journalctl -u rclone.service -f
    
    # 或者查看 rclone 自己的日志文件
    tail -f /var/log/rclone.log
    ```

现在，你的 Google Drive 已经成功挂载到 Ubuntu 系统，并且会在每次开机后自动挂载，稳定可靠。

---

## 第六步：解决文件同步问题 (自动与手动)

在使用 `rclone mount` 时，你可能会遇到一个常见问题：在其他设备上（或通过网页）向云端添加或删除了文件后，本地挂载点没有立即显示这些变化。这是因为 `rclone` 为了提高性能和减少 API 调用，会缓存文件和目录的元数据信息（比如文件名、大小、修改时间）。

默认情况下，这个目录缓存 (`--dir-cache-time`) 的有效期是 5 分钟。这意味着 `rclone` 每 5 分钟才会重新从云端检查一次目录结构。对于需要近实时同步的场景，这显然是不够的。

幸运的是，`rclone` 提供了几种强大的机制来解决这个问题。

### 1. 自动同步：使用轮询 (Polling) - 推荐方式

对于 Google Drive 这样支持"变更通知"的后端，最佳方法是启用轮询功能。它允许 `rclone` 定期"询问"云端是否有任何变动，一旦发现变动就立即更新本地缓存，非常高效。

要启用轮询，只需在挂载命令中添加 `--poll-interval` 参数。

-   `--poll-interval <时间>`: 设置轮询间隔。一个合理的值是 `1m` (1分钟)。设置得太短可能会增加少量API调用，但对于 Google Drive 通常不是问题。

**更新 `systemd` 服务文件**:

我们可以将这个参数添加到之前的 `rclone.service` 文件中。同时，既然我们依赖轮询来获取更新，就可以把目录缓存时间 `--dir-cache-time` 设得非常长（例如 `1000h`），以避免因缓存过期而产生不必要的全量刷新。

```bash
sudo vim /etc/systemd/system/rclone.service
```

修改后的 `ExecStart` 如下：

```ini
[Unit]
Description=Rclone Mount for Google Drive
AssertPathIsDirectory=/mnt/GoogleDrive
After=network-online.target

[Service]
Type=simple
User=root
Group=root
ExecStart=/usr/bin/rclone mount GoogleDrive:/ /mnt/GoogleDrive \
    --config=/etc/rclone/rclone.conf \
    --allow-other \
    --vfs-cache-mode writes \
    --cache-dir=/tmp/rclone-cache \
    --poll-interval 1m \
    --dir-cache-time 1000h \
    --log-file=/var/log/rclone.log \
    --log-level INFO
ExecStop=/usr/bin/fusermount3 -u /mnt/GoogleDrive
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> [!NOTE]
> **新参数解释**:
> - `--poll-interval 1m`: 每隔1分钟检查一次 Google Drive 的内容是否有变化。
> - `--dir-cache-time 1000h`: 将目录结构的缓存有效期延长到1000小时。这使得文件列表的刷新主要依赖于更高效率的 `poll-interval` 机制，而不是定期的暴力刷新。

修改并保存文件后，不要忘记重新加载配置并重启服务：

```bash
sudo systemctl daemon-reload
sudo systemctl restart rclone.service
```

### 2. 手动同步：使用远程控制 (Remote Control)

在某些情况下（例如，使用的云存储不支持轮询，或者你刚刚完成一个大操作，想立即看到结果），你可以使用 `rclone` 的远程控制功能来手动触发一次全量刷新。

**步骤 1: 启用 RC (Remote Control)**

首先，需要在挂载命令中添加 `--rc` 参数来开启远程控制接口。

**步骤 2: 修改 `systemd` 文件**

将 `--rc` 添加到 `ExecStart` 命令中：

```ini
ExecStart=/usr/bin/rclone mount GoogleDrive:/ /mnt/GoogleDrive \
    ... # 其他参数
    --rc \
    --log-level INFO
```

然后重载并重启服务。

**步骤 3: 发送刷新命令**

当 `rclone` mount 服务正在运行时，你可以随时在终端执行以下命令来强制刷新整个挂载点的缓存：

```bash
rclone rc vfs/refresh recursive=true
```

-   `rc`: 表示这是一个远程控制命令。
-   `vfs/refresh`: 调用 VFS 缓存的刷新功能。
-   `recursive=true`: 表示递归刷新所有子目录，确保整个挂載都得到更新。

执行后，`rclone` 会立即丢弃现有缓存并从云端重新获取最新的文件列表。这个操作可能需要一点时间，具体取决于你的文件数量。

通过以上两种方法，你就可以灵活地控制 `rclone` 挂载点与云端文件的同步时机，确保数据的一致性。


