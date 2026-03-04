# Incus 宿主机本地路径挂载



# Incus 容器本地路径挂载与用户权限映射完整指南

## 概述

在使用 Incus 容器时，经常需要将宿主机的目录挂载到容器内部，特别是在部署需要持久化存储的应用（如 Immich 相册系统）时。本文将详细介绍如何正确配置 Incus 容器的本地路径挂载，以及如何解决常见的权限映射问题。

## 场景说明

本文以一个实际场景为例：
- **宿主机**：NanoPC-T6，路径 `/srv/nfs/Photos`
- **TrueNAS**：通过 rsync 同步数据到宿主机（使用 uid 3000）
- **Incus 容器**：debain，需要挂载 Photos 目录
- **应用**：Immich 相册系统，需要读写权限

## 基础挂载配置

### 1. 添加磁盘设备

```bash
# Basic mount command
incus config device add debain nfs-mount disk \
  source=/srv/nfs/Photos \
  path=/mnt/nanopc/Photos
```

> [!NOTE]
> 这是最基础的挂载方式，但可能会遇到权限问题，文件显示为 `nobody:nogroup`。

### 2. 验证挂载

```bash
# Check if mount is successful
incus exec debain -- ls -la /mnt/nanopc/Photos
```

## 权限问题分析

### 问题现象

挂载后发现文件所有者显示为 `nobody:nogroup`：

```bash
root@debain:/mnt/nanopc/Photos# ls -la
drwxr-xr-x 5 nobody nogroup 4096 Mar  3 17:04 .
drwx------ 2 nobody nogroup 4096 Mar  3 09:07 immich-app
```

### 根本原因

Incus 默认使用 **UID/GID 映射机制**来实现容器隔离：

```bash
# Check current idmap configuration
incus config show debain | grep -A5 idmap
```

输出示例：
```
volatile.idmap.current: '[{"Isuid":true,"Isgid":false,"Hostid":1000000,"Nsid":0,"Maprange":1000000000}]'
```

这意味着：
- 容器内的 uid 0（root）映射到宿主机的 uid 1000000
- 容器内的 uid 3000 映射到宿主机的 uid 1003000
- 宿主机的 uid 3000 在容器内无法直接识别，显示为 `nobody`

## 解决方案

### 方案一：特权容器（推荐用于生产环境）

这是最简单可靠的方案，适合需要直接访问宿主机文件系统的场景。

```bash
# Stop container first
incus stop debain

# Enable privileged mode
incus config set debain security.privileged true
incus config set debain security.nesting true

# Add mount device
incus config device add debain nfs-mount disk \
  source=/srv/nfs/Photos \
  path=/mnt/nanopc/Photos

# Start container
incus start debain

# Verify permissions
incus exec debain -- ls -la /mnt/nanopc/Photos
```

> [!WARNING]
> 特权容器会降低隔离性，容器内的 root 用户等同于宿主机 root。仅在可信环境中使用。

### 方案二：UID/GID 映射（推荐用于开发环境）

保持容器非特权模式，通过 `raw.idmap` 映射特定 UID。

#### 步骤 1：配置宿主机 subuid/subgid

```bash
# Check current configuration
cat /etc/subuid | grep root
cat /etc/subgid | grep root

# Add uid 3000 mapping (if not exists)
echo "root:3000:1" | sudo tee -a /etc/subuid
echo "root:3000:1" | sudo tee -a /etc/subgid
```

#### 步骤 2：配置容器

```bash
# Stop container
incus stop debain

# Set idmap for uid 3000
incus config set debain raw.idmap "both 3000 3000"

# Add mount device
incus config device add debain nfs-mount disk \
  source=/srv/nfs/Photos \
  path=/mnt/nanopc/Photos

# Start container
incus start debain
```

> [!TIP]
> 如果启动失败并提示 "uid range not allowed"，说明 subuid/subgid 配置未生效，需要重启 Incus 服务：
> ```bash
> sudo systemctl restart incus
> ```

#### 步骤 3：在容器内创建匹配用户

```bash
# Create user with matching uid/gid
incus exec debain -- groupadd -g 3000 immich
incus exec debain -- useradd -u 3000 -g 3000 -m -s /bin/bash immich

# Verify user mapping
incus exec debain -- id immich
# Output: uid=3000(immich) gid=3000(immich) groups=3000(immich)

# Test access
incus exec debain -- su - immich -c "ls -la /mnt/nanopc/Photos"
```

