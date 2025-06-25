# TrueNAS SCALE 利用系统盘为 Incus 创建独立存储


## 概述

当你想在 TrueNAS SCALE 上运行 Incus 时，通常会为其配置一个专属的存储池以获得最佳性能。但如果你不想为此单独分配一块物理硬盘，而是希望利用TrueNAS启动盘（`boot-pool`）上的闲置空间，该怎么办？

本文将介绍一个高级技巧：通过命令行在启动池上创建**基于文件的虚拟存储池（File-backed Vdev）**，并将其挂载给 Incus 应用，从而实现空间的高效利用。

> [!WARNING]
> **适用场景与限制**
> 这是一个**纯命令行（CLI-Only）** 的高级技巧。通过此方法创建的存储池**无法被 TrueNAS GUI 存储仪表盘识别或管理**。
>
> 它的核心优势是，虽然存储池本身对 GUI "隐形"，但从该池创建的数据集（Dataset）可以作为 **主机路径卷（Host Path Volume）** 成功挂载到 Incus 等应用中。这使其成为测试环境、非关键应用或命令行工具管理的理想选择。

## 操作步骤

以下步骤将指导你在 `boot-pool` 上创建一个 100GB 的虚拟存储池，并将其分配给 Incus 使用。

### 1. 登录 TrueNAS Shell

通过 Web UI 的 **System Settings -> Shell** 或 SSH 客户端登录。

### 2. 创建虚拟磁盘文件

使用 `truncate` 命令在启动池中创建一个稀疏文件（Sparse File）。它将作为我们新存储池的"虚拟硬盘"。

例如，在 `/mnt/boot-pool/` 目录下创建一个 100GB 的文件：
```shell
# Create a 100GB sparse file for the Incus pool.
truncate -s 100G /mnt/boot-pool/incus-vdisk.img
```
> [!NOTE]
> `boot-pool` 的挂载路径通常是 `/mnt/boot-pool`。稀疏文件初始占用空间极小，会随数据写入而增长。

### 3. 创建 ZFS 存储池

使用 `zpool create` 命令，并指定文件路径来建立新池。我们将池命名为 `incus_pool`。

```shell
# Create a pool named 'incus_pool' with one file-backed vdev.
# -O canmount=off is crucial to prevent mount errors on the parent filesystem.
zpool create -O canmount=off incus_pool /mnt/boot-pool/incus-vdisk.img
```
`zpool` 会提示你正在使用文件，这是预期行为，确认即可。

### 4. 导入存储池并创建数据集

为了让系统正确挂载和使用该池，你需要先将其导出，然后使用指定目录重新导入。导入后，我们立即创建一个专用于 Incus 的数据集。

```shell
# 1. Export the pool to make it available for import.
zpool export incus_pool

# 2. Re-import it, specifying the search path with the -d flag.
zpool import -d /mnt/boot-pool incus_pool

# 3. (非必须) Create a dataset for Incus data.
zfs create incus_pool/data
```
完成后，你可以通过 `ls /mnt/incus_pool/data` 检查数据集是否已成功挂载。

### 5. （可选）挂载存储到 Incus 应用

这是将我们的"隐形"存储池与 Incus 连接起来的关键一步。

1.  在 TrueNAS UI 中，导航至 **Apps**，找到你的 Incus 应用并点击 **Edit**。
2.  向下滚动到 **Storage** 部分。
3.  点击 **Add** 并选择 **Host Path Volume**。
4.  配置挂载点：
    *   **Host Path**: 输入我们刚刚创建的数据集挂载路径 `/mnt/incus_pool/data`。
    *   **Mount Path in Pod**: 输入 Incus 默认的数据目录 `/var/lib/incus/`。
5.  保存更改。应用将会重启并应用新的存储配置。

### 6. 关于 Incus 初始化

与标准 Incus 安装不同，我们**不需要**进入容器执行 `incus admin init`。

当你在 TrueNAS UI 中的 instance 菜单中，进行 Global Settings， 选择 Storage 的 Pool 下拉菜单，可以选择。

这样做的好处是：
- **避免网络冲突**：自动初始化过程不会尝试创建新的网桥（如 `incusbr0`），从而避免了与 TrueNAS 系统网络的潜在冲突。
- **简化部署**：所有配置均在 UI 中完成，流程更统一。

