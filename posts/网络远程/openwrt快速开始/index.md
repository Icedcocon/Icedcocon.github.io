# OpenWrt 容器化部署指南


本文将详细介绍如何在 Debian 12 系统上通过容器化方式部署 OpenWrt 或其衍生版 IStoreOS。我们将探讨两种主流方案：使用 Incus (LXC) 和使用 Docker。

## 准备工作：获取 OpenWrt Rootfs

无论选择哪种部署方案，我们首先需要准备 OpenWrt 的根文件系统 (rootfs)。

### 1. 安装依赖

我们需要 `qemu-utils` 来处理镜像文件。

```bash
# 镜像处理
apt install qemu-utils kmod
# 对于ARM架构，可能需要安装qemu-system-arm
apt -t bookworm-backports install qemu-system-arm
```

### 2. 下载并提取 Rootfs

此步骤将从官方镜像中提取出适用于容器的 rootfs 文件。

> [!TIP]
> IStoreOS for r5s 镜像地址: `https://fw.koolcenter.com/iStoreOS/r5s/`

```bash
# 1. 创建工作目录
mkdir -p /root/immortalwrt
cd /root/immortalwrt

# 2. 下载 OpenWrt/IStoreOS 镜像
# 示例为 ImmortalWRT，您可以替换为其他版本的链接
wget https://downloads.immortalwrt.org/releases/24.10.1/targets/rockchip/armv8/immortalwrt-24.10.1-rockchip-armv8-friendlyarm_nanopc-t6-squashfs-sysupgrade.img.gz
wget https://downloads.immortalwrt.org/releases/24.10.1/targets/x86/64/immortalwrt-24.10.1-x86-64-generic-squashfs-combined.img.gz

# 3. 解压镜像
mv immortalwrt*.img.gz immortalwrt-24.10.1.img.gz
gunzip immortalwrt-24.10.1.img.gz

# 4. 挂载镜像分区并提取文件
# 加载 nbd 内核模块
modprobe nbd
# 将镜像映射为块设备
qemu-nbd -c /dev/nbd0 -f raw immortalwrt-24.10.1.img

# 查看分区，通常 rootfs 在第二个分区 (nbd0p2)
lsblk -f /dev/nbd0

# 创建挂载点并挂载
mkdir /mnt/immortalwrt
mount /dev/nbd0p2 /mnt/immortalwrt/

# 5. 打包为 rootfs.tar
cd /mnt/immortalwrt/
tar -cf /root/immortalwrt/immortalwrt.rootfs.tar .
cd /root

# 6. 清理
umount /mnt/immortalwrt
qemu-nbd -d /dev/nbd0
rmdir /mnt/immortalwrt
```

## 方案一：Incus 部署

Incus 是 LXD 的一个社区分支，提供了强大的容器和虚拟机管理功能。

> [!NOTE]
> 在开始之前，建议观看此视频教程以获得更直观的了解：[OpenClash零基础入门教程](https://www.youtube.com/watch?v=7wiu1YA8Pbc&list=PLSbqX2QvapHk7VYlbyHUIOonIl7q1n410)

### 1. 准备工作

#### 1.1 安装 Incus

```bash
# LXC
apt install lxc lxc-templates bridge-utils uidmap
# netplan
apt install netplan.io
# incus
apt install incus
```

#### 1.2 配置宿主机网桥

为了让容器能与局域网通信，我们需要配置一个网桥。

> [!WARNING]
> 网桥 `br0` 的 IP 地址不应与物理网卡 `eth0` 的 IP 相同。

编辑 Netplan 配置文件 `/etc/netplan/01-netcfg.yaml`：

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      optional: true
      addresses: []
  bridges:
    br0:
      interfaces: [eth0]
      addresses: [192.168.2.2/24] # 宿主机新管理IP
      routes:
        - to: default
          via: 192.168.2.1 # 你的主路由IP
          metric: 100
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```
应用配置: `netplan apply`

#### 1.3 初始化 Incus

运行初始化向导，并将默认网络设置为我们创建的 `br0` 网桥。

```bash
root@NanoPC-T6:/etc/netplan# incus admin init
Would you like to use clustering? (yes/no) [default=no]:
Do you want to configure a new storage pool? (yes/no) [default=yes]:
Name of the new storage pool [default=default]:
Where should this storage pool store its data? [default=/var/lib/incus/storage-pools/default]:
Would you like to create a new local network bridge? (yes/no) [default=yes]: no
Would you like to use an existing bridge or host interface? (yes/no) [default=no]: yes
Name of the existing bridge or host interface: br0
Would you like the server to be available over the network? (yes/no) [default=no]: no
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]:
Would you like a YAML "init" preseed to be printed? (yes/no) [default=no]:
```
根据提示进行选择：
- **Clustering**: no
- **New storage pool**: yes (使用默认即可)
- **New local network bridge**: no
- **Use an existing bridge or host interface**: yes
- **Name of the existing bridge**: br0
- 其他选项可根据需求选择或保持默认。

### 2. 创建并配置容器

#### 2.1 创建容器

我们先从官方镜像源创建一个临时的 OpenWrt 容器，主要是为了生成容器的目录结构。

```bash
# 查看源
incus remote list
# 添加清华大学镜像源
incus remote add tsinghua-images https://mirrors.tuna.tsinghua.edu.cn/lxc-images/ --protocol=simplestreams --public
# 从镜像源创建容器
incus launch tsinghua-images:openwrt/24.10 openwrt -c security.privileged=true -c security.nesting=true
# 停止容器，准备替换文件系统
incus stop openwrt
```

#### 2.2 替换 Rootfs

```bash
# 路径默认为 /var/lib/incus/storage-pools/default/containers/
# 解压我们之前准备的 rootfs.tar 到容器的 rootfs 目录
tar xf /root/immortalwrt/immortalwrt.rootfs.tar -C /var/lib/incus/storage-pools/default/containers/openwrt/rootfs/