### 方案三：使用 shift 参数（不推荐）

```bash
incus config device add debain nfs-mount disk \
  source=/srv/nfs/Photos \
  path=/mnt/nanopc/Photos \
  shift=true
```

> [!WARNING]
> `shift=true` 会自动映射 UID，但可能导致文件显示为 `nobody`，不适合需要精确权限控制的场景。

## 宿主机权限配置

确保宿主机上的文件权限正确：

```bash
# Set ownership to uid 3000 (matching TrueNAS rsync user)
sudo chown -R 3000:3000 /srv/nfs/Photos

# Set appropriate permissions
sudo chmod -R 755 /srv/nfs/Photos

# For sensitive directories, use 700
sudo chmod 700 /srv/nfs/Photos/immich-app
sudo chmod 700 /srv/nfs/Photos/immich-data

# Verify
ls -la /srv/nfs/Photos
```

## 配置 Immich 使用正确的用户

在 `docker-compose.yml` 中指定运行用户：

```yaml
services:
  immich-server:
    image: ghcr.io/immich-app/immich-server:release
    user: "3000:3000"  # Run as uid 3000
    volumes:
      - /mnt/nanopc/Photos/immich-data:/usr/src/app/upload
    # ... other configurations
```

## 常见问题排查

### 1. 权限被拒绝（Permission Denied）

```bash
# Check file ownership in container
incus exec debain -- ls -la /mnt/nanopc/Photos

# Check if user exists
incus exec debain -- id 3000

# Check idmap configuration
incus config show debain | grep idmap
```

### 2. 文件显示为 nobody

原因：UID 映射未正确配置。

解决：
- 使用特权容器（方案一）
- 或正确配置 raw.idmap（方案二）

### 3. 容器启动失败

```bash
# View detailed error log
incus info --show-log debain

# Common error: "uid range not allowed"
# Solution: Check /etc/subuid and /etc/subgid, restart incus service
```

### 4. 移除错误的挂载配置

```bash
# Remove device
incus config device remove debain nfs-mount

# Unset idmap if needed
incus config unset debain raw.idmap

# Disable privileged mode if needed
incus config set debain security.privileged false
```

## 完整配置示例

### 场景：Immich + TrueNAS rsync + Incus

```bash
# Step 1: Configure host permissions
sudo chown -R 3000:3000 /srv/nfs/Photos
sudo chmod -R 755 /srv/nfs/Photos

# Step 2: Configure Incus container (privileged mode)
incus stop debain
incus config set debain security.privileged true
incus config set debain security.nesting true
incus config device add debain nfs-mount disk \
  source=/srv/nfs/Photos \
  path=/mnt/nanopc/Photos
incus start debain

# Step 3: Create user in container
incus exec debain -- groupadd -g 3000 immich
incus exec debain -- useradd -u 3000 -g 3000 -m immich

# Step 4: Verify access
incus exec debain -- su - immich -c "cd /mnt/nanopc/Photos && ls -la"

# Step 5: Deploy Immich with correct user
# Edit docker-compose.yml to include: user: "3000:3000"
```

## 最佳实践

1. **保持 UID 一致性**：在整个数据流中使用相同的 UID（TrueNAS → 宿主机 → 容器 → 应用）
2. **选择合适的方案**：
   - 生产环境 + 可信容器：使用特权模式
   - 开发环境 + 多租户：使用 UID 映射
3. **最小权限原则**：只映射必要的 UID，避免映射整个范围
4. **文档化配置**：记录 UID 映射关系，便于后续维护
5. **定期备份**：在修改权限配置前备份重要数据

## 参考资料

- [Incus Documentation - Instance Configuration](https://linuxcontainers.org/incus/docs/main/)
- [Linux Containers - UID/GID Mapping](https://linuxcontainers.org/lxc/manpages/man5/lxc.container.conf.5.html)
- [Immich Documentation](https://immich.app/docs)

---

> [!TIP]
> 如果遇到其他权限问题，可以使用 `incus exec debain -- bash` 进入容器，使用 `namei -l /mnt/nanopc/Photos` 命令查看完整的权限链。

