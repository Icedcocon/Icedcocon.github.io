# TrueNAS SCALE 利用系统盘为 Incus 创建独立存储


## 概述

想在 TrueNAS SCALE 上运行 Incus，但不想为其分配专用硬盘？本文介绍一个高级技巧：利用启动盘 (`boot-pool`) 的闲置空间，通过命令行创建**基于文件的虚拟存储池 (File-backed Vdev)**，并将其挂载给 Incus 应用，实现空间的高效利用。

> [!WARNING]
> **适用场景与限制**
> 这是一个**纯命令行 (CLI-Only)** 的高级技巧，所创建的存储池**无法被 TrueNAS GUI 识别或管理**。
>
> 核心优势在于：尽管存储池对 GUI "隐形"，但其数据集 (Dataset) 可作为**主机路径卷 (Host Path Volume)** 成功挂载到 Incus 等应用中。此方法非常适合测试环境、非关键应用或纯命令行管理场景。

## 操作步骤

以下步骤将指导你在 `boot-pool` 上创建一个 100GB 的虚拟存储池，并分配给 Incus。

### 登录 TrueNAS Shell

通过 Web UI 的 **System Settings -> Shell** 或 SSH 客户端登录。

### 创建虚拟磁盘文件

使用 `truncate` 命令在启动池中创建一个稀疏文件 (Sparse File)，它将作为新存储池的"虚拟硬盘"。

例如，在 `/mnt/boot-pool/` 目录下创建一个 100GB 的文件：
```shell
# Create a 100GB sparse file for the Incus pool.
truncate -s 100G /mnt/boot-pool/incus-vdisk.img
```
> [!NOTE]
> `boot-pool` 的挂载路径通常是 `/mnt/boot-pool`。稀疏文件初始占用空间极小，会随数据写入而增长。

### 创建 ZFS 存储池

使用 `zpool create` 命令并指定文件路径，建立名为 `incus_pool` 的新池。

```shell
# Create a pool named 'incus_pool' with one file-backed vdev.
# -O canmount=off is crucial to prevent mount errors on the parent filesystem.
zpool create -O canmount=off incus_pool /mnt/boot-pool/incus-vdisk.img
```
`zpool` 会提示你正在使用文件，这是预期行为，请确认。

### 导入存储池并创建数据集

为确保系统正确挂载，需先导出存储池，再使用指定目录重新导入，然后创建 Incus 专用数据集。

```shell
# 1. Export the pool to make it available for import.
zpool export incus_pool

# 2. Re-import it, specifying the search path with the -d flag.
zpool import -d /mnt/boot-pool incus_pool

# 3. (Optional) Create a dataset for Incus data.
zfs create incus_pool/data
```
完成后，通过 `ls /mnt/incus_pool/data` 检查数据集是否已成功挂载。

### 在TrueNAS中查看数据集（DataSet）

>[!TIP]
>TrueNAS 的 `Storage` 存储池页面无法管理 `incus_pool` ，但在 `Datasets` 数据集页面却可以管理并使用该存储池。

虽然在 TrueNAS 存储池（Pool）页面无法查看 `incus_pool` ，但是在数据集页面却可以看到该存储池，并可以在 `incus_pool` 存储池中创建新的数据集。
同样的原理，在后台通过命令行创建的数据集，以及在 instances 页面初始化 incus 时创建的数据集不会在 `Datasets` 页面展示。

### Incus 初始化注意事项

> [!WARNING] 
> 不推荐通过shell在命令行执行 `incus admin init` 初始化，避免影响后续 TrueNAS 升级稳定性。

正确做法是：在 TrueNAS UI 的 Incus 应用设置中，直接从存储 (Storage) 下拉菜单里选择创建好的池。TrueNAS 会处理初始化。

## 常见问题排查

#### 问题：`zpool create` 失败并报错 `is part of active pool`

**日志示例**:
```
invalid vdev specification
use '-f' to override the following errors:
/mnt/boot-pool/incus-vdisk.img is part of exported pool 'incus_pool'
```
**原因**:
命令中断 (例如忘记添加 `-O canmount=off`) 导致池创建不完整。ZFS 仍会将该文件标记为"已使用"。

**解决方案**:
确认要覆盖残留配置后，在 `zpool create` 命令中加入 `-f` (force) 标志，强制销毁并重建池。
```shell
# Force create the pool, overwriting any lingering configuration on the file.
zpool create -f -O canmount=off incus_pool /mnt/boot-pool/incus-vdisk.img
# zpool create -f -O mountpoint=/mnt/virtual_pool -O canmount=off virtual_pool /mnt/boot-pool/vdisk1.img
```
完成后，继续执行后续的 `export` 和 `import` 步骤。

