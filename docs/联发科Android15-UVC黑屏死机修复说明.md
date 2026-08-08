# 联发科 Android 15 UVC 黑屏与死机修复说明

## 1. 文档信息

- 修复日期：2026-08-08
- 项目：AndroidUSBCamera
- 修复提交：`ebcb02ff3b979d1cb83df50b1ce1fe9dfe3b122c`
- 测试设备：联想 TB311FU，Android 15，MediaTek 平台
- UVC 设备：`2bdf:02a8`，`2K USB Camera`
- USB 模式：High Speed，480 Mbit/s，Bulk IN 端点 `0x82`
- 最终默认格式：MJPEG 640×480

## 2. 问题现象

同一只 UVC 摄像头在华为手机和其他平板上可以正常预览，但在 TB311FU 上出现以下问题：

1. Android 可以发现 USB 设备并弹出授权窗口。
2. 应用可以读取 UVC 描述符和支持的 MJPEG/YUYV 分辨率。
3. `startPreview` 返回后始终没有首帧，界面保持黑屏。
4. 关闭黑屏预览或反复启动预览时，平板可能卡死、重启，历史上出现过内核崩溃。
5. 日志同时出现以下警告：

   ```text
   UsbVCInterface: Unknown Video Class Interface subtype: 0x7
   UsbVCInterface: Unknown Video Class Interface subtype: 0xd
   ```

`0x7/0xd` 表示解析器遇到未识别的 Video Class 扩展描述符。摄像头的 VideoStreaming 接口、端点、MJPEG/YUYV 格式和分辨率仍能完整读取，因此该警告不是本次黑屏的直接原因。

## 3. 定位证据

### 3.1 修复前

摄像头协商得到的单次 Bulk 负载为 204800 字节。平板 usbfs 能力为 `0x1f7`，支持大 Bulk 包和 Bulk Continuation，但不支持 `USBFS_CAP_BULK_SCATTER_GATHER`（`0x08`）。

libusb 因此把每个 204800 字节请求拆成 13 个 16 KiB URB：

```text
UVC bulk stream: interface=1 endpoint=0x82 payload=204800 frame=460800
bulk submit endpoint=0x82 length=204800 caps=0x1f7 chunk=16384 urbs=13 continuation=1
```

提交成功后没有任何视频 Bulk 完成回调：

```text
raw=0 rejected=0 renderReady=0 decodeFailures=0 javaCallbacks=0
```

原实现同时保持 10 个 UVC transfer，相当于最多排队 130 个视频 URB。黑屏关闭时需要取消大量未完成 URB，放大了 MediaTek USB 主控驱动的卡死风险。

### 3.2 上游源码结论

检查 libusb 1.0.30 的 `submit_bulk_transfer` 后确认，上游仍按以下顺序选择 Bulk 策略：

1. 支持 scatter-gather：提交单个大 URB。
2. 支持 bulk-continuation：拆成多个 16 KiB URB。
3. 只支持 no-packet-size-limit：提交单个大 URB。

因此，仅替换为官方新版 libusb，TB311FU 仍会命中第二条 continuation 路径，不能直接解决该设备问题。

上游参考：