# 启动容器
incus start openwrt
```

### 3. 容器内部配置

#### 3.1 进入容器

```bash
incus exec openwrt -- bash
```

#### 3.2 配置网络

编辑 OpenWrt 的网络配置 `/etc/config/network`，设置静态 IP。

```ini
config interface 'lan'
        option device 'eth0'
        option proto 'static'
        option ipaddr '192.168.2.10'    # OpenWrt 的 IP
        option netmask '255.255.255.0'
        option gateway '192.168.2.1'      # 主路由 IP
        option dns '8.8.8.8 1.1.1.1'

config interface 'wan6'
        option device '@lan'          # 使用和 lan 接口相同的物理设备 (eth0)
        option proto 'dhcpv6'         # 设置为 DHCPv6 客户端模式
        option reqaddress 'try'       # 尝试向DHCPv6服务器请求一个地址
        option reqprefix 'auto'       # 自动向上游路由器请求IPv6前缀
```

> [!TIP]
> 只需保留 `lan` 口的基础配置即可，可删除或注释掉文件中其他无关的 `interface` 和 `globals` 配置。修改后执行 `/etc/init.d/network restart` 重启网络服务。如果网络不通，尝试重启整个容器 `reboot`。

#### 3.3 开启防火墙

> [!WARNING]
> 将 OpenWrt 作为旁路由时，通常建议关闭防火墙以避免不必要的网络问题。但如果您计划使用 OpenClash 的 Fake-IP 模式，请 **不要** 关闭容器的防火墙，否则会导致 OpenClash 无法正常工作。

```bash
/etc/init.d/firewall start
/etc/init.d/firewall enable
```

#### 3.4 安装常用插件与配置

##### 1. 更新软件源 (推荐)
为了加速插件下载，建议将 `opkg` 源更换为国内镜像。

> [!TIP]
> 请注意镜像源需要与您的 OpenWrt/ImmortalWRT 版本匹配。以下命令以 ImmortalWRT 为例，将其官方源替换为阿里镜像源。如果镜像地址失效，请查找其他可用镜像。

```bash
# 您可以手动编辑 /etc/opkg/distfeeds.conf 文件，或使用以下 sed 命令快速替换
sed -i 's|https://downloads.immortalwrt.org|https://mirrors.aliyun.com/openwrt|g' /etc/opkg/distfeeds.conf

# 更新软件包列表
opkg update
```

##### 2. 安装基础工具
```bash
# 安装 SSH 服务端、SFTP 服务
opkg install openssh-server openssh-sftp-server 
# 安装 网页终端 (ttyd) 和文件浏览器（可选）
opkg install luci-i18n-ttyd-zh-cn luci-i18n-filebrowser-go-zh-cn
```

##### 3. 安装 Argon 主题 (可选)
Argon 是一个非常流行和美观的 LuCI 界面主题。
```bash
# 安装主题依赖
opkg install luci-compat luci-lib-ipkg

# 从 GitHub 下载并安装主题（请检查并使用最新的版本链接）
wget --no-check-certificate https://github.com/jerrykuku/luci-theme-argon/releases/download/v2.3.2/luci-theme-argon_2.3.2-r20250207_all.ipk
opkg install luci-theme-argon*.ipk

