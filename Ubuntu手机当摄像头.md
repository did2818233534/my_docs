# Ubuntu 手机当摄像头：USB 连接、后台运行与画面方向

记录日期：2026-10-12；更新日期：2026-10-13。适用环境：Ubuntu 桌面系统 + 支持 USB 调试的 Android 手机；本次验证环境为 Ubuntu 24.04.4 LTS（x86_64）、Android 15、scrcpy 5.0.1。

本文用于换机后复现：手机通过 USB 提供真实摄像头画面，Ubuntu 应用通过标准摄像头接口使用。手动运行 `phone-webcam connect` 后在后台持续采集，不显示视频预览窗口；运行 `phone-webcam disconnect` 或拔掉 USB 后停止。**默认后置镜头、竖屏正向、不镜像，优先采集 1920×1080，输出 1080×1920。**

## 给 ChatGPT 的执行提示词

把本文交给能操作目标电脑的 ChatGPT，并附上下面这段提示词：

```text
请依据《Ubuntu手机当摄像头.md》，直接在这台 Ubuntu 电脑上完成检查、安装、脚本部署和验证。

目标：
1. 通过 USB 使用 Android 手机摄像头，让普通摄像头应用可以选择 Android USB Camera。
2. 使用 phone-webcam connect 手动启动，phone-webcam disconnect 断开，status 查询状态，resolutions 查询当前镜头支持的分辨率编号。
3. 启动后无预览窗口，后台持续运行，关闭启动它的终端不影响采集；拔掉 USB 后停止，不自动重连。
4. 默认竖屏正向、不镜像、自动对焦；支持横屏、倒置、左右镜像、按编号选择分辨率，以及查询、锁定、手动设置、恢复自动对焦。connect 必须检查实际收到的画面后报告成功。
5. --help 和子命令帮助使用中文，表格按终端显示宽度对齐，正常内容适合 80 列终端。
6. 如果我使用 BigCam，让这个手机来源出现在普通摄像头列表中，保留对 BigCam 自己的虚拟输出设备的过滤。

执行要求：
1. 先检查实际 Ubuntu 版本、CPU 架构、内核、现有 ADB/scrcpy、USB 授权、V4L2 设备和 systemd 用户会话。不能假定用户名、手机序列号、分辨率编号、镜头 ID 或内核版本与记录相同。
2. 软件本体安装到 /opt/scrcpy；默认装进主目录的软件改放 /opt/软件名。复用已有合适版本，不清空已有软件目录。
3. 优先使用当前内核已有的签名 v4l2loopback 模块。确实缺少时才安装匹配的内核头文件和兼容模块；遇到 Secure Boot 或新内核问题，诊断原因，不默认关闭安全启动。
4. 使用本文附录 A 的完整脚本。对焦控制还依赖同仓库的 `phone-webcam-focus-server.patch`，先按第 4 节构建并安装定制的 scrcpy 手机端服务；只有脚本、没有手机端补丁时，对焦命令不能工作。ADB 默认复用 scrcpy 静态包内的 /opt/scrcpy/adb；已有兼容 ADB 时也可调整常量。不要重启或杀死无关 ADB 会话。
5. 不添加开机启动、登录启动、USB 插入触发、永久启用的服务、自动重连或模块自动加载配置。只在 connect 时按需加载模块和创建临时用户服务。
6. 写入前备份已有脚本和要修改的 BigCam 文件。若 /dev/video10 已属于其他设备，选择空闲编号并统一调整脚本，不能覆盖原设备。
7. 切换分辨率或横竖屏前释放摄像头读取应用；应用仍占用且输出尺寸要改变时，报错并保留原连接。错误编号和冲突参数也不能破坏原连接。
8. 不把 BigCam 中的普通摄像头分类说成内核已经变为物理 USB 摄像头。手机是真实画面来源，但本文使用 v4l2loopback 适配，驱动身份仍可识别。
9. 实测竖屏正向的画面方向。不同手机传感器可能不同，必要时使用 --rotate 校准；方向固定，不做手机姿态自动跟随。应用本身的预览镜像也要单独检查。
10. 需要管理员密码时，让我在系统认证窗口或本机终端输入，不要求在聊天里发送密码。
11. 用限时、有限帧的方式验收。清理临时文件和后台测试，保留正式连接及备份；不要擅自结束无关应用。不要自动在侧边栏打开编辑后的文档。
12. 报告安装路径、实际连接/输出尺寸、已完成的测试和未测试事项。手机声明支持的分辨率不能直接写成实测稳定支持。
```

## 本次最终配置

| 内容 | 版本、位置或行为 |
| --- | --- |
| 电脑 | Ubuntu 24.04.4 LTS，x86_64；本次内核 `7.0.0-38-generic` |
| 手机 | Redmi K70 Pro，设备型号 `23117RK66C`，Android 15 |
| scrcpy | 官方 Linux x86_64 静态包 5.0.1，`/opt/scrcpy` |
| ADB | `/opt/scrcpy/adb`，静态包附带；与本机 Android SDK 的 ADB 兼容 |
| V4L2 模块 | 本次内核已有签名的 `v4l2loopback` 0.15.3，未另装 DKMS |
| CLI 本体 | `/opt/scrcpy/phone-webcam`，仅依赖 Python 3 标准库 |
| 命令入口 | `/usr/local/bin/phone-webcam` → `/opt/scrcpy/phone-webcam` |
| 摄像头名称 | `Android USB Camera` |
| 摄像头设备 | `/dev/video10`，`exclusive_caps=1` |
| 后台服务 | 手动创建的临时用户服务 `phone-webcam.service`，`Restart=no` |
| 对焦控制 | 默认自动；脚本与定制的 scrcpy 手机端服务配合，支持 `focus status/lock/set/auto` |
| 默认连接 | 后置、1920×1080 采集、1080×1920 输出、请求 30 FPS、竖屏正向、不镜像 |
| 停止方式 | 手动 `disconnect` 或 USB 断开；不自动重连 |
| BigCam | 4.5.0，针对这个手机来源增加一项检测例外 |

用户名 `guga`、手机序列号和本次分辨率编号只是记录，部署不依赖这些固定值。程序每次连接都会查询当前唯一已授权的 USB 手机。

## 整体思路与边界

```text
Android 实体摄像头 → scrcpy 手机端采集/旋转/镜像 → USB ADB
                  → Ubuntu scrcpy 解码 → /dev/video10 → BigCam/浏览器/会议软件
```

