# 根据 GhostLock 进行 root 提权

记录日期：2026-09-30。本文件总结本次在个人 Redmi K70 Pro 上实际完成的流程；设备或固件变化后，需要重新核对兼容性。

## 1. 本次设备与结论

| 项目 | 实测值 |
| --- | --- |
| 机型 | Redmi K70 Pro，设备代号 `manet` |
| 系统 | HyperOS `OS2.0.208.0.VNMCNXM`，Android 15 |
| 完整内核版本 | `6.1.118-android14-11-ga3b9c44908dd-ab13320413` |
| Bootloader | `ro.boot.flash.locked=1`，`ro.boot.vbmeta.device_state=locked` |
| 启动验证状态 | `ro.boot.verifiedbootstate=green` |
| Root 验证 | `uid=0(root) gid=0(root) context=u:r:ksu:s0` |
| KernelSU 管理器 | `me.weishu.kernelsu`，本次检查版本 `v3.3.0` |
| GhostLock | `com.ghostlock.app`，版本 `1.0`，versionCode `65` |

本次安装 GhostLock 后，由用户在手机上完成提权，随后通过 ADB 确认 Root 有效。没有刷写 boot/init_boot，没有解锁 Bootloader，没有设置开机自动提权。

**这是运行期间的临时 Root，不应视为永久 Root。** 本次没有通过重启测试持久性；按这条利用路线，重启后应预期需要重新执行 GhostLock。KernelSU 管理器已安装、应用能获得授权，都不等于 Root 已写入启动镜像。

## 2. 连接 ADB

电脑为本地 Linux，用户名 `guga`。ADB 已安装于：

```text
/opt/android-sdk/platform-tools/adb
```

手机开启开发者选项和 USB 调试，用数据线连接，解锁屏幕并允许电脑的 USB 调试请求。

```bash
adb devices -l
adb shell uname -r
```

设备状态必须为 `device`。本次最初出现 `unauthorized`，电脑的 `~/.android/adbkey` 已存在；重启 ADB 后仍需手机端确认授权：

```bash
adb kill-server
adb start-server
adb devices -l
```

没有授权弹窗时，可以重新插拔数据线；必要时在开发者选项撤销 USB 调试授权，再重新连接。错误提示中的 `ADB_VENDOR_KEYS is not set` 不代表必须手动设置这个变量，也不要为此直接删除原来的密钥。

## 3. 核对完整内核，而不是只看机型或处理器

使用的代码仓库：<https://github.com/EVcake/GhostLock>。这是本次实际选择的分支，不应把任意同名仓库或 APK 当成同一版本。

该仓库按完整 `uname -r` 匹配。本次实测字符串与其支持表完全一致：

```text
6.1.118-android14-11-ga3b9c44908dd-ab13320413
```

支持表中这一条标注为 Redmi Note 15 Pro+，但实际匹配字段是完整内核字符串。匹配成功只说明存在对应配置，不保证每次利用成功。不要凭“处理器一样”混用其他设备的 boot、偏移量或提权文件。

## 4. 本次 APK 如何获得

当时检查发现，该仓库没有 Releases 安装包，也没有可下载的 Actions 产物，因此采用本地源码构建。

| 项目 | 本次构建记录 |
| --- | --- |
| 源码提交 | `c9cf13d560eba92929e96723f8b35fed83444a19` |
| Git 提交计数 | `65`，构建前补齐了 Git 历史 |
| Gradle | 本机 `/opt/gradle`，版本 `9.7.1` |
| Java | `/opt/android-studio/jbr`，JDK `25.0.3` |
| 原生工具链 | `topjohnwu/ondk`，`r30.1`，使用其 Rust 工具链 |
| Android SDK | 原配置要求 compileSdk 37，但 SDK 管理器当时未找到相应包；本次改用 compileSdk 36、buildTools 36.0.0 |
| Android 版本声明 | minSdk 34，targetSdk 37 保持原值 |
| APK 类型 | release 构建，使用本地生成的调试签名，非作者发布签名 |

