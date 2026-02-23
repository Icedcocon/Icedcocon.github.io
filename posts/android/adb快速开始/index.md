# ADB 快速开始（Mac 连接电视盒子）


## 使用流程（最短路径）

1. Mac 上下载 APK（浏览器或 `curl` 均可）
2. 通过 ADB 与电视盒子建立连接
3. 传输并安装 APK（`adb install` 或 `adb push` + `pm install`）

## 安卓端配置（电视盒子）

### Android 11+ 无线调试（推荐）

1. 打开开发者选项
2. 打开无线调试
3. 进入「无线调试」页，选择「使用配对码配对设备」或「使用二维码配对设备」
4. 记下屏幕显示的 IP 与端口，并使用配对码完成配对

> 电视设备需要 Android 13+ 才支持无线调试配对流程。[(Android Developers)](https://developer.android.com/tools/adb)

### 旧版 Wi‑Fi 方式（需先 USB）

1. 打开开发者选项
2. 打开 USB 调试
3. 通过 USB 与 Mac 连接一次，允许授权弹窗
4. 使用 `adb tcpip 5555` 切换到 Wi‑Fi 端口，再执行 `adb connect <ip:5555>`

## ADB 命令词典（按功能分类）

### 连接与配对（无线调试）

| 命令                         | 说明                                             |     |
| -------------------------- | ---------------------------------------------- | --- |
| `adb start-server`         | 启动 adb 服务                                      |     |
| `adb kill-server`          | 停止 adb 服务                                      |     |
| `adb devices -l`           | 查看已连接设备（含详细信息）                                 |     |
| `adb pair <ip:port>`       | Android 11+ 无线配对，电视需 Android 13+；使用屏幕上的配对码完成配对 |     |
| `adb connect <ip:port>`    | 连接已配对设备                                        |     |
| `adb disconnect`           | 断开所有连接                                         |     |
| `adb disconnect <ip:port>` | 断开指定设备                                         |     |

> 无线调试需要设备与 Mac 在同一局域网，并使用配对码/二维码完成配对，然后再执行连接命令。安卓官方说明可参考 ADB 文档。[(Android Developers)](https://developer.android.com/tools/adb)

### 连接与配对（旧版 Wi‑Fi 方式，需先 USB）

| 命令 | 说明 |
| --- | --- |
| `adb usb` | 切回 USB 连接模式 |
| `adb tcpip 5555` | 切换到 TCP/IP 端口（需先 USB 连接） |
| `adb connect <ip:5555>` | 通过固定端口连接（老方式） |

### 设备选择与状态

| 命令 | 说明 |
| --- | --- |
| `adb -s <serial> <command>` | 多设备时指定目标设备 |
| `adb wait-for-device` | 等待设备上线 |
| `adb get-state` | 获取设备连接状态 |

### 文件传输

| 命令 | 说明 |
| --- | --- |
| `adb push <local> <remote>` | 从 Mac 传文件到设备 |
| `adb pull <remote> <local>` | 从设备拉取文件到 Mac |
| `adb sync` | 同步文件（按分区） |

### 下载与安装（围绕 “Mac 下载 → 电视盒子安装”）

| 命令 | 说明 |
| --- | --- |
| `adb install <apk>` | 直接安装本地 APK |
| `adb install -r <apk>` | 覆盖安装（保留数据） |
| `adb install -d <apk>` | 允许降级安装 |
| `adb install -g <apk>` | 安装时授予所有运行时权限 |
| `adb install-multiple <apk...>` | 安装 Split APK |
| `adb push <apk> /sdcard/Download/` | 先传到设备再安装 |
| `adb shell pm install -r /sdcard/Download/<apk>` | 从设备路径安装 |
| `adb shell pm uninstall <package>` | 卸载应用 |
| `adb shell pm list packages` | 列出已安装包名 |
| `adb shell pm path <package>` | 查看已安装 APK 路径 |

### 远程控制与排障

| 命令 | 说明 |
| --- | --- |
| `adb shell` | 进入设备 Shell |
| `adb shell getprop ro.product.model` | 查看设备型号 |
| `adb shell getprop ro.build.version.release` | 查看安卓版本 |
| `adb shell am start -n <pkg>/<activity>` | 启动指定页面 |
| `adb logcat` | 查看日志 |
| `adb logcat -s <TAG>` | 按标签过滤日志 |

## 典型示例

### 1) 无线配对并连接

```bash
adb pair 192.168.1.20:37123
adb connect 192.168.1.20:37127
adb devices -l
```

### 1.1) 认证失败的处理（failed to authenticate）

如果出现 `failed to authenticate`，通常是以下原因之一：

- 电视端未完成配对流程，或配对码已过期
- 电视端弹出了授权对话框但未确认
- 旧版 Wi‑Fi 方式仍依赖首次 USB 授权

处理步骤：

1. 在电视盒子上打开「无线调试」并重新生成配对码
2. Mac 上执行 `adb kill-server` 后重新 `adb pair <ip:port>`
3. 配对成功后再执行 `adb connect <ip:port>`
4. 若使用旧版 Wi‑Fi 方式，先用 USB 连接并确认授权弹窗，再执行 `adb tcpip 5555`

### 1.2) unauthorized 与 ADB_VENDOR_KEYS 提示

如果出现 `unauthorized` 或提示 `ADB_VENDOR_KEYS is not set`，说明设备侧还未授权当前 Mac 的 ADB 公钥。

处理步骤：

1. 电视端打开「USB 调试」或「无线调试」后，查看是否弹出授权对话框并确认
2. 若未弹出授权对话框，先断开连接并清理授权记录
3. 重新触发授权弹窗后再连接

可用命令：

- `adb kill-server`
- `adb disconnect`
- `adb connect <ip:port>`

如果电视盒子可进入开发者选项，执行一次“撤销 USB 调试授权”，再重新连接即可生成新的授权弹窗。

### 2) Mac 下载 APK 后安装到电视盒子

```bash
curl -L -o app.apk https://example.com/app.apk
adb install app.apk
```

### 3) 先传到盒子再安装

```bash
adb push app.apk /sdcard/Download/
adb shell pm install -r /sdcard/Download/app.apk
```