#### 解析：`mountpoint` 与 `canmount=off` 的作用

在示例 `zpool create -f -O mountpoint=/mnt/virtual_pool -O canmount=off ...` 中，`mountpoint` 和 `canmount` 的组合可能令人困惑。

-   **`-O mountpoint=/mnt/virtual_pool`**：为池的根文件系统设置 `mountpoint` 属性，定义了"如果允许挂载，应挂载到何处"。

-   **`-O canmount=off`**：此属性**禁止** ZFS 自动挂载该根文件系统。`mountpoint` 定义了**位置**，`canmount` 决定**权限**。`canmount=off` 会阻止挂载，因此 `mountpoint` 设置看起来"未生效"。

**为什么这么做？**

这是一种 ZFS 最佳实践，目的在于：
1.  **保持根的纯净**：将池的根目录作为组织数据集的"容器"，避免直接在根上读写数据。
2.  **设定挂载基准**：根文件系统的 `mountpoint` 成为所有子数据集挂载点的**父路径**。

例如，创建新数据集 `zfs create virtual_pool/data` 后，其挂载点会自动继承并设为 `/mnt/virtual_pool/data`，并因默认 `canmount=on` 而成功挂载。此法使存储结构更清晰。

## 高级操作：手动管理容器文件系统

在数据恢复或离线修改配置等特殊场景下，你可能需要直接操作 Incus 容器的文件系统。Incus 将其存储于独立的 ZFS 数据集，并设置挂载点为 `legacy`，导致在宿主机上不可见。

以下步骤指导你如何在容器停止时，安全地手动挂载文件系统，并在操作完成后恢复原状。

> [!TIP]
> 本章节面向 incus 容器/虚拟机操作系统定制化场景，由于 incus 官方容器或虚拟机仅支持常见发行版，部分小众的版本如 ImmotalWrt 需要自行构建，因此需要对 incus 容器/虚拟机的文件系统进行挂载和调整。

### 查看容器的 ZFS 数据集

首先，找到容器对应的 ZFS 数据集。

```shell
# 递归列出 virtual_pool 下的所有数据集
zfs list -r virtual_pool
# 查看所有dataset 
# zfs list 
```

你会看到类似输出：
```
NAME                                           USED  AVAIL  REFER  MOUNTPOINT
virtual_pool                                   367M  96.0G    24K  /mnt/virtual_pool
...
virtual_pool/.ix-virt/containers/openwrt       55.5K  96.0G  7.49M  legacy
...
```
这里的关键信息是：
- **`NAME`**: `virtual_pool/.ix-virt/containers/openwrt` 就是 `openwrt` 容器的文件系统数据集。
- **`REFER`**: `7.49M` 是这个数据集自身占用的空间，也就是容器文件系统的实际大小。
- **`MOUNTPOINT`**: `legacy` 确认了该数据集的挂载由 Incus 控制，而非系统自动管理。

### 手动挂载、操作与恢复

> [!WARNING]
> **请务必在容器停止的状态下执行以下操作！**

#### 停止容器并创建临时挂载点

```shell
# 确保目标容器已停止
incus stop openwrt

# 创建一个临时目录用于挂载
mkdir -p /mnt/temp_openwrt_root
```

#### 临时挂载文件系统
我们将临时覆盖 `legacy` 设置，以手动挂载文件系统。

```shell
# 1. 临时设置新的挂载点
zfs set mountpoint=/mnt/temp_openwrt_root virtual_pool/.ix-virt/containers/openwrt

# 2. 挂载数据集
zfs mount virtual_pool/.ix-virt/containers/openwrt
```
此时，你可以通过 `ls -l /mnt/temp_openwrt_root` 查看和操作容器的完整文件系统。

#### 操作完成后的清理
完成文件操作后，**必须**将挂载控制权交还给 Incus。

```shell
# 1. 卸载文件系统
zfs unmount virtual_pool/.ix-virt/containers/openwrt

# 2. 将挂载点属性恢复为 legacy
zfs set mountpoint=legacy virtual_pool/.ix-virt/containers/openwrt

# 3. 删除临时目录
rmdir /mnt/temp_openwrt_root
```
完成后，即可通过 `incus start openwrt` 安全地重启容器。这个过程确保了 Incus 能继续正常管理容器的生命周期。