- [libusb linux_usbfs.c](https://github.com/libusb/libusb/blob/v1.0.30/libusb/os/linux_usbfs.c)
- [libuvc](https://github.com/libuvc/libuvc)

## 4. 根因

本次问题的完整链路为：

```text
摄像头协商 204800 字节 Bulk IN
  → MediaTek usbfs 无 scatter-gather 能力
  → libusb 优先拆成 13 个 continuation URB
  → 主控驱动不返回该 continuation 链
  → libuvc 永远收不到视频 payload
  → 预览黑屏
  → 关闭时批量取消未完成 URB
  → 平板卡死或内核崩溃风险
```

摄像头本身、USB Hub、UVC 格式描述符和 MJPEG 解码器均不是主要根因。

## 5. 修复方案

### 5.1 对中等大小 Bulk IN 使用单 URB

修改文件：

```text
libuvc/src/main/jni/libusb/libusb/os/android_usbfs.c
```

满足以下条件时，不再使用 continuation 拆分：

- 数据方向为 IN；
- usbfs 支持 `USBFS_CAP_NO_PACKET_SIZE_LIM`；
- 长度大于 16 KiB；
- 长度不超过 256 KiB；
- 主控没有命中原有 scatter-gather 分支。

本摄像头的 204800 字节负载因此变为：

```text
bulk submit endpoint=0x82 length=204800 caps=0x1f7 chunk=204800 urbs=1 continuation=0
```

256 KiB 上限用于控制兼容性影响。更大的通用 USB Bulk 传输仍保留上游拆分策略，避免内核申请过大连续物理内存。

### 5.2 UVC transfer 数量从 10 降为 2

修改文件：

```text
libuvc/src/main/jni/libuvc/include/libuvc/libuvc_internal.h
```

```c
#define LIBUVC_NUM_TRANSFER_BUFS 2
```

两个 204800 字节请求可以持续覆盖当前 High Speed 视频流，同时显著减少停止预览时需要取消的内核请求数量。

### 5.3 确保 APK 使用当前源码编译的 SO

修改文件：

```text
libuvc/build.gradle
```

原项目同时存在旧的 `src/main/jniLibs` 预编译库。仅修改 C/C++ 源码时，APK 仍可能打包旧 SO，造成“源码已改但设备行为不变”。

现在构建流程为：

1. `preBuild` 强制依赖 `ndkBuild`。
2. 当前 C/C++ 源码输出到 `src/main/libs/<ABI>`。
3. Android Gradle Plugin 只从 `src/main/libs` 打包 JNI 库。
4. 没有源码的 `libUACAudio.so` 从原 `jniLibs` 复制到输出目录。

### 5.4 增加无首帧保护

修改文件：

```text
app/src/main/java/com/jiangdg/demo/DemoFragment.kt
```

相机打开后注册预览数据回调。如果 3 秒内没有收到首个 NV21 帧，立即关闭当前预览并报告错误，避免黑屏会话长时间占用 USB/渲染资源。

重要限制：同一个物理 USB 会话发生无首帧或卡帧后，不得自动重复调用 `startPreview`。必须完整关闭会话并物理重插 UVC 后再测试。

### 5.5 使用 SurfaceView 进行 NativeWindow 渲染

`NORMAL` 渲染模式通过 `ANativeWindow_lock` 写入 RGBA 数据。示例应用改用 `AspectRatioSurfaceView`，避免 TextureView producer 路径与原生直接写入方式不匹配。

### 5.6 修复 Android 14/15 USB 授权回调

修改文件：

```text
libuvc/src/main/java/com/jiangdg/usb/USBMonitor.java
```

Android 14/15 的 `UsbManager` 需要向 PendingIntent 填充 `EXTRA_DEVICE` 和 `EXTRA_PERMISSION_GRANTED`。使用 immutable PendingIntent 时可能丢失系统补充的 extras，因此 API 31+ 改为 mutable 并使用 `FLAG_UPDATE_CURRENT`。

### 5.7 增加有界诊断日志

新增日志覆盖以下阶段：

- usbfs Bulk 请求长度、分块大小、URB 数量和 continuation 状态；
- libusb transfer 首次完成、周期性完成、取消和重试；
- libuvc 原始帧、拒绝帧、MJPEG 解码、渲染就绪帧；
- Java 预览回调首帧；
- 会话结束汇总。

日志经过计数限制，不会逐帧无限输出。

## 6. SO 与源码对应关系

| 输出 SO | 本仓库源码 | 构建定义 |
|---|---|---|
| `libusb100.so` | `libuvc/src/main/jni/libusb/libusb/` | `libusb/android/jni/libusb.mk` |
| `libuvc.so` | `libuvc/src/main/jni/libuvc/src/` | `libuvc/android/jni/Android.mk` |
| `libUVCCamera.so` | `libuvc/src/main/jni/UVCCamera/` | `UVCCamera/Android.mk` |
| `libjpeg-turbo1500.so` | `libuvc/src/main/jni/libjpeg-turbo-1.5.0/` | `libjpeg-turbo-1.5.0/Android.mk` |
| `libUACAudio.so` | 仓库未提供 C/C++ 源码，仅有 Java 包装层 | 从 `src/main/jniLibs` 复制 |

本次单 URB 修复最终进入 `libusb100.so`；帧和解码诊断进入 `libuvc.so` 与 `libUVCCamera.so`。

当前提交包含以下 ABI：

- `arm64-v8a`
- `armeabi-v7a`

旧 x86/x86_64 ABI 未纳入当前 NDK 25 构建，因为旧 JPEG 汇编代码存在不兼容重定位问题。

## 7. 构建方法

### 7.1 环境

- JDK 11
- Android NDK `25.1.8937393`
- Gradle `6.7.1`
- Gradle wrapper：腾讯云镜像
- Maven：阿里云 Google/Central/Public 镜像

构建时必须只在当前构建子进程中清除代理，不修改用户全局网络设置：

```bash
env -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY \
    -u http_proxy -u https_proxy -u all_proxy \
    ./gradlew :app:assembleDebug --no-daemon --rerun-tasks
```

产物：

```text
app/build/outputs/apk/debug/app-debug.apk
```

已验证 APK SHA-256：

```text
b3db1495ec56b38cf75db64aac8d867fdb7940ba04776faa28bbf281a0cdc1d4
```

安装时保留应用数据：

```bash
adb -s <serial> install -r app/build/outputs/apk/debug/app-debug.apk
```

## 8. 真机测试方法

### 8.1 测试前检查

1. 所有 adb 命令必须指定目标设备：`adb -s <serial>`。
2. 确认 UVC 物理重插后 `/dev/bus/usb/...` 的设备号已经变化。
3. 确认 VID/PID 为 `2bdf:02a8`。
4. 停止可能自动抢占摄像头的其他 UVC 应用。
5. 清空本轮 logcat 后再启动测试应用。
6. 每个新物理 USB 会话只做一次首次启动验证。

### 8.2 成功判据

日志必须依次出现：

```text
processConnect:device=/dev/bus/usb/...
UVC bulk stream: interface=1 endpoint=0x82 payload=204800 ... buffers=2
bulk submit ... length=204800 ... chunk=204800 urbs=1 continuation=0
UVC USB transfer callback #1: status=completed
Decoded first MJPEG UVC frame to YUYV
Delivered first converted UVC frame to Java callback
Received first UVC preview frame
```

如果仍出现以下内容，说明 APK 仍在使用旧 SO 或兼容分支未命中：

```text
chunk=16384 urbs=13 continuation=1
```

### 8.3 失败处理

如果 3 秒内没有首帧：

1. 等待应用安全看门狗关闭预览。
2. 确认 adb 仍在线并保存本轮日志。
3. 不得在相同物理会话再次启动预览。
4. 物理拔插摄像头，确认 USB 设备号变化后再测试。

## 9. 真机验证结果

| 格式 | 分辨率 | 结果 | 关键证据 |
|---|---:|---|---|
| MJPEG | 640×480 | 通过 | 约 80 秒、2399 原始帧、2371 Java 回调 |
| MJPEG | 2560×1440 | 通过 | 首帧解码成功，持续收到 2K MJPEG 帧 |
| YUYV | 640×480 | 通过 | 1035 原始帧，帧长固定 614400，0 次解码失败 |
| 正常关闭 | MJPEG/YUYV | 通过 | 两个 transfer 正常取消，ADB 在线，无 ANR/崩溃 |

测试期间平板未死机，应用未出现新增 Java/native crash，`ApplicationExitInfo` 只有 APK 更新导致的正常进程退出记录。

MJPEG 摄像头启动时可能先返回极短的占位 payload，日志中会出现少量 `error=-98`，随后完整 MJPEG 首帧能够正常解码并持续显示。这类启动期短帧不等同于黑屏故障。

## 10. 排障清单

### 10.1 能发现设备但没有授权成功

- 检查系统授权窗口是否被其他 UVC 应用抢占。
- 检查 `USBMonitor` 是否收到 `EXTRA_PERMISSION_GRANTED=true`。
- 检查打包代码是否包含 mutable PendingIntent 修复。

### 10.2 有授权和格式列表，但没有 USB 回调

- 检查 `bulk submit` 是否仍为 `urbs=13 continuation=1`。
- 用 `strings libusb100.so | grep "bulk submit"` 确认 APK 中包含当前诊断字符串。
- 检查 APK 是否从 `src/main/libs` 打包，而不是旧 `jniLibs`。

### 10.3 有 USB 回调但没有原始帧

- 检查 UVC payload header、FID/EOF 和 `dwMaxVideoFrameSize`。
- 检查 `raw/rejected` 计数。
- 对比 MJPEG 与 YUYV 协商结果。

### 10.4 有原始帧但画面仍黑

- 检查 `Decoded first MJPEG` 或 `Received first render-ready YUYV`。
- 检查 `Delivered first converted UVC frame to Java callback`。
- 确认 NORMAL 模式使用 SurfaceView。
- 检查是否停留在效果选择、权限或其他覆盖页面。

### 10.5 Hub 下设备不出现

- 检查 Hub 供电和 OTG 主机模式。
- 检查 `/sys/bus/usb/devices` 是否存在目标 VID/PID。
- 检查摄像头是否以 480 Mbit/s 枚举。
- 在系统完全没有 UVC 节点时不要启动预览。

## 11. 已知边界

### 11.1 其他设备是否自动适配

兼容逻辑没有写死设备品牌、型号、VID/PID 或 MJPEG/YUYV 格式。其他 Android 设备只要同时满足以下条件，就会自动使用本次单 URB 路径：

1. USB 请求方向为 IN；
2. usbfs 声明支持 `USBFS_CAP_NO_PACKET_SIZE_LIM`；
3. 主控没有 `USBFS_CAP_BULK_SCATTER_GATHER`，否则原逻辑本来就会使用单 URB；
4. 单次请求大于 16 KiB且不超过 256 KiB；
5. 传输类型最终进入 libusb 的 Bulk/Interrupt 提交函数。

因此，其他品牌手机或平板如果与 TB311FU 一样，表现为“UVC 协商成功，但 16 KiB continuation URB 链始终没有完成回调”，通常可以直接受益，不需要再添加新的设备白名单。

| 设备/传输条件 | 实际行为 |
|---|---|
| 有 scatter-gather 能力 | 继续使用上游单 URB 路径 |
| 无 scatter-gather，但支持大包，IN 请求为 16～256 KiB | 使用本次兼容单 URB 路径 |
| 请求不超过 16 KiB | 保持原逻辑 |
| 请求大于 256 KiB | 保持上游拆分策略 |
| 内核不声明支持大包 | 保持上游拆分策略 |
| UVC 使用 Isochronous 端点 | 本补丁不介入 |

如果其他设备虽然黑屏，但已经持续收到 USB transfer callback，则根因更可能位于 UVC payload 拼帧、MJPEG 解码、YUYV 转换或渲染链路，本补丁不会解决这一类问题。必须先比较 `bulk submit`、USB callback、`raw/renderReady/javaCallbacks` 三层日志，不能仅凭“画面黑”判断为同一故障。

### 11.2 其他限制

1. 单 URB 上限为 256 KiB；大于该值的传输继续使用上游策略。
2. 示例应用的无首帧保护不会自动覆盖所有接入 `libausbc` 的业务页面，业务应用需要实现相同会话保护。
3. `libUACAudio.so` 没有随仓库提供原生源码，无法与其他四个 SO 一样完全重建。
4. 当前真机验收覆盖约分钟级运行和正常关闭，不代替数小时长稳、反复插拔、低电量或多 Hub 压力测试。
5. 本仓库内置的 libusb/libuvc 版本较旧；后续整体升级时必须重新验证该单 URB 补丁是否仍需要。

## 12. 回退

需要整体撤销本次修复时，可在确认工作区干净后执行：

```bash
git revert ebcb02ff3b979d1cb83df50b1ce1fe9dfe3b122c
```

不要只恢复旧预编译 SO 而保留新的 Java/Kotlin 逻辑，否则源码、打包产物和实际运行行为会再次不一致。