只调整了 `app/build.gradle` 的编译 SDK 与 buildToolsVersion，没有修改提权代码或内核配置。构建命令的核心是：

```bash
# 在工具链、Android SDK、Rust/ONDK 均配置好后，于源码目录执行：
/opt/gradle/bin/gradle :app:assembleRelease --no-daemon --console=plain --max-workers=4
```

输出原文件名为 `GhostLock-v1.0(65)-release.apk`，保留时改名为 `GhostLock-v1.0-65-c9cf13d.apk`。

临时 Rust、ONDK、Gradle 缓存等放在 `/opt/android-sdk/ghostlock-build`，源码放在会话工作目录。按用户要求，构建后已删除源码、临时工具、缓存及签名私钥，并确认构建进程退出。**上述临时环境现在已不存在；重新编译需要重新准备。**

原签名私钥已经删除，未来重新构建的 APK 可能无法直接覆盖当前安装版本。不要把签名不同导致的安装失败当成内核不支持；卸载重装会清除该应用数据。

## 5. 保留的 APK 与安装验证

APK：

[GhostLock-v1.0-65-c9cf13d.apk](/home/guga/Documents/Codex/2026-09-30/guga-adb-shell-uname-r-adb/outputs/GhostLock-v1.0-65-c9cf13d.apk)

SHA-256：

```text
bf07ecc0c544a63ba7bc3bc22eaa8a63dd43654833a3b577ffccb01ee648335e
```

安装命令：

```bash
adb install -r '/home/guga/Documents/Codex/2026-09-30/guga-adb-shell-uname-r-adb/outputs/GhostLock-v1.0-65-c9cf13d.apk'
```

本次结果为 `Success`，APK v2 签名校验通过；手机中实际安装的 `base.apk` 与保留文件的 SHA-256 完全一致。

## 6. 使用与验证 Root

1. 先安装 KernelSU 管理器。本次手机已经安装，因此未重复安装。
2. 打开 GhostLock，确认当前内核显示受支持。
3. 保存正在编辑的内容，再由用户执行应用中的提权操作。失败可能导致应用异常、手机重启或卡死。
4. 按需在 KernelSU 中给可信应用授权；不要无差别授权全部应用。
5. 通过 ADB 验证：

```bash
adb shell "su -c 'id'"
```

本次返回包含 `uid=0(root)` 和 `context=u:r:ksu:s0`，确认 Root 已可用。后续成功用于系统更新组件冻结、Blocker 排查和广告组件停用。

GhostLock 的说明区分了获得 uid 0 与加载 KernelSU 模块环境；仅安装一个管理器 APK 并不等于已经具备完整或持久的 Root 环境。

## 7. 锁定 Bootloader 时不要把临时 Root 直接刷成永久 Root

本次手机仍为 locked/green。常规 KernelSU 持久安装会涉及修改 boot 或 init_boot；有运行时 Root 不等于修改后的镜像可以通过下一次启动验证。

因此，本方案没有执行 KernelSU 的“直接安装”“安装到另一槽位”，也没有修改 AVB、vbmeta 或启动分区。没有查证到适用于本次设备和固件、保持 Bootloader 锁定的可靠永久 Root 方案。

参考：[KernelSU 安装说明](https://kernelsu.org/guide/installation.html)。

## 8. 本次还冻结了系统 OTA 更新组件

用于保持当前系统环境，操作对象为 `com.android.updater`（8.9.5）。正常 `pm disable-user` 被 HyperOS 的核心包保护拦截，所以使用了可恢复的隐藏方式：

```bash
adb shell "su -c 'pm hide --user 0 com.android.updater'"
adb shell "su -c 'am force-stop --user 0 com.android.updater'"
```

已验证 `hidden=true`、`stopped=true`，无运行服务，系统检查更新入口无法解析。手动 OTA 更新入口也会暂时不可用；这不是阻止所有可能的固件更新方式，更不会把临时 Root 变成永久 Root。

恢复：

```bash
adb shell "su -c 'pm unhide --user 0 com.android.updater'"
```

系统 APK 未删除。没有为验证而重启手机。