# 清理安装包
rm luci-theme-argon*.ipk
```

##### 4. 安装 iStore 应用商店 (可选)
iStore 提供了图形化的插件中心，方便查找和安装插件。
```bash
wget -qO imm.sh https://cafe.cpolar.top/wkdaily/zero3/raw/branch/main/zero3/imm.sh && chmod +x imm.sh && ./imm.sh
```

##### 5. 为 OpenClash 等插件做准备 (可选)
> [!NOTE]
> 此步骤仅适用于计划安装和使用 OpenClash 等高级代理插件的用户。
>
> 1.  **确保防火墙已开启**：如 `3.3` 节所述，OpenClash 的 Fake-IP 模式需要系统防火墙正常工作。
> 2.  **安装 `dnsmasq-full`**：`openclash` 等插件依赖 `dnsmasq` 的完整功能。

```bash
# 移除基础版 dnsmasq 并安装功能更全的 dnsmasq-full
opkg remove dnsmasq
opkg install dnsmasq-full
```

#### 3.5 `dnsmasq` 启动失败排错

在某些情况下，尤其是在容器环境中，`dnsmasq` 可能会因为无法访问随机数生成器而启动失败。

**故障现象**

执行 `/etc/init.d/dnsmasq status` 显示 `not running`。
查看日志 `logread | grep dnsmasq` 会发现关键错误：
```log
daemon.crit dnsmasq[1]: failed to seed the random number generator: No such file or directory
daemon.crit dnsmasq[1]: FAILED to start up
```

**原因分析**

新版 OpenWrt 为了安全，会将 `dnsmasq` 进程置于一个沙箱 (`ujail`) 环境中运行。默认情况下，沙箱会阻止 `dnsmasq` 访问其运行所需的系统熵源（随机数文件），导致启动失败。

**解决方案**

我们需要明确授权 `dnsmasq` 的沙箱环境访问 `/dev/urandom`。执行以下命令：

```bash
# 允许 dnsmasq 的沙箱环境访问 /dev/urandom
uci set dhcp.@dnsmasq[0].random_seed='/dev/urandom'
uci add_list dhcp.@dnsmasq[0].addnmount='/dev/urandom'
uci commit dhcp

# 重启服务使配置生效
/etc/init.d/dnsmasq restart
```

> [!TIP]
> 如果重启服务后问题依旧，建议执行 `reboot` 重启整个 OpenWrt 容器，以确保在完全干净的环境下应用新配置。

## 方案二：Docker 部署

Docker 是另一种流行的容器化方案。

> [!WARNING]
> Docker 的 macvlan 网络模式可能与 Incus 的桥接网络存在冲突。请避免在同一宿主机上混合使用这两种方案的网络配置，以免造成网络问题。

### 1. 准备工作

#### 1.1 安装 Docker

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh --mirror Aliyun
```

#### 1.2 构建 Docker 镜像

使用我们之前准备的 `rootfs.tar` 来构建一个 Docker 镜像。

```bash
# 切换到工作目录
cd /root/immortalwrt/

# 创建 Dockerfile
cat <<EOF >"Dockerfile"
FROM scratch
ADD immortalwrt.rootfs.tar /
EOF

# 构建镜像
docker build -t immortalwrt:latest .
```

### 2. 配置宿主机网络

#### 2.1 创建 macvlan 网络

macvlan 可以让容器直接连接到物理网络，拥有独立的 IP 地址。

```bash
docker network create -d macvlan \
  --subnet=192.168.2.0/24 \
  --gateway=192.168.2.1 \
  -o parent=eth0 \
  macnet
```
- `--subnet` 和 `--gateway` 需要根据你的局域网配置修改。
- `parent=eth0` 指定了所依赖的物理网卡。

#### 2.2 (可选) 创建 macvlan 子接口

创建 macvlan 网络后，宿主机默认无法与容器通信。创建一个虚拟子接口可以解决这个问题。

```bash
ip link add macvlan-shim link eth0 type macvlan mode bridge
ip addr add 192.168.2.3/24 dev macvlan-shim # 分配一个未被占用的IP
ip link set macvlan-shim up
ip route add 192.168.2.0/24 dev macvlan-shim
```

### 3. 启动并配置容器

#### 3.1 启动容器

```bash
docker run --name immortalwrt \
  --restart always \
  -d \
  --network macnet \
  --ip 192.168.2.10 \
  --privileged \
  immortalwrt:latest \
  /sbin/init
```
- `--ip 192.168.2.10` 为容器指定一个静态 IP。

#### 3.2 容器内部配置

与 Incus 方案类似，我们需要进入容器修改网络配置。

```bash
# 进入容器
docker exec -it immortalwrt bash

# 编辑 /etc/config/network
# ... (配置内容同 Incus 方案) ...

# 重启网络
/etc/init.d/network restart
```

### 4. 安装依赖与持久化

在容器内安装完所需插件后，建议将更改保存为一个新的 Docker 镜像。

```bash
# (在容器内安装插件...)
opkg update && opkg install ...

# (在宿主机上) 保存镜像
docker commit immortalwrt immortalwrt:configured
```

## 常见问题排查

- **网络不通**:
  - 检查宿主机和 OpenWrt 容器的 IP、网关、DNS 配置是否正确。
  - 在宿主机上执行 `iptables -P FORWARD ACCEPT` 尝试放行所有转发流量。
- **查看日志**: 在 OpenWrt 容器内，使用 `logread` 命令可以查看系统日志，帮助定位问题。

## 参考资料

- [悟空的日常](https://wkdaily.cpolar.top/15)
- [Linux 虚拟网卡技术：Macvlan &#183; 云原生实验室](https://icloudnative.io/posts/netwnetwork-virtualization-macvlan/)
- [LXC容器：概念介绍及简单上手操作指导](https://www.cnblogs.com/onestarlearner/p/17605214.html)
- [incus主体安装 | 一键虚拟化项目](https://www.spiritlhl.net/guide/incus/incus_install.html)