## GUI 集成困境与技术解析

**为什么这个方案能行得通？**

根本原因在于 TrueNAS 不同层级之间的关注点不同：

-   **管理层（`midclt`）的局限**：TrueNAS 的核心管理服务 `midclt` 和 GUI 被设计为管理**物理块设备**。它们在扫描存储池时，不会检查文件路径，因此我们的 `incus_pool` 对它们是"隐形"的。实践证明，任何尝试使用 `midclt call pool.import_pool` 导入文件池的操作都会静默失败，无法令其在 GUI 中可见。
-   **容器层的灵活性**：而 Incus 应用所在的容器（Kubernetes）层不关心存储的来源。它只需要一个有效的**主机路径（Host Path）**。由于我们的 `incus_pool/data` 数据集被 ZFS 成功挂载到了 `/mnt/incus_pool/data`，这个路径是真实存在的。因此，容器可以毫无问题地将其挂载到内部。

最终，我们巧妙地利用了这个架构差异，让一个对上层管理系统"隐形"的存储池，为底层应用提供了切实可用的存储空间。

## 常见问题排查

#### 问题：`zpool create` 失败并报错 `is part of active pool`

**日志示例**:
```
invalid vdev specification
use '-f' to override the following errors:
/mnt/boot-pool/incus-vdisk.img is part of exported pool 'incus_pool'
```
**原因**:
首次创建池时，命令可能因某些原因（如忘记添加 `-O canmount=off`）而中断，但池已被不完整地创建。ZFS 会将该文件标记为"已使用"。

**解决方案**:
如果你确认要覆盖之前的残留配置，可以直接在 `zpool create` 命令中使用 `-f` (force) 标志。这会一步完成销毁和重建。
```shell
# Force create the pool, overwriting any lingering configuration on the file.
zpool create -f -O canmount=off incus_pool /mnt/boot-pool/incus-vdisk.img
# zpool create -f -O mountpoint=/mnt/virtual_pool -O canmount=off virtual_pool /mnt/boot-pool/vdisk1.img
```
完成后，继续执行后续的 `export` 和 `import` 步骤。

#### 解析：`mountpoint` 与 `canmount=off` 的作用

在示例命令 `zpool create -f -O mountpoint=/mnt/virtual_pool -O canmount=off ...` 中，你可能会对 `mountpoint` 和 `canmount` 的组合感到困惑。

-   **`-O mountpoint=/mnt/virtual_pool`**: 这个参数为存储池的**根文件系统**设置 `mountpoint`（挂载点）属性。它告诉 ZFS："如果这个文件系统被允许挂载，它应该被挂载到 `/mnt/virtual_pool` 目录。"

-   **为什么它看起来"没生效"？**: 原因是紧随其后的 **`-O canmount=off`**。这个属性**禁止** ZFS 自动挂载该文件系统。`mountpoint` 定义了**挂载到哪里**，而 `canmount` 决定了**是否允许挂载**。当 `canmount` 为 `off` 时，挂载操作被完全阻止。

**为什么这么做？**

这是一种 ZFS 的最佳实践，其目的在于：
1.  **将根作为纯容器**：不直接在存储池的根目录读写数据，而是将其作为一个干净的、用于组织子文件系统（Dataset）的容器。
2.  **为子数据集提供挂载基准**：根文件系统的 `mountpoint` 属性会成为其下所有子数据集挂载路径的**父路径**。

例如，设置完成后，当你创建一个新的数据集：
```shell
zfs create virtual_pool/data
```
这个 `virtual_pool/data` 数据集的挂载点会自动设置为 `/mnt/virtual_pool/data`，并因为其默认的 `canmount=on` 属性而被成功挂载。这种方式让存储结构更清晰、管理更方便。

## 总结

通过创建基于文件的虚拟存储池，我们可以在 TrueNAS SCALE 中为 Incus 实现一种灵活、经济的存储方案。这对于熟悉命令行的专家来说，是平衡成本与功能需求的实用技巧。

但请务必清醒地认识到它的局限性：它是一个脱离了 TrueNAS 核心管理体系的"孤岛"，其性能和功能均有折衷。对于任何关键的生产环境，最佳实践永远是使用**独立的物理硬盘**来构建由 GUI 管理的高可用存储池。