Linux 使用 `v4l2loopback` 把采集到的画面接入摄像头接口。只给这个来源设置容易识别的名称，并在 BigCam 中作为普通摄像头来源展示；这不会把驱动变成 `uvcvideo`，也不能保证绕过所有应用对虚拟摄像头的限制。`exclusive_caps=1` 使设备在有画面生产者时向应用呈现采集能力，有利于摄像头软件识别。见 [v4l2loopback 官方说明](https://github.com/v4l2loopback/v4l2loopback#options)。

本次曾尝试手机原生 USB UVC 功能：USB HAL 返回状态 4，随后回到 ADB；虽然系统存在 `com.android.DeviceAsWebcam`，仍未实现电脑直接识别手机为物理 UVC 摄像头。Root 并不能替代固件/内核支持，因此最终采用上述方案，**无需 Root**，也没有修改手机固件、SELinux 或 USB 系统属性。

本文只传视频，不传手机麦克风，不录制文件。手动断开后，`/dev/video10` 节点可能仍存在，但不再持续产生手机画面；这不表示还在连接。正常桌面会话中关闭终端不会断开，注销/关机不承诺继续采集。拔掉再插回手机后，需要重新运行 `connect`。手机无需 Root；本次 Redmi K70 Pro 虽已 Root，但摄像头转接和对焦控制没有使用 Root 权限。

## 1. 检查环境和 USB 授权

在 Ubuntu 的普通桌面用户终端操作，**不要用 `sudo phone-webcam connect`**。整个程序依赖该用户的 `systemd --user` 会话，只在首次加载驱动时内部调用 `sudo modprobe`。

```bash
cat /etc/os-release
uname -m
uname -r
command -v python3 adb scrcpy v4l2-ctl
systemctl --user is-system-running
v4l2-ctl --list-devices
modinfo v4l2loopback
```

`systemctl --user is-system-running` 显示 `running` 或因其他服务显示 `degraded` 时，还要确认用户总线可访问；出现无法连接总线的错误，应先在本机桌面会话中执行。不要默认用 root 服务代替。

手机开启开发者选项和 USB 调试，使用能传数据的 USB 线，解锁手机并接受电脑的 USB 调试授权。安装 ADB 后检查：

```bash
/opt/scrcpy/adb devices -l
/opt/scrcpy/adb -d get-state
```

应看到一个 USB 设备，状态为 `device`。`unauthorized` 表示手机尚未授权；多个 USB 手机同时连接时，本文脚本会拒绝继续，应先只保留目标手机。Android 摄像头采集要求 Android 12 或更新版本，厂商限制仍可能造成失败。见 [scrcpy 摄像头功能说明](https://github.com/Genymobile/scrcpy/blob/v5.0.1/doc/camera.md)。

## 2. 安装 scrcpy 和命令依赖

先检查是否已有可用的软件。如果要复现本文固定版本，以下下载块仅适用于 `uname -m` 为 `x86_64` 的机器；其他架构应按 [官方 Linux 安装文档](https://github.com/Genymobile/scrcpy/blob/v5.0.1/doc/linux.md)选择对应构建方式，不能运行这个 x86_64 包。

```bash
sudo apt update
sudo apt install python3 v4l-utils curl ca-certificates
```

Ubuntu 旧版本软件源中的 scrcpy 可能缺少本文所需选项，不能只看命令存在就认为可用。以下块按官方 5.0.1 包安装；若 `/opt/scrcpy` 已存在，先检查版本并备份，按实际情况复用/升级，不直接覆盖。

```bash
bash <<'SH'
set -euo pipefail
test "$(uname -m)" = x86_64
if test -e /opt/scrcpy; then
    echo '/opt/scrcpy 已存在，请先检查和备份，再决定是否安装。' >&2
    exit 1
fi
webcam_stage=$(mktemp -d /tmp/phone-webcam-install.XXXXXX)
trap 'rm -r -- "$webcam_stage"' EXIT
webcam_archive=scrcpy-linux-x86_64-v5.0.1.tar.gz
curl -fL --retry 3 -o "$webcam_stage/$webcam_archive" \
    "https://github.com/Genymobile/scrcpy/releases/download/v5.0.1/$webcam_archive"
printf '%s  %s\n' \
    9f969d30cc574816077edecda65719c1c70b0aee1df1ba5fa00779ac07b8fd6e \
    "$webcam_stage/$webcam_archive" | sha256sum -c -
tar -xzf "$webcam_stage/$webcam_archive" -C "$webcam_stage"
sudo install -d -m 755 /opt/scrcpy
sudo cp -a "$webcam_stage/scrcpy-linux-x86_64-v5.0.1/." /opt/scrcpy/
sudo chown -R root:root /opt/scrcpy
/opt/scrcpy/scrcpy --version
/opt/scrcpy/adb version
SH
```

下载包及校验值来自 [scrcpy 5.0.1 官方发布](https://github.com/Genymobile/scrcpy/releases/tag/v5.0.1)。若需要改版本，重新核对来源、校验值和所有参数，再做验收；不要对新包沿用旧哈希。

脚本使用包内 ADB，不需要为了它新建主目录下的 Android SDK。已有兼容 ADB 可以继续使用，只需统一附录脚本中的 `ADB` 常量和诊断命令。本机之前使用 `/opt/android-sdk/platform-tools/adb`，最终脚本改用 `/opt/scrcpy/adb` 便于换机复现，没有停止共享 ADB 服务。

## 3. 准备 V4L2 模块

优先复用当前内核已提供的模块：

```bash
modinfo -F filename v4l2loopback
modinfo -F version v4l2loopback
modinfo -F signer v4l2loopback
lsmod | rg v4l2loopback
```

没有 `rg` 时用 `grep` 即可。模块存在并能加载，就跳过 DKMS 安装。本次使用内核已有的 0.15.3 模块；不要因为本文提到 DKMS 就把它替换成旧的软件包。

**仅当模块不存在，且当前 Ubuntu 软件源中的版本兼容当前内核时**，执行：

```bash
sudo apt install "linux-headers-$(uname -r)" v4l2loopback-dkms
modinfo v4l2loopback
```

DKMS 是按内核编译模块的机制。头文件包不存在、编译失败或模块签名被拒绝时，应检查日志和当前发行版支持的模块版本，不继续假定安装成功。Secure Boot 下按 Ubuntu 的模块签名/MOK 流程处理，不能以关闭安全启动作为默认解决方式。参见 [v4l2loopback 构建要求](https://github.com/v4l2loopback/v4l2loopback#build)和 [Ubuntu Secure Boot 说明](https://wiki.ubuntu.com/UEFI/SecureBoot)。

部署前确认 `/dev/video10` 没有属于其他摄像头。本文脚本只会创建/使用名称为 `Android USB Camera` 的 `video10`，不会替换已有硬件。如果 `v4l2loopback` 已为其他用途加载，应先检查现有设备，不能卸载仍在使用的模块；必要时由 ChatGPT 选择空闲设备号，并统一修改脚本中的 `VIDEO`、sysfs 路径和 `modprobe video_nr`。

脚本内部按需执行的加载命令为：

```bash
sudo modprobe v4l2loopback devices=1 video_nr=10 \
    card_label='Android USB Camera' exclusive_caps=1
```

它不是开机配置。**不创建** `/etc/modules-load.d/`、udev 热插拔、桌面自启动或永久服务文件；如果安装包新增了自动加载配置，应查明来源并按本次“只手动启动”的目标调整，保留原配置备份。

## 4. 部署 phone-webcam 脚本和对焦服务

附录 A 给出了当前 CLI 的完整源码。可将其保存为 `/opt/scrcpy/phone-webcam`，权限 `755`，再创建入口链接。基础摄像头转接使用 scrcpy 5.0.1；**对焦控制额外依赖定制的 scrcpy 手机端服务**。同仓库的 [`phone-webcam-focus-server.patch`](phone-webcam-focus-server.patch) 是基于 scrcpy 5.0.1 的 `CameraCapture.java` 改动，包含 Camera2 对焦控制和实际镜头位置反馈；不能只更新 Python 脚本便认为对焦已可用。

在另一台机器复现对焦功能时，应从 scrcpy 5.0.1 源码按官方构建说明准备 Android 构建环境，在源码根目录应用该补丁并构建手机端 `scrcpy-server`，先备份原 `/opt/scrcpy/scrcpy-server`，再安装构建产物。补丁只适用于相符的源码版本；若 `git apply --check` 失败，应检查版本，不要强行套用。可先检查：

```bash
git apply --check /path/to/my_docs/phone-webcam-focus-server.patch
```

服务文件是 scrcpy 启动时传到手机的组件；仅更换电脑端的 `/opt/scrcpy/scrcpy` 不会获得对焦功能。官方源码及构建方法见 [scrcpy 项目](https://github.com/Genymobile/scrcpy/tree/v5.0.1)。本机已安装并实测定制服务，未把编译后的二进制放入本文仓库。

下面的块会从**当前目录中的本文文件**提取附录 A，先检查 Python 语法，再备份并安装。换机器后先切换到本文所在目录；若文档放在其他位置，修改 `webcam_doc`。

```bash
bash <<'SH'
set -euo pipefail
webcam_doc="$PWD/Ubuntu手机当摄像头.md"
webcam_stage=$(mktemp -d /tmp/phone-webcam-script.XXXXXX)
trap 'rm -r -- "$webcam_stage"' EXIT
python3 - "$webcam_doc" "$webcam_stage/phone-webcam" <<'PY'
from pathlib import Path
import sys
doc = Path(sys.argv[1]).read_text(encoding='utf-8')
section = doc.split('## 附录 A：完整脚本\n', 1)[1]
source = section.split('```python\n', 1)[1].split('\n```', 1)[0] + '\n'
compile(source, 'phone-webcam', 'exec')
Path(sys.argv[2]).write_text(source, encoding='utf-8')
PY
if test -e /opt/scrcpy/phone-webcam; then
    sudo cp -a /opt/scrcpy/phone-webcam \
        "/opt/scrcpy/phone-webcam.bak.$(date +%Y%m%d-%H%M%S)"
fi
if test -e /usr/local/bin/phone-webcam || test -L /usr/local/bin/phone-webcam; then
    if test "$(readlink /usr/local/bin/phone-webcam)" != /opt/scrcpy/phone-webcam; then
        echo 'phone-webcam 命令入口已存在且不是本文链接，请先检查。' >&2
        exit 1
    fi
fi
sudo install -m 755 "$webcam_stage/phone-webcam" /opt/scrcpy/phone-webcam
sudo ln -sfn /opt/scrcpy/phone-webcam /usr/local/bin/phone-webcam
phone-webcam --help
phone-webcam connect --help
SH
```

这里的后台运行由 `systemd-run --user` 创建临时服务实现。服务没有 `enable`，没有写入开机单元，使用 `Restart=no`。日志交给用户 journal，不持续向启动终端输出；连接检查完成后命令返回。使用 `--no-window` 完全禁用 scrcpy 窗口，`--no-terminal-title` 避免修改终端标题。

## 5. 使用命令、分辨率编号和画面方向

### 基本用法

```bash
phone-webcam --help
phone-webcam connect --help
phone-webcam resolutions
phone-webcam resolutions --camera front
phone-webcam connect
phone-webcam status
phone-webcam disconnect
```

`connect` 成功后会有类似输出：

```text
连接成功：后置摄像头，分辨率编号 20，采集 1920x1080，设置 30 FPS。
画面：竖屏正向，不镜像；输出 1080x1920。
应用中请选择 Android USB Camera（/dev/video10）。摄像头正在后台运行。
```

其中编号 20 仅是本次手机后置镜头的记录。脚本会检查设备输出尺寸，并实际读取一帧后才报告成功。相同参数重复连接显示“已经连接”，保持原采集进程；改变参数会重新建立连接。默认参数每次重新计算，**不会沿用上一次的镜像/倒置设置**。

### 选择分辨率

```bash
phone-webcam resolutions
phone-webcam connect 8
phone-webcam connect --resolution 8
phone-webcam connect -r 6 --camera front
```

编号按该镜头的采集像素数从大到小排序，去重，不包含高速采集模式。`back` 使用第一个后置镜头，`front` 使用第一个前置镜头，不代表所有后置物理镜头都合并进了一个列表。没有编号时优先选择 1920×1080；手机不支持时选择脚本规则下的较小尺寸。

| 本次手机镜头 | 声明的普通分辨率数 | 1080p 编号 | 4K 编号 | 列表最大采集尺寸 |
| --- | --- | --- | --- | --- |
| 后置 ID 0 | 32 | 20：1920×1080 | 8：3840×2160 | 4096×3072 |
| 前置 ID 1 | 18 | 6：1920×1080 | 本次列表未提供相同 4K 尺寸 | 2304×1728 |

1080p 是默认值，不是上限；后置 4K 曾收到实际画面。4096×3072 是手机声明的最大尺寸，本次没有证明它稳定可用，也没有验证所有尺寸与 30 FPS 的组合。更换手机/镜头后必须重新查编号。`30 FPS` 表示请求值，实际帧率、延迟和稳定性受手机、编码器、USB 和电脑解码性能影响。

### 选择横竖屏、倒置和镜像

| 连接命令 | 输出方向 |
| --- | --- |
| `phone-webcam connect` | 竖屏正向、不镜像 |
| `phone-webcam connect --orientation portrait` | 显式选择竖屏正向 |
| `phone-webcam connect --orientation landscape` | 横屏正向、不镜像 |
| `phone-webcam connect --upside-down` | 竖屏倒置、不镜像 |
| `phone-webcam connect --orientation landscape --upside-down` | 横屏倒置、不镜像 |
| `phone-webcam connect --mirror` | 竖屏正向、左右镜像 |
| `phone-webcam connect --camera front --mirror` | 前置、竖屏正向、左右镜像 |
| `phone-webcam connect 8 --orientation landscape --mirror` | 指定分辨率、横屏正向、左右镜像 |

`portrait` 直译为“肖像”，这里表示竖屏；`landscape` 直译为“风景”，这里表示横屏；`upside-down` 表示上下倒置；`mirror` 表示镜像。

方向固定，不随着手机自动旋转。“正向”预设适用于本次手机：原始采集旋转 90° 对应竖屏，旋转 0° 对应横屏；倒置额外旋转 180°。手机应按相应方向安装，不能把“选择竖屏”当成程序已经读取了手机当前姿态。

需要校准其他手机或安装方向时，可直接指定相对原始采集画面的**顺时针**旋转角度：

```bash
phone-webcam connect --rotate 0
phone-webcam connect --rotate 90
phone-webcam connect --rotate 180
phone-webcam connect --rotate 270 --mirror
```

`--rotate` 与 `--orientation` 二选一；可继续与 `--upside-down` 组合，后者在指定角度上再加 180°。换机验收时要看实际画面校准，脚本不会查询陀螺仪或自动判断哪边朝上。

旋转 90°/270° 会交换宽高，例如采集 1920×1080 后输出 1080×1920；分辨率编号仍指**原始采集尺寸**，旋转不会增加或减少像素总数。方形分辨率旋转后仍为方形。

实现使用 `--capture-orientation`，变换发生在手机发送的视频上，因此其他应用实际读取的画面也会改变。脚本把镜像定义为“**旋转完成后，将最终画面左右翻转**”，并转换为 scrcpy 对应的参数组合。不要用只影响客户端显示的方向参数替代。参见 [scrcpy 方向参数说明](https://github.com/Genymobile/scrcpy/blob/v5.0.1/doc/video.md#orientation)。不同手机摄像头的传感器安装方向可能不同，参考 [Android 摄像头方向说明](https://developer.android.com/media/camera/camera2/camera-preview#camera_orientation)。

BigCam 或会议软件可能另有“镜像预览”设置。要判断传给其他程序的真实左右方向，应关闭应用的额外镜像，并用文字或明显不对称的物体检查。

### 自动对焦与手动对焦

默认使用当前自动对焦方案。摄像头已连接且安装了第 4 节的定制手机端服务后，可运行：

```bash
phone-webcam focus status   # 查看模式与实际镜头位置
phone-webcam focus lock     # 将当前自动对焦位置固定住
phone-webcam focus set 120  # 手动请求对焦位置
phone-webcam focus auto     # 恢复默认自动对焦
```

`set` 接受 0–200 的百分比，0 对应远处，数值越大越靠近镜头。百分比以手机报告的对焦距离上限为 100% 计算，**不是画面的变焦倍数**。状态同时区分“设定值”和手机返回的“实际位置”。超出镜头可达范围时，即使命令被接受，也可能停在较低的实际位置。

本机后置镜头实测：自动对焦曾显示超过 100%；手动设为 120% 时，手机回报约 128.2%；设为 200% 时仍约 128.2%。这只说明这台手机当前镜头与固件的实测行为，不能保证其他镜头达到同样位置。要回到原来的自动对焦，运行 `phone-webcam focus auto`。

## 6. BigCam 选择摄像头和普通摄像头分类

BigCam 尚未安装时，先按同目录的 [Ubuntu好用的相机.md](Ubuntu好用的相机.md) 安装到 `/opt/bigcam`。线上副本也在 [GitHub 安装记录](https://github.com/did2818233534/my_docs/blob/main/Ubuntu好用的相机.md)。本文的手机接入并不依赖 BigCam，其他 V4L2 应用也可使用。

本次 BigCam 4.5.0 对应源码提交 `447bfb1eece06d311038c3692c4267300f09e753`，来源为 [BigCam 官方仓库](https://github.com/biglinux/bigcam/tree/447bfb1eece06d311038c3692c4267300f09e753)。其 V4L2 检测原本跳过名称含 loopback 等标识的设备，所以需要给 `Android USB Camera` 增加一个明确的例外。

修改文件：

```text
/opt/bigcam/share/biglinux/bigcam/core/backends/v4l2_backend.py
```

在 `_parse_devices()` 内将过滤部分改为：

```python
header = lines[0].rstrip(":")
# The manually started phone webcam is an external capture source.
# Keep filtering BigCam's own virtual outputs to avoid feedback loops.
external_phone = header.split("(")[0].strip() == "Android USB Camera"
if not external_phone and (
    "v4l2loopback" in header.lower()
    or "loopback" in header.lower()
    or "(v4l2)" in header.lower()
):
    continue
```

保留周围的设备能力检测和 `CameraInfo` 构造。本次版本的 `CameraInfo.is_virtual` 默认 `False`，这样手机进入普通 V4L2 摄像头列表；并未改内核驱动身份。不能直接取消整个 loopback 过滤，否则 BigCam 的输出可能重新作为输入造成反馈。

下面的安装块会匹配本次版本的原始代码，先备份、检查语法再写入；已经有相同补丁则不重复修改，版本不匹配则停止，需检查实际代码后调整。

```bash
sudo python3 - <<'PY'
from pathlib import Path
import shutil
import time
path = Path('/opt/bigcam/share/biglinux/bigcam/core/backends/v4l2_backend.py')
source = path.read_text(encoding='utf-8')
old = '''            # Skip v4l2loopback virtual devices and the proxy ones created by the script
            if (
                "v4l2loopback" in header.lower()
                or "loopback" in header.lower()
                or "(v4l2)" in header.lower()
            ):
                continue'''
new = '''            # The manually started phone webcam is an external capture source.
            # Keep filtering BigCam's own virtual outputs to avoid feedback loops.
            external_phone = header.split("(")[0].strip() == "Android USB Camera"
            if not external_phone and (
                "v4l2loopback" in header.lower()
                or "loopback" in header.lower()
                or "(v4l2)" in header.lower()
            ):
                continue'''
if new in source:
    print('BigCam 已有此补丁，无需重复修改。')
elif source.count(old) != 1:
    raise SystemExit('BigCam 版本或代码不匹配，请检查后再修改；文件未改变。')
else:
    updated = source.replace(old, new, 1)
    compile(updated, str(path), 'exec')
    backup = path.with_name(path.name + '.bak.' + time.strftime('%Y%m%d-%H%M%S'))
    shutil.copy2(path, backup)
    path.write_text(updated, encoding='utf-8')
    print('补丁已安装；备份：', backup)
PY
```

启动顺序：

1. `phone-webcam connect` 成功。
2. 启动/重新启动 BigCam，加载修改后的检测代码。
3. 移动鼠标使自动隐藏的工具栏出现。
4. 在顶部摄像头下拉框“选择摄像头”中选择 **Android USB Camera**；没有看到时点旁边的“刷新摄像头”。

切换横竖屏或分辨率前，先停止 BigCam 的摄像头预览，必要时关闭 BigCam。脚本发现其他应用持有设备且输出尺寸要改变时会提示并保留原连接。重连成功后重新选择/刷新摄像头，避免应用继续使用旧尺寸缓存。

## 7. 验收与日志

### 检查实际输出

```bash
phone-webcam connect
phone-webcam status
v4l2-ctl --list-devices
v4l2-ctl -d /dev/video10 --get-fmt-video
timeout 8s v4l2-ctl -d /dev/video10 \
    --stream-mmap=2 --stream-count=30 --stream-to=/dev/null
```

默认连接应输出 `1080/1920`，有限帧读取应正常结束。该命令不弹预览、不保存照片；不要留下无限采集测试进程。出现尺寸不同或读帧超时，检查手机状态、连接结果和日志。

### 检查重复连接、停止和异常保护

```bash
systemctl --user show phone-webcam.service -p MainPID --value
phone-webcam connect
systemctl --user show phone-webcam.service -p MainPID --value
phone-webcam connect 999
phone-webcam status
phone-webcam disconnect
phone-webcam status
```

相同连接的 PID 应保持不变。无效编号应有中文错误并保留原连接。`disconnect` 后没有采集进程，重复断开也应正常返回。查询分辨率/帮助不应自动启动正式连接。

再用上一节命令检查四种方向与镜像，查看实际画面中的文字/不对称物体；在有读取应用占用时验证尺寸切换提示。需要的临时读取进程必须限时并在结束后清理。

关闭启动终端后检查应用是否仍有画面。真正拔掉 USB 后检查采集服务是否停止；插回手机应保持未连接，直到手动 `connect`。新机器没有做实际拔线测试时，报告为“待验收”，不能只凭 `Restart=no` 声称实测完成。

### 查看后台日志

```bash
systemctl --user status phone-webcam.service --no-pager
journalctl --user -u phone-webcam.service -b --no-pager -n 60
```

临时服务启用了 `--collect`，停止后单元可能被回收，因此 `status` 显示未找到单元是正常的；journal 中仍可查询本次日志。正式脚本的 `phone-webcam status` 会将此情况显示为未连接。

## 8. 常见问题

### 为什么曾出现 Android 图标窗口或终端一直占着？

scrcpy 默认启动界面，直接前台运行也会占着终端。本次 5.0.1 只加 `--no-video-playback` 仍出现图标窗口，最终使用 `--no-window` 完全关闭界面；临时用户服务负责脱离终端。不要用 `--no-video`，那会关闭视频传输本身。

### 为什么自动对焦会显示 120%，手动输入 200% 却达不到 200%？

这个数字是镜头对焦距离相对于手机报告范围的百分比，不是画面放大倍数。手机的实际镜头反馈可能超过报告的 100% 上限；本机手动请求 120% 和 200% 后均回报约 128.2%。因此 `200` 是命令允许的输入上限，不代表镜头必然能达到 200%。

### 手机 Root 了，为什么仍不是物理摄像头？

原生 UVC 需要手机厂商系统和 USB gadget 支持。本文实测的固件没有成功启用该路径；Root 状态不能保证这些能力。BigCam 将手机来源显示为普通摄像头不会改变 V4L2 驱动。若某个软件要求硬件 UVC 并检查驱动，本文不能保证兼容。

### 这样标记为普通摄像头有什么影响？

本次只增加一个明确名称的检测例外，使 BigCam 能正常选择手机来源，保留其他虚拟输出过滤。影响是 BigCam 内部分类与驱动事实不完全相同；升级可能覆盖补丁，依赖“虚拟”标签的功能可能不再按原规则处理这个来源。不要给所有 loopback 设备统一设置普通摄像头标签。

### 连接成功却没有出现在应用列表？

先确认脚本成功且能读帧，再刷新/重启应用、检查应用摄像头权限或沙箱访问。BigCam 检查专用补丁。没有生产者时，`exclusive_caps=1` 的设备只暴露输出能力，部分应用不会把它列作摄像头；因此先 `connect` 再打开选择列表。

### 切换竖屏或 4K 失败？

先停止读取应用的预览，避免旧尺寸被固定。如果应用已释放但仍失败，检查日志、改用较低分辨率，并确认手机没有被其他相机应用抢占。手机声明分辨率不是保证；也不要把 4096×3072、4K 或请求 30 FPS 写成所有手机的能力上限。

### 默认“竖屏正向”却看起来倒了或横了？

确认物理安装方向，并关闭应用自己的旋转/镜像。本文预设对应本次传感器方向；用 `--rotate 0/90/180/270` 校准其他手机。输出尺寸决定画面比例，不能单独证明场景中的人物/文字已经正向。

### `connect` 报模块加载、设备权限或用户总线错误？

检查 `modinfo`、内核头文件、签名和加载日志；确认 `/dev/video10` 没被其他设备占用。普通用户需要读写设备权限，可检查 `ls -l /dev/video10`、`getfacl /dev/video10` 和当前桌面会话 ACL；必要时按发行版权限规则加入 `video` 组后重新登录，不把设备直接设为所有人可写。用户总线问题应在普通桌面会话修复，不把整条命令用 sudo 运行。

### 手机锁屏后停止，或出现黑画面？

检查手机摄像头隐私开关、USB 授权、厂商后台限制及是否有其他应用占用摄像头。不同固件的锁屏/息屏策略不同，本次没有把所有锁屏、长时运行和省电组合都验收为支持。

本次测试间隙也出现过手机端 `Camera disconnected`，但 ADB 仍能看到手机。服务按设定停止，没有自动重启；重新手动连接可以收到画面。仅凭这些日志不能确定是锁屏、其他应用抢占还是厂商策略，因此换机验收应包含持续采集，不能只看启动时读到一帧就认定长期稳定。

### 中文帮助为什么曾错行、没对齐？

原来的 argparse 子命令布局把较长命令的说明放到下一行；手工写空格也使长示例的说明错位。最终脚本按 Unicode 终端显示宽度统一计算列宽，中文宽字符按两列计算，命令说明同列对齐。使用正常支持中文的等宽终端字体；窄于 80 列的窗口仍可能被终端折行。

## 9. 回退与清理

先断开正式采集，再恢复指定文件的备份：

```bash
phone-webcam disconnect
```

本机保留了增加方向、对焦、扩展手动对焦范围前的脚本备份；换机部署块会生成带时间戳的备份。需要恢复时先确认备份内容，再将其安装回 `/opt/scrcpy/phone-webcam`。定制手机端服务也应先备份后安装，恢复时须与脚本版本配套。BigCam 应恢复实际记录的对应备份，重启应用加载。

如果要移除本文 CLI，确认命令入口仍指向本脚本后删除：

```bash
readlink /usr/local/bin/phone-webcam
sudo rm -- /usr/local/bin/phone-webcam
sudo rm -- /opt/scrcpy/phone-webcam
```

scrcpy、ADB 或 `v4l2loopback` 可能被其他用途使用，不因为移除 CLI 就卸载共享软件或内核模块。只有确认本次独占创建、所有使用者都已释放时才可 `sudo modprobe -r v4l2loopback`；否则保留已加载模块。没有永久启动服务需要禁用，也不用删除手机端系统应用。

## 本次验证结果与待验收事项

| 项目 | 本次情况 |
| --- | --- |
| 后置/前置 1080p、按编号连接 | 收到实际画面；错误编号未破坏原连接 |
| 后置 4K | 曾收到 3840×2160 画面；未证明最高声明分辨率或所有组合稳定 |
| 默认竖屏 | 实际输出 1080×1920，连接成功并读到帧 |
| 四种方向 × 两种镜像状态 | 逐一连接并读取实际帧；状态和输出宽高符合参数 |
| 镜像顺序 | 用不对称像素矩阵检查 8 种参数映射，符合最终画面左右镜像的定义 |
| 相同连接重复调用、冲突参数 | 同参数保留原进程，错误参数不破坏原连接 |
| 读取应用占用时切换横竖屏 | 限时读取进程占用设备时正确拒绝尺寸切换，保留原 PID；测试进程已清理 |
| BigCam 检测及读帧 | 已识别手机为普通 V4L2 来源；后端收到实际帧 |
| 帮助排版 | 按显示列宽对齐；全局和连接帮助正常内容在 80 列内 |
| 无 GUI、手动后台服务 | 配置与运行进程已检查，使用 `--no-window`，无自动启用的单元 |
| 对焦控制 | 定制手机端服务与脚本已实测；默认自动、手动设定、实际位置反馈和恢复自动可用；输入 120%、200% 后实际约 128.2% |
| 新机器实际拔线/关闭终端、长期使用 | 换机后仍需按第 7 节实际验收，不能用本机结果替代 |
| 最高分辨率、锁屏省电、所有应用兼容性 | 未全部实测，不作保证 |

附录源码与当前安装脚本保持一致。对焦功能还须安装同仓库补丁构建的手机端服务；再次部署后应以新机器实际检查为准。

## 参考资料

- [scrcpy 官方项目](https://github.com/Genymobile/scrcpy)
- [scrcpy 5.0.1 固定发布及 Linux 静态包](https://github.com/Genymobile/scrcpy/releases/tag/v5.0.1)
- [scrcpy V4L2 输出说明](https://github.com/Genymobile/scrcpy/blob/v5.0.1/doc/v4l2.md)
- [v4l2loopback 官方项目](https://github.com/v4l2loopback/v4l2loopback)
- [systemd-run 临时服务说明](https://www.freedesktop.org/software/systemd/man/latest/systemd-run.html)
- [Android Camera2 `LENS_FOCUS_DISTANCE` 说明](https://developer.android.com/reference/android/hardware/camera2/CaptureRequest#LENS_FOCUS_DISTANCE)

## 附录 A：完整脚本

将下面完整代码保存为 `/opt/scrcpy/phone-webcam`。默认安装路径、设备编号和应用名称已集中在常量与加载位置；如确需改设备编号，要同时改 `VIDEO`、sysfs 路径及 `modprobe video_nr`。

```python
#!/usr/bin/python3
"""手动启动、查询和停止手机 USB 摄像头。"""

import argparse
import math
import os
from pathlib import Path
import re
import secrets
import subprocess
import sys
import time
import unicodedata

ADB = "/opt/scrcpy/adb"
SCRCPY = "/opt/scrcpy/scrcpy"
UNIT = "phone-webcam.service"
VIDEO = "/dev/video10"
LABEL = "Android USB Camera"
FACING = {"back": "后置", "front": "前置"}
ORIENTATION = {0: "横屏正向", 90: "竖屏正向", 180: "横屏倒置", 270: "竖屏倒置"}


def display_width(text):
    return sum(0 if unicodedata.combining(char) else
               2 if unicodedata.east_asian_width(char) in ("W", "F") else 1
               for char in text)


def help_columns(rows):
    width = max(display_width(label) for label, _ in rows)
    return "\n".join("  " + label + " " * (width - display_width(label) + 2) + description
                     for label, description in rows)


class CameraError(Exception):
    pass


class ChineseParser(argparse.ArgumentParser):
    def __init__(self, *args, **kwargs):
        kwargs.setdefault("add_help", False)
        super().__init__(*args, **kwargs)
        self._positionals.title = "参数"
        self._optionals.title = "选项"
        self.add_argument("-h", "--help", action="help", help="显示中文帮助")

    def format_usage(self):
        return super().format_usage().replace("usage:", "用法：", 1)

    def format_help(self):
        sections = [self.format_usage().rstrip()]
        if self.description:
            sections.append(self.description)
        formatter = self._get_formatter()
        for group in self._action_groups:
            rows = []
            for action in group._group_actions:
                if isinstance(action, argparse._SubParsersAction):
                    rows.extend((formatter._format_action_invocation(command), command.help)
                                for command in action._get_subactions())
                elif action.help is not argparse.SUPPRESS:
                    rows.append((formatter._format_action_invocation(action),
                                 formatter._expand_help(action)))
            if rows:
                sections.append(group.title + "：\n" + help_columns(rows))
        if self.epilog:
            sections.append(self.epilog)
        return "\n\n".join(sections) + "\n"

    def error(self, message):
        self.print_usage(sys.stderr)
        self.exit(2, "参数错误：请运行 phone-webcam --help 查看中文用法。\n")


def run(args, timeout=8):
    try:
        return subprocess.run(args, capture_output=True, text=True, timeout=timeout)
    except subprocess.TimeoutExpired:
        raise CameraError("操作超时，请检查手机连接后重试。") from None
    except OSError:
        raise CameraError("无法运行所需程序：" + args[0]) from None


def active():
    return run(["systemctl", "--user", "--quiet", "is-active", UNIT]).returncode == 0


def current_config():
    result = run(["systemctl", "--user", "show", UNIT, "-p", "MainPID", "--value"])
    try:
        pid = int(result.stdout.strip())
        args = Path(f"/proc/{pid}/cmdline").read_bytes().decode().split("\0")
    except (OSError, ValueError):
        return 0, {}
    options = dict(arg[2:].split("=", 1) for arg in args if arg.startswith("--") and "=" in arg)
    return pid, options


def parse_catalogue(output, facing):
    cameras = []
    camera = None
    high_speed = False
    for line in output.splitlines():
        header = re.search(r"--camera-id=(\S+)\s+\((back|front|external),", line)
        if header:
            camera = {"id": header[1], "facing": header[2], "sizes": set()}
            cameras.append(camera)
            high_speed = False
        elif "High speed capture" in line:
            high_speed = True
        elif camera is not None and not high_speed:
            size = re.fullmatch(r"\s*-\s+(\d+)x(\d+)\s*", line)
            if size:
                camera["sizes"].add((int(size[1]), int(size[2])))
    camera = next((c for c in cameras if c["facing"] == facing), None)
    if camera is None or not camera["sizes"]:
        raise CameraError("没有查询到" + FACING[facing] + "摄像头的可用分辨率。")
    sizes = sorted(camera["sizes"], key=lambda s: (s[0] * s[1], s[0], s[1]), reverse=True)
    return camera["id"], sizes


def catalogue(facing):
    serial = run([ADB, "-d", "get-serialno"]).stdout.strip()
    if not serial or serial == "unknown" or run([ADB, "-s", serial, "get-state"]).stdout.strip() != "device":
        raise CameraError("未找到已授权的 USB 手机，请检查数据线并允许 USB 调试。")
    result = run([SCRCPY, "-s", serial, "--video-source=camera", "--list-camera-sizes",
                  "--no-audio", "--no-terminal-title"], timeout=15)
    if result.returncode != 0:
        raise CameraError("查询分辨率失败，请检查手机的 USB 调试连接。")
    camera_id, sizes = parse_catalogue(result.stdout + "\n" + result.stderr, facing)
    model = run([ADB, "-s", serial, "shell", "getprop", "ro.product.marketname"]).stdout.strip() or serial
    return serial, model, camera_id, sizes


def show_resolutions(facing):
    _, model, camera_id, sizes = catalogue(facing)
    print(f"手机：{model}；镜头：{FACING[facing]}（ID {camera_id}）")
    print("编号  分辨率        宽高比")
    for index, (width, height) in enumerate(sizes, 1):
        divisor = math.gcd(width, height)
        default = "（默认）" if (width, height) == (1920, 1080) else ""
        print(f"{index:>4}  {width}x{height:<6}  {width // divisor}:{height // divisor}{default}")
    print("\n使用编号连接：phone-webcam connect 编号" + (" --camera front" if facing == "front" else ""))
    print("编号按像素数从大到小排列；更换手机或镜头后，请重新查询。")
    print("以上是手机声明支持的普通采集分辨率，实际能否使用以连接结果为准。")


def dimensions():
    result = run(["v4l2-ctl", "-d", VIDEO, "--get-fmt-video"])
    match = re.search(r"Width/Height\s*:\s*(\d+)/(\d+)", result.stdout)
    return (int(match[1]), int(match[2])) if match else None


def capture_transform(rotation, mirror):
    # scrcpy 先镜像再旋转；反转角度，使 --mirror 始终镜像最终画面的左右。
    return f"flip{(-rotation) % 360}" if mirror else str(rotation)


def read_transform(options):
    value = options.get("capture-orientation", "0")
    mirror = value.startswith("flip")
    angle = int(value[4:] if mirror else value)
    return (-angle) % 360 if mirror else angle, mirror


def transform_description(rotation, mirror):
    return f"{ORIENTATION[rotation]}，{'左右镜像' if mirror else '不镜像'}"


def device_users(exclude_pid):
    users = []
    for process in Path("/proc").iterdir():
        if not process.name.isdigit() or int(process.name) == exclude_pid:
            continue
        try:
            if any(os.readlink(fd) == VIDEO for fd in (process / "fd").iterdir()):
                command = (process / "cmdline").read_bytes().decode().split("\0")
                users.append("BigCam" if any("/bigcam/" in arg for arg in command) else Path(command[0]).name)
        except (OSError, UnicodeError):
            continue
    return sorted(set(users))


def stop():
    if active():
        if run(["systemctl", "--user", "stop", UNIT], timeout=12).returncode != 0:
            raise CameraError("断开失败，请重试。")


def focus_state(serial, camera_id):
    path = f"/data/local/tmp/scrcpy-focus-state-{camera_id}"
    result = run([ADB, "-s", serial, "shell", "cat", path], timeout=4)
    if result.returncode != 0:
        raise CameraError("读取对焦状态失败，请确认摄像头仍在运行。")
    fields = result.stdout.strip().split()
    if len(fields) < 3 or fields[1] not in ("auto", "manual") or (fields[1] == "manual" and len(fields) < 4):
        raise CameraError("手机端尚未提供对焦控制，请重新连接摄像头。")
    return fields


def focus_actual_percent(fields, maximum):
    actual = next((field[7:] for field in fields if field.startswith("actual=")), None)
    if actual is None or maximum <= 0:
        return None
    try:
        return 100 * float(actual) / maximum
    except ValueError:
        return None


def focus(mode, percent=None):
    if not active():
        raise CameraError("摄像头尚未连接，请先运行 phone-webcam connect。")
    _, options = current_config()
    serial = options.get("serial", "")
    camera_id = options.get("camera-id", "")
    if not serial or not re.fullmatch(r"[a-zA-Z0-9_-]+", camera_id):
        raise CameraError("无法识别当前手机或摄像头。")
    fields = focus_state(serial, camera_id)
    if mode == "status":
        maximum = float(fields[2] if fields[1] == "auto" else fields[3])
        actual = focus_actual_percent(fields, maximum)
        if fields[1] == "auto":
            print(f"对焦：自动，当前镜头位置约 {actual:.1f}%。" if actual is not None else "对焦：自动。")
        else:
            requested = 100 * float(fields[2]) / maximum if maximum > 0 else 0
            detail = f"；镜头实际约 {actual:.1f}%" if actual is not None else ""
            print(f"对焦：固定，设定 {requested:.1f}%{detail}（0 为远处，数值越大越近）。")
        return
    nonce = secrets.token_hex(6)
    value = f" {percent:g}" if percent is not None else ""
    command = f"{nonce} {mode}{value}"
    path = f"/data/local/tmp/scrcpy-focus-command-{camera_id}"
    result = run([ADB, "-s", serial, "shell", f"printf '%s\\n' '{command}' > {path}"], timeout=4)
    if result.returncode != 0:
        raise CameraError("发送对焦命令失败，请检查 USB 连接。")
    deadline = time.monotonic() + 4
    while time.monotonic() < deadline:
        result = run([ADB, "-s", serial, "shell", "cat", f"/data/local/tmp/scrcpy-focus-state-{camera_id}"], timeout=2)
        reply = result.stdout.strip().split()
        if reply and reply[0] == nonce:
            if len(reply) > 1 and reply[1] == "error":
                reason = reply[2] if len(reply) > 2 else "unknown"
                messages = {"fixed-lens": "此镜头没有可调焦马达。",
                            "no-focus-result": "尚未取得当前自动对焦位置，请稍等再试。",
                            "camera-rejected": "摄像头拒绝了对焦设置。"}
                raise CameraError(messages.get(reason, "对焦命令无效：" + reason))
            time.sleep(0.7)
            focus("status")
            return
        time.sleep(0.15)
    raise CameraError("对焦命令超时，请检查手机端摄像头是否仍在运行。")


def ready(expected):
    deadline = time.monotonic() + 12
    while time.monotonic() < deadline and active():
        if dimensions() == expected:
            try:
                probe = run(["v4l2-ctl", "-d", VIDEO, "--stream-mmap=2",
                             "--stream-count=1", "--stream-to=/dev/null"], timeout=2)
                if probe.returncode == 0 and active():
                    return True
            except CameraError:
                pass
        time.sleep(0.15)
    return False


def connect(facing, requested_index, rotation, mirror):
    serial, _, camera_id, sizes = catalogue(facing)
    if requested_index is None:
        size = (1920, 1080) if (1920, 1080) in sizes else next(
            (s for s in sizes if s[0] <= 1920 and s[1] <= 1080), sizes[-1])
        index = sizes.index(size) + 1
    else:
        index = requested_index
        if index < 1 or index > len(sizes):
            raise CameraError(f"分辨率编号无效，应为 1 至 {len(sizes)}。请先运行 phone-webcam resolutions。")
        size = sizes[index - 1]
    width, height = size
    output_size = (height, width) if rotation in (90, 270) else size
    transform = capture_transform(rotation, mirror)
    was_active = active()
    pid, options = current_config()
    same_camera = options.get("camera-id") == camera_id or options.get("camera-facing") == facing
    same_phone = options.get("serial") in (None, serial)
    same_connection = (was_active and same_camera and same_phone
                       and options.get("camera-size") == f"{width}x{height}"
                       and options.get("capture-orientation", "0") == transform)

    if not same_connection:
        if was_active and Path(VIDEO).exists() and dimensions() != output_size:
            users = device_users(pid)
            if users:
                raise CameraError("切换分辨率或横竖屏前，请先停止这些应用的摄像头预览：" + "、".join(users) + "。原连接已保留。")
        if not Path(VIDEO).exists():
            result = subprocess.run(["sudo", "modprobe", "v4l2loopback", "devices=1", "video_nr=10",
                                     f"card_label={LABEL}", "exclusive_caps=1"])
            if result.returncode != 0:
                raise CameraError("加载摄像头驱动失败。")
            run(["udevadm", "settle", "--timeout=3"])
        try:
            name = Path("/sys/class/video4linux/video10/name").read_text().strip()
        except OSError:
            name = ""
        if name != LABEL:
            raise CameraError("/dev/video10 不可用或已被其他设备占用。")
        stop()
        result = run(["systemd-run", "--user", f"--unit={UNIT}", "--collect", "--quiet", "--service-type=exec",
                      "--property=Restart=no", "--property=KillSignal=SIGINT", "--property=TimeoutStopSec=8",
                      f"--setenv=ADB={ADB}", SCRCPY, f"--serial={serial}", "--video-source=camera",
                      f"--camera-id={camera_id}", f"--camera-size={width}x{height}", "--camera-fps=30",
                      f"--capture-orientation={transform}",
                      "--no-audio", f"--v4l2-sink={VIDEO}", "--no-window", "--no-terminal-title"])
        if result.returncode != 0:
            raise CameraError("启动后台摄像头失败，请重试。")
    if not ready(output_size):
        if not same_connection:
            stop()
        raise CameraError("连接失败：未收到摄像头画面。请检查手机，或选择其他分辨率编号。")
    prefix = "已经连接" if same_connection else "连接成功"
    print(f"{prefix}：{FACING[facing]}摄像头，分辨率编号 {index}，采集 {width}x{height}，设置 30 FPS。")
    print(f"画面：{transform_description(rotation, mirror)}；输出 {output_size[0]}x{output_size[1]}。")
    print(f"应用中请选择 {LABEL}（{VIDEO}）。摄像头正在后台运行。")


def make_parser():
    parser = ChineseParser(prog="phone-webcam", description="通过 USB 把手机用作电脑摄像头，手动连接后在后台运行。",
                           formatter_class=argparse.RawDescriptionHelpFormatter,
                           epilog="示例：\n" + help_columns([
                               ("phone-webcam resolutions", "查询后置摄像头的分辨率编号"),
                               ("phone-webcam resolutions --camera front", "查询前置摄像头的分辨率编号"),
                               ("phone-webcam connect", "默认竖屏正向、不镜像，优先 1080p"),
                               ("phone-webcam connect 8", "使用后置镜头的 8 号分辨率"),
                               ("phone-webcam connect -r 8 --camera front", "使用前置镜头的 8 号分辨率"),
                               ("phone-webcam connect --orientation landscape", "横屏正向"),
                               ("phone-webcam connect --upside-down", "竖屏倒置"),
                               ("phone-webcam connect --mirror", "竖屏正向、左右镜像"),
                               ("phone-webcam status", "查询当前连接状态"),
                               ("phone-webcam focus status", "查看当前对焦模式"),
                               ("phone-webcam focus lock", "锁定当前自动对焦位置"),
                               ("phone-webcam focus set 120", "手动设置对焦位置，可输入 0 到 200"),
                               ("phone-webcam focus auto", "恢复自动对焦"),
                               ("phone-webcam disconnect", "断开摄像头"),
                           ]) + "\n\n" + """连接会检查实际画面并报告结果；日志保留在后台，关闭终端不会断开。
不添加自动启动；拔掉 USB 后停止。切换编号会重新连接。
默认竖屏正向、不镜像；方向固定，不跟随手机自动旋转。
分辨率列表显示采集尺寸；竖屏输出交换宽高，不包含高速采集模式。""")
    commands = parser.add_subparsers(dest="command", required=True, metavar="命令", title="命令")
    resolutions = commands.add_parser("resolutions", help="查询支持的分辨率并显示编号", description="查询手机声明支持的普通采集分辨率。")
    connection = commands.add_parser("connect", help="连接摄像头，并报告是否成功", description="默认竖屏正向、不镜像；不传编号时优先采集 1920x1080。",
                                     epilog="方向固定：横屏正向旋转 0°，竖屏正向旋转 90°；倒置再旋转 180°。\n镜像作用于旋转后画面的左右；换手机后方向异常可用 --rotate 校准。\n切换横竖屏前，请先停止应用的摄像头预览。")
    for command in (resolutions, connection):
        command.add_argument("-c", "--camera", choices=["back", "front"], default=None, metavar="back|front",
                             help="back：后置（默认）；front：前置")
    connection.add_argument("index", nargs="?", metavar="编号", help="分辨率列表中的编号（也兼容原来的 back/front 参数）")
    connection.add_argument("-r", "--resolution", type=int, metavar="编号", help="通过选项指定分辨率编号")
    connection.add_argument("--orientation", choices=["portrait", "landscape"], metavar="方向",
                            help="portrait：竖屏（默认）；landscape：横屏")
    connection.add_argument("--upside-down", action="store_true", help="在所选方向上倒置 180°（默认正向）")
    connection.add_argument("--mirror", action="store_true", help="将最终画面左右镜像（默认不镜像）")
    connection.add_argument("--rotate", type=int, choices=[0, 90, 180, 270], metavar="角度",
                            help="顺时针旋转；与 --orientation 二选一")
    commands.add_parser("disconnect", help="断开手机摄像头")
    commands.add_parser("status", help="查询当前连接状态和分辨率")
    focus_parser = commands.add_parser("focus", help="查看或调整对焦，默认保持自动对焦")
    focus_commands = focus_parser.add_subparsers(dest="focus_command", required=True, metavar="操作")
    focus_commands.add_parser("status", help="查看当前对焦模式")
    focus_commands.add_parser("lock", help="锁定当前自动对焦位置")
    focus_commands.add_parser("auto", help="恢复自动对焦")
    focus_set = focus_commands.add_parser("set", help="手动设置对焦位置，0 为远处；实际位置以手机反馈为准")
    focus_set.add_argument("position", type=float, metavar="0-200", help="焦点位置百分比")
    return parser


def main():
    parser = make_parser()
    if len(sys.argv) == 1:
        parser.print_help()
        return 0
    args = parser.parse_args()
    if args.command == "disconnect":
        running = active()
        stop()
        print("已断开手机摄像头。" if running else "手机摄像头当前未连接。")
    elif args.command == "status":
        if active():
            _, options = current_config()
            print("已连接：手机摄像头正在后台运行。")
            output_size = dimensions()
            output_text = f"{output_size[0]}x{output_size[1]}" if output_size else "未知"
            rotation, mirror = read_transform(options)
            print(f"采集分辨率：{options.get('camera-size', '未知')}；输出：{output_text}；设置 {options.get('camera-fps', '30')} FPS。")
            print(f"画面：{transform_description(rotation, mirror)}（顺时针旋转 {rotation}°）。")
            print(f"摄像头名称：{LABEL}；设备：{VIDEO}。")
        else:
            print("手机摄像头当前未连接。")
    elif args.command == "focus":
        if args.focus_command == "set" and (not math.isfinite(args.position) or not 0 <= args.position <= 200):
            raise CameraError("对焦位置应为 0 到 200；0 是远处，超出镜头能力时会停在实际可达位置。")
        focus("manual" if args.focus_command == "set" else args.focus_command,
              args.position if args.focus_command == "set" else None)
    elif args.command == "resolutions":
        show_resolutions(args.camera or "back")
    else:
        if args.rotate is not None and args.orientation is not None:
            raise CameraError("--rotate 和 --orientation 请只使用一种。")
        rotation = args.rotate if args.rotate is not None else (0 if args.orientation == "landscape" else 90)
        if args.upside_down:
            rotation = (rotation + 180) % 360
        facing = args.camera or "back"
        index = args.resolution
        if args.index in FACING:
            if args.camera and args.camera != args.index:
                raise CameraError("前后置镜头参数冲突，请只指定一个镜头。")
            facing = args.index
        elif args.index is not None:
            if index is not None:
                raise CameraError("分辨率编号重复，请只使用位置参数或 --resolution 中的一种。")
            try:
                index = int(args.index)
            except ValueError:
                raise CameraError("分辨率编号必须是整数，请先运行 phone-webcam resolutions。") from None
        connect(facing, index, rotation, args.mirror)
    return 0


if __name__ == "__main__":
    try:
        sys.exit(main())
    except CameraError as error:
        print("错误：" + str(error), file=sys.stderr)
        sys.exit(1)
```
