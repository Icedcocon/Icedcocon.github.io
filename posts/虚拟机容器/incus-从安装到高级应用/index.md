# Incus 从安装到高级应用


## 前言

Incus 是一个功能强大的系统容器和虚拟机管理器，作为 LXD 的一个社区驱动的分支，它继承了 LXD 的轻量、高效与安全等优点，并致力于提供一个更加开放和独立的平台。无论你是想创建隔离的开发环境、运行网络应用（如 OpenWrt），还是管理虚拟机，Incus 都能提供一套简洁而强大的工具链。

本文将作为一份全面的入门指南，带你从零开始探索 Incus 的世界，内容涵盖：

-   基础安装与环境配置
-   核心概念与常用命令解析
-   实践案例：部署 OpenWrt 旁路由
-   高级技巧：在 Incus 中构建隔离的 Docker 开发环境

## 一、基础安装与配置

首先，让我们在系统上把 Incus 环境搭建起来。

### 1.1 安装 Incus

方式一
```bash
apt install incus
```

方式二

当无法通过 `apt` 安装时，可以使用 [Zabbly](https://pkgs.zabbly.com/) 提供的仓库，它可以确保你获得最新且稳定的 Incus 版本。

```bash
# 添加 Zabbly GPG 密钥
sudo mkdir -p /etc/apt/keyrings/
curl -fsSL https://pkgs.zabbly.com/key.asc | sudo tee /etc/apt/keyrings/zabbly.asc > /dev/null

# 添加 Zabbly Incus 软件源
sudo sh -c 'cat <<EOF > /etc/apt/sources.list.d/zabbly-incus-stable.sources
Enabled: yes
Types: deb
URIs: https://pkgs.zabbly.com/incus/stable
Suites: $(. /etc/os-release && echo ${VERSION_CODENAME})
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/zabbly.asc
EOF'

# 更新软件包列表并安装 Incus
sudo apt-get update
sudo apt-get install incus -y
```

安装完成后，可以通过 `incus version` 或 `incus -h` 来验证是否安装成功。

### 1.2 初始化 Incus

安装后的第一步是初始化 Incus。执行 `incus admin init` 命令，它会通过一系列问答引导你完成基础配置。

> [!TIP]
> 对于大多数单机用户，直接一路回车选择默认选项即可。
> 但对于网络有较高要求的场景，如 OpenWrt 软路由，请自行建立网桥并在初始化时指定。

以下是一个典型的初始化过程示例：

```bash
incus admin init
Would you like to use clustering? (yes/no) [default=no]: 
Do you want to configure a new storage pool? (yes/no) [default=yes]: 
Name of the new storage pool [default=default]: 
Where should this storage pool store its data? [default=/var/lib/incus/storage-pools/default]: 
Would you like to create a new local network bridge? (yes/no) [default=yes]: 
What should the new bridge be called? [default=incusbr0]: 
What IPv4 address should be used? (CIDR subnet notation, "auto" or "none") [default=auto]: 
What IPv6 address should be used? (CIDR subnet notation, "auto" or "none") [default=auto]: 
Would you like the server to be available over the network? (yes/no) [default=no]: 
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]: no
Would you like a YAML "init" preseed to be printed? (yes/no) [default=no]:
```

> [!NOTE]
> - **storage pool**: 存储池是用于存放实例（容器和虚拟机）数据的地方，默认为 `dir` 类型，直接使用宿主机文件系统。
> - **network bridge**: Incus 会创建一个名为 `incusbr0` 的网桥，并配置 NAT，让实例可以访问外部网络。
> - **stale cached images**: 建议对"是否自动更新旧镜像"选择 `no`，以避免不必要的后台流量和磁盘占用，手动更新通常是更好的选择。

### 1.3 配置国内镜像源

为了加快下载实例镜像的速度，我们可以添加国内的镜像源。

```bash
# 查看当前已有的远程镜像服务器
incus remote list

# 添加国内镜像源 (以北外镜像源为例)
incus remote add mirror https://mirrors.bfsu.edu.cn/lxc-images/ --protocol=simplestreams --public

# 你也可以添加清华大学的镜像源
incus remote add tsinghua-images https://mirrors.tuna.tsinghua.edu.cn/lxc-images/ --protocol=simplestreams --public
```

添加后，你就可以从更快的源拉取镜像了。例如，查看北外镜像源提供的 Debian 镜像：

```bash
incus image list mirror:debian
```

## 二、核心概念与常用命令

掌握了基础配置后，我们来学习如何管理 Incus 实例。

### 2.1 创建实例

Incus 是基于镜像的，你可以从远程镜像源启动容器或虚拟机。

```bash
# 启动一个名为 my-ubuntu 的 Ubuntu 22.04 容器
incus launch images:ubuntu/22.04 my-ubuntu

# 启动一个名为 my-debian-vm 的 Debian 12 虚拟机
# --vm 标志是关键
incus launch images:debian/12 my-debian-vm --vm
```
> [!TIP]
> 第一次启动某个镜像时，Incus 会先下载并缓存它，后续再使用该镜像创建实例会快得多。

### 2.2 管理实例生命周期

管理实例的启停、删除等非常直观。

```bash
# 列出所有实例
incus list

# 启动、停止、重启实例
incus start my-ubuntu
incus stop my-ubuntu
incus restart my-ubuntu

# 获取实例的详细信息
incus info my-ubuntu

# 复制实例
incus copy my-ubuntu my-ubuntu-clone

# 删除实例
incus delete my-debian-vm
# 如果实例正在运行，需要先停止或强制删除
incus delete my-ubuntu --force
```

### 2.3 资源限制

你可以轻松地限制实例可以使用的系统资源。

```bash
# 创建时限制：启动一个只有 1 个 vCPU 和 512MB 内存的容器
incus launch images:ubuntu/22.04 limited-box -c limits.cpu=1 -c limits.memory=512MiB

# 运行时修改：将内存限制调整为 1GB
incus config set limited-box limits.memory=1GiB

# 查看实例的当前配置
incus config show limited-box
```

### 2.4 与实例交互

与实例内部环境进行交互是日常使用中最频繁的操作。

```bash
# 在实例中执行单个命令
incus exec my-ubuntu -- apt update

# 获取实例的交互式 shell
incus exec my-ubuntu -- bash
# exit
```

文件传输同样简单：

```bash
# 从实例中拉取文件到本地
incus file pull my-ubuntu/etc/hosts .

# 将本地文件推送到实例中
echo "1.2.3.4 my-example" >> hosts
incus file push hosts my-ubuntu/etc/hosts
```

### 2.5 快照管理

快照功能是 Incus 的一大亮点，它能让你轻松地保存和恢复实例的状态，非常适合在进行危险操作前做备份。

```bash
# 为 my-ubuntu 实例创建一个名为 pre-update 的快照
incus snapshot create my-ubuntu pre-update

# 查看实例的所有快照
incus snapshot list my-ubuntu

# 假设你在实例里搞砸了一些东西...
# incus exec my-ubuntu -- rm -rf /bin

# 从快照恢复实例
incus snapshot restore my-ubuntu pre-update

# 确认一切恢复正常后，删除快照
incus snapshot delete my-ubuntu pre-update
```

## 三、实践案例：在 Incus 中部署 OpenWrt 旁路由

这个案例将向你展示如何利用 Incus 创建一个 OpenWrt 容器，并将其配置为局域网的旁路由，实现更灵活的网络管理。

### 3.1 网络准备：创建独立网桥

为了让 OpenWrt 容器能与你的物理网络设备处于同一网段，我们需要让它直接桥接到物理网卡上，而不是使用 Incus 默认的 NAT 网络。

首先，需要修改宿主机的网络配置，创建一个名为 `br0` 的网桥，并将物理网卡（例如 `eth0`）桥接上去。

> [!WARNING]
> 修改网络配置是高危操作，尤其是在远程服务器上。请确保你了解每一项配置的含义，或在物理接触到设备的情况下进行。配置错误可能导致服务器网络中断。

以 `netplan` (Ubuntu 常用) 为例，修改 `/etc/netplan/01-netcfg.yaml`：

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      optional: true
  bridges:
    br0:
      interfaces: [eth0]
      addresses: [192.168.2.2/24]  # 宿主机的 IP
      routes:
        - to: default
          via: 192.168.2.1          # 你的主路由 IP
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

配置完成后，使用 `sudo netplan apply` 应用更改。

接下来，在 `incus admin init` 时，当问及网络配置，选择使用现有的 `br0` 网桥。如果已经初始化过，可以后续再添加 profile 或直接在启动时指定。

### 3.2 创建并配置 OpenWrt 容器

现在，我们可以启动一个 OpenWrt 容器了。

```bash
# 从镜像源启动一个 OpenWrt 容器
# security.*=true 给予容器更高的权限，这对于需要管理网络的 OpenWrt 是必要的
incus launch images:openwrt/23.05 openwrt -c security.privileged=true -c security.nesting=true
```
如果你有第三方的 OpenWrt 固件（例如 `immortalwrt.rootfs.tar`），可以先停止容器，然后用 `tar` 命令解压覆盖掉容器的根文件系统。
```bash
incus stop openwrt
# 路径请根据你的存储池位置调整
tar xvf immortalwrt.rootfs.tar -C /var/lib/incus/storage-pools/default/containers/openwrt/rootfs/
incus start openwrt
```


### 3.3 配置容器网络

进入 OpenWrt 容器，修改其网络配置，使其与你的局域网匹配。

```bash
# 进入容器
incus exec openwrt -- bash

# 修改 network 配置文件
vi /etc/config/network
```

修改 `lan` 口的配置如下：
```yaml
config interface 'lan'
        option device 'eth0'
        option proto 'static'
        option ipaddr '192.168.2.10'      # 分配给 OpenWrt 的静态 IP
        option netmask '255.255.255.0'
        option gateway '192.168.2.1'        # 指向你的主路由
        option dns '8.8.8.8 114.114.114.114' # DNS 服务器
```
修改后，执行 `/etc/init.d/network restart` 重启网络服务。现在，你应该可以通过 `192.168.2.10` 访问 OpenWrt 的管理界面了。

> [!NOTE]
> 在 OpenWrt 中，你可能还需要关闭其自身的防火墙，因为它作为旁路由，路由和防火墙功能由主路由负责。
> `/etc/init.d/firewall stop && /etc/init.d/firewall disable`
> 但如果使用 Fake-IP 等特定功能，请不要关闭防火墙。

## 四、高级技巧：在 Incus 中搭建隔离的开发环境

Incus 是创建隔离开发环境的绝佳工具。例如，我们可以在一个 Incus 容器中安装和运行 Docker，这样就不会污染我们的宿主操作系统。

### 4.1 创建开发容器

与创建 OpenWrt 容器类似，我们需要启用特权和嵌套模式，以便在容器内部署 Docker。

```bash
incus launch images:debian/12 dev-env -c security.privileged=true -c security.nesting=true
```

### 4.2 在容器中安装 Docker

进入容器，然后使用 Docker 官方提供的一键安装脚本来安装 Docker Engine。

```bash
# 进入容器
incus exec dev-env -- bash

# 下载并执行安装脚本
curl -fsSL https://get.docker.com -o get-docker.sh

# 使用国内镜像源加速安装
sh get-docker.sh --mirror Aliyun
```

安装完成后，你就在这个 `dev-env` 容器中有了一个功能齐全的 Docker 环境。你可以在这里构建镜像、运行容器，而所有的操作都与你的宿主机系统完全隔离。

## 总结

从简单的容器管理到复杂的网络应用部署，再到创建隔离的开发环境，Incus 展示了其作为现代化虚拟化平台的强大能力和灵活性。希望这篇指南能帮助你顺利入门，并激发你探索更多 Incus 高级用法的兴趣。

## 参考资料
- [Incus official documentation](https://linuxcontainers.org/incus/docs/main/)
- [Zabbly Incus Packages](https://pkgs.zabbly.com/) 
