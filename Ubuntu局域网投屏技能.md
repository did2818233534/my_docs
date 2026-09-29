# Ubuntu 局域网低延迟单窗口投屏与输入回传

这是一套适用于 Ubuntu 图形程序的可复现方案。远端每个程序运行在独立的 X11 虚拟屏幕；发送端只采集该程序窗口，编码为 H.264，经 RTP/UDP 单向发送；本地以普通窗口显示。鼠标和键盘走独立 SSH 连接回传。可以为多个程序各开一套虚拟屏幕、视频端口和本地窗口。

本文以 **Ubuntu 24.04、两台机器处于同一可信局域网、两个远端窗口** 为例。所有大写变量和示例显示号、端口都可以按现场修改。三个完整脚本附在文末，不依赖特定业务软件。

## 1. 数据流和低延迟设计

```text
远端 Xvfb :90 ─ 图形程序 A ─ ximagesrc ─ H.264 ─ RTP/UDP:50410 ─ 本地窗口 A
远端 Xvfb :91 ─ 图形程序 B ─ ximagesrc ─ H.264 ─ RTP/UDP:50412 ─ 本地窗口 B
本地窗口 A/B ─ 鼠标键盘事件 ─ SSH 标准输入 ─ XTEST ─ 各自的远端 X 窗口
```

视频通道以当前画面为准：发送端编码前的队列只容纳一帧，接收端的抖动缓冲有时间上限，解码后的 GStreamer `appsink` 只保留一帧。丢失的 UDP 包不会等重传，所以网络拥塞时可能出现短暂花屏或跳帧；下一个关键帧会恢复画面。输入通道则保留按键、按下和释放事件的顺序，只合并尚未发送的连续鼠标移动事件。

本方案中 **视频 RTP/UDP 不加密**，输入 SSH 加密。仅在可信局域网或已经建立的受保护网络内使用视频端口，不要直接向公网开放。若远端软件承担设备控制，投屏界面不能代替设备现场的物理急停。

## 2. 准备机器和参数

在 **本地机器** 的终端设置变量；`LOCAL_IP` 必须是远端可以直接访问的本机局域网地址。`REMOTE_HOST` 可以是 IP 或可解析主机名。

```bash
REMOTE_USER='远端用户名'
REMOTE_HOST='远端主机名或局域网IP'
REMOTE="${REMOTE_USER}@${REMOTE_HOST}"
LOCAL_IP='本机可由远端访问的局域网IP'

ip -4 addr show
ssh "$REMOTE" 'hostname; id'
```

查看本机到远端实际使用的网卡与源地址，可运行 `ip -4 route get 远端IP`。确认远端能向本机 UDP 50410、50412 发送数据；如果本机启用了 UFW，仅放行来自远端的这两个端口，例如：

```bash
sudo ufw allow from 远端IP to any port 50410 proto udp
sudo ufw allow from 远端IP to any port 50412 proto udp
```

下面的接收程序使用 `ssh -o BatchMode=yes` 建立两个持久连接，所以应先配置 SSH 公钥登录并验证：

```bash
ssh-keygen -t ed25519   # 已有合适的密钥可跳过
ssh-copy-id "$REMOTE"
ssh -o BatchMode=yes "$REMOTE" 'printf "SSH OK\n"'
```

## 3. 安装依赖

在 **远端机器** 执行：

```bash
sudo apt update
sudo apt install -y \
  openssh-server xvfb x11-utils x11-xserver-utils xdotool x11-apps \
  python3 python3-xlib \
  gstreamer1.0-tools gstreamer1.0-plugins-base \
  gstreamer1.0-plugins-good gstreamer1.0-plugins-ugly
```

在 **本地机器** 执行：

```bash
sudo apt update
sudo apt install -y \
  openssh-client python3 python3-pyqt5 python3-gi python3-gst-1.0 \
  gstreamer1.0-tools gstreamer1.0-plugins-base \
  gstreamer1.0-plugins-good gstreamer1.0-libav
```

确认关键 GStreamer 元件存在。远端运行：

```bash
for element in ximagesrc x264enc rtph264pay udpsink; do
  gst-inspect-1.0 "$element" >/dev/null || echo "缺少: $element"
done
```

本地运行：

```bash
for element in udpsrc rtpjitterbuffer rtph264depay h264parse avdec_h264 appsink; do
  gst-inspect-1.0 "$element" >/dev/null || echo "缺少: $element"
done
python3 -c 'import gi; from PyQt5 import QtWidgets; print("Python GUI OK")'
```

`x264enc` 通常由 `gstreamer1.0-plugins-ugly` 提供，`avdec_h264` 由 `gstreamer1.0-libav` 提供。本文选择系统 Python `/usr/bin/python3`，避免虚拟环境找不到 Ubuntu 的 `gi`、GStreamer 或 PyQt5 模块。

## 4. 放置脚本

在两台机器分别创建安装目录。使用 `/opt`，不把程序安装进用户主目录。

```bash
sudo install -d -o "$USER" -g "$(id -gn)" /opt/lan-lowlatency
```

将文末 **附录 A** 保存到远端 `/opt/lan-lowlatency/capture_supervisor.py`，**附录 B** 保存到远端 `/opt/lan-lowlatency/input_relay.py`，**附录 C** 保存到本地 `/opt/lan-lowlatency/viewer.py`。保存后分别运行：

```bash
chmod 755 /opt/lan-lowlatency/*.py
python3 -m py_compile /opt/lan-lowlatency/*.py
```

远端只会有附录 A/B，本地只会有附录 C；相应命令仅检查当前机器上已保存的脚本。

## 5. 远端启动独立虚拟屏幕与程序

先检查 `:90`、`:91` 没被占用；如果被占用，换成其他未使用的显示号，并在后续命令中一致修改。

```bash
test ! -e /tmp/.X90-lock && echo ':90 可用'
test ! -e /tmp/.X91-lock && echo ':91 可用'
```

在 **远端机器** 启动两个 1920×1080、24 位色的 Xvfb。这里使用用户级 transient systemd 服务，SSH 断开后虚拟屏幕仍在；重启机器后须重新执行这些启动命令。

若远端没有持续的桌面登录，先查 `loginctl show-user "$USER" -p Linger`。结果为 `Linger=no` 时，最后一个登录会话退出后用户级服务也可能停止；需要断开 SSH 后继续运行的机器可由管理员执行 `sudo loginctl enable-linger "$USER"`。这只影响服务存活，不影响视频协议。

```bash
systemd-run --user --unit=lan-screen-a -- \
  /usr/bin/Xvfb :90 -screen 0 1920x1080x24 \
  +extension RANDR +extension RENDER +extension GLX -nolisten tcp -noreset

systemd-run --user --unit=lan-screen-b -- \
  /usr/bin/Xvfb :91 -screen 0 1920x1080x24 \
  +extension RANDR +extension RENDER +extension GLX -nolisten tcp -noreset

DISPLAY=:90 xdpyinfo | grep dimensions
DISPLAY=:91 xdpyinfo | grep dimensions
DISPLAY=:90 xdpyinfo -queryExtensions | grep XTEST
```

先用小程序检查窗口采集链路（测试完成后停止它们）：

```bash
systemd-run --user --unit=lan-test-a -- env DISPLAY=:90 /usr/bin/xclock
systemd-run --user --unit=lan-test-b -- env DISPLAY=:91 /usr/bin/xclock
DISPLAY=:90 xwininfo -root -tree | head -20
DISPLAY=:91 xwininfo -root -tree | head -20
systemctl --user stop lan-test-a lan-test-b
```

实际使用时，为每个图形程序各写一个启动脚本，设置对应 `DISPLAY` 后用 `exec` 启动。例如 `/opt/lan-lowlatency/start-app-a.sh`：

```bash
#!/usr/bin/env bash
set -eo pipefail
export DISPLAY=:90
export QT_QPA_PLATFORM=xcb  # Qt 程序需要 X11 后端时保留；非 Qt 程序可删除
exec /绝对路径/图形程序 参数1 参数2
```

第二个脚本把 `DISPLAY` 改成 `:91`。需要 ROS 等环境的程序，在 `exec` 之前加载相应环境；如要设置进程专用变量，也写在各自脚本中。启动：

```bash
chmod 755 /opt/lan-lowlatency/start-app-a.sh /opt/lan-lowlatency/start-app-b.sh
systemd-run --user --unit=lan-app-a -- /opt/lan-lowlatency/start-app-a.sh
systemd-run --user --unit=lan-app-b -- /opt/lan-lowlatency/start-app-b.sh
systemctl --user --no-pager status lan-app-a lan-app-b
```

获取每个程序**最外层、真正有画面的窗口 ID**。窗口 ID 会随程序重启变化，不能把旧 ID 永久写死。

```bash
DISPLAY=:90 xwininfo -root -tree | head -40
DISPLAY=:91 xwininfo -root -tree | head -40
```

从输出中抄下对应窗口的 `0x...` ID。将程序窗口调整到虚拟屏幕尺寸；程序若有自己的窗口参数，也可从启动参数直接设为 1920×1080。

```bash
XID_A='0x从第一块屏幕查询得到的窗口ID'
XID_B='0x从第二块屏幕查询得到的窗口ID'
DISPLAY=:90 xdotool windowsize "$XID_A" 1920 1080
DISPLAY=:90 xdotool windowmove "$XID_A" 0 0
DISPLAY=:91 xdotool windowsize "$XID_B" 1920 1080
DISPLAY=:91 xdotool windowmove "$XID_B" 0 0
DISPLAY=:90 xwininfo -id "$XID_A" | grep -E 'Width:|Height:|Absolute upper-left'
DISPLAY=:91 xwininfo -id "$XID_B" | grep -E 'Width:|Height:|Absolute upper-left'
```

以上 `XID_A`、`XID_B` 还须记录在本地的启动命令中；它们不随 SSH 自动传到本地。

## 6. 本地启动接收窗口

在 **本地机器**，重新设置第 2 节的 `REMOTE`、`LOCAL_IP`，并填入刚查到的两个 XID。两个命令在两个终端分别运行。`--width`/`--height` 是本地窗口初始大小，源分辨率仍为远端的 1920×1080；本地可自由拖动、缩放或最大化，画面按比例适配。

```bash
XID_A='0x第一块屏幕的窗口ID'
XID_B='0x第二块屏幕的窗口ID'

QT_QPA_PLATFORM=xcb /usr/bin/python3 /opt/lan-lowlatency/viewer.py \
  --name app-a --title '窗口 A' --remote "$REMOTE" --local-ip "$LOCAL_IP" \
  --display :90 --xid "$XID_A" --port 50410 \
  --fps 12 --bitrate 8000 --width 1360 --height 850
```

```bash
QT_QPA_PLATFORM=xcb /usr/bin/python3 /opt/lan-lowlatency/viewer.py \
  --name app-b --title '窗口 B' --remote "$REMOTE" --local-ip "$LOCAL_IP" \
  --display :91 --xid "$XID_B" --port 50412 \
  --fps 12 --bitrate 8000 --width 1360 --height 850
```

视频 RTP/UDP 和输入 SSH 各有独立连接。一个远端程序退出时，其接收窗口会提示“视频已退出”或显示较旧画面；重新启动程序后要重新查 XID，再重启对应接收窗口。接收窗口关闭时，它会关闭 SSH 连接；远端 `capture_supervisor.py` 检测到连接结束便停止对应的 GStreamer 发送进程。

## 7. 验证与故障排查

1. 本地窗口标题应显示源分辨率、接收帧率和“画面更新”年龄。分别移动两个远端窗口中的内容，确认两个本地窗口持续变化；再测试点击、拖动、键盘和滚轮。
2. 查远端窗口：`DISPLAY=:90 xwininfo -id "$XID_A"`。如果报错或只有黑画面，重新查窗口 ID，并确认图形程序真的运行在 `:90`；`:91` 同理。
3. 查本地 UDP：`ss -lun | grep -E '50410|50412'`。窗口在运行而没有画面时，检查 `LOCAL_IP` 是否是远端可达的地址、端口是否与本地一致、防火墙是否放行。
4. 查日志：本地 `/tmp/lan-lowlatency/app-a-video.log`、`app-a-input.log`；第二窗口对应 `app-b-*.log`。远端 `journalctl --user -u lan-app-a -b` 查看程序启动错误。
5. 检查 SSH：`ssh -o BatchMode=yes "$REMOTE" true`。视频正常但输入无效时，检查 `python3-xlib`、XTEST 扩展和 XID；如果远端启动程序使用不同用户，X11 访问权限也必须允许输入回传进程连接同一显示。
6. 如果画面清楚但动作后出现明显滞后，先检查远端 CPU、网络丢包和窗口标题里的画面年龄；不应通过增大接收缓冲“修复”。降低 `--fps` 或源分辨率可减小持续负荷；在网络有余量且画面文字模糊时提高 `--bitrate`。

本例为 **每路 1920×1080、12 fps、8000 kbit/s**。`--bitrate` 单位是 kbit/s，两个窗口的额定视频码率合计约 16 Mbit/s，实际链路还含 RTP/IP 开销。发送端 `key-int-max=fps` 大致每秒一个关键帧；`rtpjitterbuffer latency=35 drop-on-latency=true` 将抖动等待限制在约 35 ms；`appsink max-buffers=1 drop=true` 丢弃未及时显示的旧解码帧。这些是当前已用过的参数，**没有对端到端延迟做仪器测量**，因此不承诺固定毫秒值。

Qt 窗口在本地显示器上若有平台插件错误，先确认 `DISPLAY` 可用，再检查 `python3-pyqt5` 和 XWayland；本文命令显式选择 `QT_QPA_PLATFORM=xcb`。如果图形程序自身无法改变大小，本地最大化只能放大视频，不能增加源画面的像素；应先调整远端窗口或 Xvfb 屏幕尺寸。

对于需要 OpenGL 的 3D 程序，远端 Xvfb 可能使用软件渲染。若画面源头就只有很低的帧率，先在远端检查渲染器（安装 `mesa-utils` 后运行 `DISPLAY=:91 glxinfo -B`）。有可用 GPU 且已安装 VirtualGL 时，可把启动脚本中的程序命令改成类似 `/opt/VirtualGL/bin/vglrun -d /dev/dri/card0 /绝对路径/3D程序 参数`，并确认运行用户拥有对应 `/dev/dri/card*`、`/dev/dri/renderD*` 的访问权限。此步骤只改变远端的 3D 渲染路径，视频和输入脚本保持一致；普通 GUI 程序不需要它。GPU 设备号要按 `ls -l /dev/dri/` 的实际输出选择。

## 8. 停止、重启和清理

先关闭本地两个接收窗口（或在两个前台终端按 `Ctrl+C`），再在远端停止业务程序，最后停止虚拟屏幕：

```bash
systemctl --user stop lan-app-a lan-app-b
systemctl --user stop lan-screen-a lan-screen-b
```

确认没有遗留的发送进程和显示进程：

```bash
pgrep -af 'capture_supervisor.py|gst-launch-1.0|Xvfb' || true
```

重启远端图形程序后，重新执行 `xwininfo` 获取新的窗口 ID，再启动本地接收窗口。本文使用的是 transient systemd 服务；机器重启后从第 5 节重新启动即可。

## 9. RViz 滚轮不响应的处理

部分 RViz/X11/远程输入组合中，直接转发 X11 的滚轮按钮 `Button4`/`Button5` 后，RViz 视角的 `Distance` 没变化。此时只对 **RViz 对应的接收窗口** 增加 `--wheel-mode right-drag`。它把每次滚轮事件转换为远端视口内一次短暂的鼠标右键拖动，其他窗口仍按普通滚轮事件发送。

```bash
# 在 RViz 对应的 viewer.py 启动命令末尾追加：
--wheel-mode right-drag --wheel-pixels 20
```

`--wheel-pixels 20` 表示常见的 `angleDelta=120` 每格映射为 20 个远端像素；这保留了本次验证过的每格缩放幅度。滚轮向上/向下对应右键拖动的相反方向，脚本会把指针移回原位置。只有鼠标位于视频内容范围内才发送该操作。若其他 3D 软件也用右键拖动缩放，可同样选择该模式；普通表单、网页和文本程序使用默认 `normal`。

## 附录 A：远端 `capture_supervisor.py`

将下面代码完整保存到 `/opt/lan-lowlatency/capture_supervisor.py`。

```python
#!/usr/bin/env python3
"""Run one UDP video sender only while its SSH client remains connected."""

import argparse
import select
import subprocess
import sys


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--display", required=True)
    parser.add_argument("--xid", required=True)
    parser.add_argument("--local-ip", required=True)
    parser.add_argument("--port", type=int, required=True)
    parser.add_argument("--fps", type=int, default=12)
    parser.add_argument("--bitrate", type=int, default=8000)
    args = parser.parse_args()
    command = [
        "gst-launch-1.0", "-q", "--no-position", "ximagesrc",
        f"display-name={args.display}", f"xid={int(args.xid, 0)}",
        "use-damage=false", "show-pointer=true", "!",
        f"video/x-raw,framerate={args.fps}/1", "!",
        "videoconvert", "!", "queue", "max-size-buffers=1", "leaky=downstream", "!",
        "x264enc", "tune=zerolatency", "speed-preset=ultrafast",
        f"bitrate={args.bitrate}", f"key-int-max={args.fps}",
        "bframes=0", "byte-stream=true", "threads=2", "!",
        "video/x-h264,profile=baseline", "!",
        "rtph264pay", "config-interval=1", "pt=96", "mtu=1200", "!",
        "udpsink", f"host={args.local_ip}", f"port={args.port}",
        "sync=false", "async=false",
    ]
    child = subprocess.Popen(command, stdin=subprocess.DEVNULL)
    try:
        while child.poll() is None:
            readable, _, _ = select.select([sys.stdin.buffer], [], [], 0.5)
            if readable and not sys.stdin.buffer.read(1):
                break
        if child.poll() is not None:
            return child.returncode
        return 0
    finally:
        if child.poll() is None:
            child.terminate()
            try:
                child.wait(timeout=2)
            except subprocess.TimeoutExpired:
                child.kill()
                child.wait()


if __name__ == "__main__":
    raise SystemExit(main())
```

## 附录 B：远端 `input_relay.py`

将下面代码完整保存到 `/opt/lan-lowlatency/input_relay.py`。

```python
#!/usr/bin/env python3
"""Inject ordered pointer/keyboard events into one remote X11 window via SSH stdin."""

import argparse
import json
import sys
import time

from Xlib import X, XK, display
from Xlib.ext import xtest


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--display", required=True)
    parser.add_argument("--xid", required=True)
    args = parser.parse_args()
    server = display.Display(args.display)
    root = server.screen().root
    window = server.create_resource_object("window", int(args.xid, 0))

    for line in sys.stdin:
        try:
            event = json.loads(line)
            kind = event["type"]
            if kind == "move":
                origin = window.translate_coords(root, 0, 0)
                xtest.fake_input(
                    server, X.MotionNotify,
                    x=origin.x + int(event["x"]),
                    y=origin.y + int(event["y"]),
                )
            elif kind == "button":
                if event.get("pressed"):
                    window.raise_window()
                    window.set_input_focus(X.RevertToParent, X.CurrentTime)
                xtest.fake_input(
                    server,
                    X.ButtonPress if event.get("pressed") else X.ButtonRelease,
                    detail=int(event["button"]),
                )
                if event.get("pressed"):
                    server.sync()
                    time.sleep(0.02)
            elif kind == "zoom":
                origin = window.translate_coords(root, 0, 0)
                x = origin.x + int(event["x"])
                y = origin.y + int(event["y"])
                pixels = int(event["pixels"])
                xtest.fake_input(server, X.MotionNotify, x=x, y=y)
                window.raise_window()
                window.set_input_focus(X.RevertToParent, X.CurrentTime)
                xtest.fake_input(server, X.ButtonPress, detail=3)
                server.sync()
                time.sleep(0.015)
                try:
                    xtest.fake_input(server, X.MotionNotify, x=x, y=y + pixels)
                    server.sync()
                    time.sleep(0.015)
                finally:
                    xtest.fake_input(server, X.ButtonRelease, detail=3)
                    server.sync()
                xtest.fake_input(server, X.MotionNotify, x=x, y=y)
            elif kind == "key":
                keysym = int(event.get("keysym", 0))
                if not keysym:
                    keysym = XK.string_to_keysym(event.get("name", ""))
                keycode = server.keysym_to_keycode(keysym)
                if keycode:
                    window.set_input_focus(X.RevertToParent, X.CurrentTime)
                    xtest.fake_input(
                        server,
                        X.KeyPress if event.get("pressed") else X.KeyRelease,
                        detail=keycode,
                    )
            server.sync()
        except Exception as exc:
            print(f"input event rejected: {exc}", file=sys.stderr, flush=True)


if __name__ == "__main__":
    main()
```

## 附录 C：本地 `viewer.py`

将下面代码完整保存到 `/opt/lan-lowlatency/viewer.py`。

```python
#!/usr/bin/python3
"""Low-latency X11 window video with reliable SSH input forwarding."""

import argparse
from collections import deque
import json
import os
import shlex
import subprocess
import threading
import time

import gi
gi.require_version("Gst", "1.0")
from gi.repository import Gst
from PyQt5.QtCore import QObject, Qt, QTimer, pyqtSignal
from PyQt5.QtGui import QImage, QPainter
from PyQt5.QtWidgets import QApplication, QWidget


class Frames(QObject):
    available = pyqtSignal()

    def __init__(self):
        super().__init__()
        self._lock = threading.Lock()
        self._latest = QImage()
        self._pending = False

    def offer(self, image):
        with self._lock:
            self._latest = image
            if self._pending:
                return
            self._pending = True
        self.available.emit()

    def take(self):
        with self._lock:
            image = self._latest
            self._latest = QImage()
            self._pending = False
        return image


class InputWriter:
    def __init__(self, process):
        self.process = process
        self.pending = deque()
        self.condition = threading.Condition()
        self.running = True
        self.thread = threading.Thread(target=self._run, daemon=True)
        self.thread.start()

    def send(self, event):
        with self.condition:
            if event["type"] == "move" and self.pending and self.pending[-1]["type"] == "move":
                self.pending[-1] = event
            else:
                if len(self.pending) > 256:
                    self.pending = deque(item for item in self.pending if item["type"] != "move")
                self.pending.append(event)
            self.condition.notify()

    def _run(self):
        while True:
            with self.condition:
                while not self.pending and self.running:
                    self.condition.wait()
                if not self.running:
                    return
                event = self.pending.popleft()
            try:
                self.process.stdin.write((json.dumps(event, separators=(",", ":")) + "\n").encode())
                self.process.stdin.flush()
            except (BrokenPipeError, OSError):
                return

    def close(self):
        with self.condition:
            self.running = False
            self.condition.notify()
        try:
            self.process.stdin.close()
        except OSError:
            pass
        self.thread.join(timeout=1)


class Viewer(QWidget):
    def __init__(self, args):
        super().__init__()
        self.args = args
        self.image = QImage()
        self.image_rect = None
        self.frame_times = deque(maxlen=30)
        self.last_frame = 0.0
        self.setWindowTitle(f"{args.title}（低延迟）")
        self.resize(args.width, args.height)
        self.setMouseTracking(True)
        self.setFocusPolicy(Qt.StrongFocus)

        ssh = ["ssh", "-o", "BatchMode=yes", "-o", "ServerAliveInterval=10", args.remote]
        video_command = [
            "python3", "/opt/lan-lowlatency/capture_supervisor.py",
            "--display", args.display, "--xid", args.xid,
            "--local-ip", args.local_ip, "--port", str(args.port),
            "--fps", str(args.fps), "--bitrate", str(args.bitrate),
        ]
        input_command = [
            "python3", "/opt/lan-lowlatency/input_relay.py",
            "--display", args.display, "--xid", args.xid,
        ]
        logdir = "/tmp/lan-lowlatency"
        os.makedirs(logdir, exist_ok=True)
        self.video_log = open(f"{logdir}/{args.name}-video.log", "w")
        self.input_log = open(f"{logdir}/{args.name}-input.log", "w")
        self.video_process = subprocess.Popen(
            ssh + [shlex.join(video_command)],
            stdin=subprocess.PIPE, stdout=self.video_log, stderr=subprocess.STDOUT,
        )
        self.input_process = subprocess.Popen(
            ssh + [shlex.join(input_command)],
            stdin=subprocess.PIPE, stdout=self.input_log, stderr=subprocess.STDOUT,
        )
        self.input = InputWriter(self.input_process)

        Gst.init(None)
        pipeline = (
            f"udpsrc port={args.port} buffer-size=262144 "
            "caps=application/x-rtp,media=video,encoding-name=H264,payload=96,clock-rate=90000 "
            "! rtpjitterbuffer latency=35 drop-on-latency=true "
            "! rtph264depay ! h264parse ! avdec_h264 max-threads=2 "
            "! videoconvert ! video/x-raw,format=RGB "
            "! appsink name=latest sync=false max-buffers=1 drop=true emit-signals=true"
        )
        self.pipeline = Gst.parse_launch(pipeline)
        self.sink = self.pipeline.get_by_name("latest")
        self.sink.connect("new-sample", self._sample)
        self.frames = Frames()
        self.frames.available.connect(self._receive)
        self.pipeline.set_state(Gst.State.PLAYING)
        self.timer = QTimer(self)
        self.timer.timeout.connect(self._check)
        self.timer.start(250)

    def _sample(self, sink):
        sample = sink.emit("pull-sample")
        if sample is None:
            return Gst.FlowReturn.ERROR
        caps = sample.get_caps().get_structure(0)
        width = caps.get_value("width")
        height = caps.get_value("height")
        buffer = sample.get_buffer()
        ok, mapped = buffer.map(Gst.MapFlags.READ)
        if not ok:
            return Gst.FlowReturn.ERROR
        try:
            pixels = bytes(mapped.data)
            stride = len(pixels) // height
            image = QImage(pixels, width, height, stride, QImage.Format_RGB888).copy()
        finally:
            buffer.unmap(mapped)
        self.frames.offer(image)
        return Gst.FlowReturn.OK

    def _receive(self):
        self.image = self.frames.take()
        self.last_frame = time.monotonic()
        self.frame_times.append(self.last_frame)
        self.update()

    def _check(self):
        message = self.pipeline.get_bus().pop_filtered(Gst.MessageType.ERROR)
        if message:
            error, debug = message.parse_error()
            self.setWindowTitle(f"{self.args.title}：视频错误 {error.message}")
            return
        if self.video_process.poll() is not None:
            self.setWindowTitle(f"{self.args.title}：远端视频已退出，请查日志")
            return
        if self.last_frame:
            age = time.monotonic() - self.last_frame
            fps = 0.0
            if len(self.frame_times) >= 2:
                fps = (len(self.frame_times) - 1) / (self.frame_times[-1] - self.frame_times[0])
            self.setWindowTitle(
                f"{self.args.title}（低延迟，{self.image.width()}×{self.image.height()}，"
                f"{fps:.0f} fps，画面更新 {age:.1f} s）"
            )

    def paintEvent(self, _event):
        painter = QPainter(self)
        painter.fillRect(self.rect(), Qt.black)
        if self.image.isNull():
            painter.setPen(Qt.white)
            painter.drawText(self.rect(), Qt.AlignCenter, "正在连接视频…")
            self.image_rect = None
            return
        size = self.image.size()
        size.scale(self.size(), Qt.KeepAspectRatio)
        x = (self.width() - size.width()) // 2
        y = (self.height() - size.height()) // 2
        self.image_rect = (x, y, size.width(), size.height())
        painter.setRenderHint(QPainter.SmoothPixmapTransform, False)
        painter.drawImage(x, y, self.image.scaled(size, Qt.IgnoreAspectRatio, Qt.FastTransformation))

    def _coords(self, position):
        if self.image_rect is None:
            return None
        x0, y0, width, height = self.image_rect
        if not (x0 <= position.x() < x0 + width and y0 <= position.y() < y0 + height):
            return None
        return {
            "x": min(self.image.width() - 1, int((position.x() - x0) * self.image.width() / width)),
            "y": min(self.image.height() - 1, int((position.y() - y0) * self.image.height() / height)),
        }

    def mouseMoveEvent(self, event):
        point = self._coords(event.pos())
        if point:
            self.input.send({"type": "move", **point})

    def mousePressEvent(self, event):
        point = self._coords(event.pos())
        if not point:
            return
        self.setFocus()
        self.input.send({"type": "move", **point})
        button = {Qt.LeftButton: 1, Qt.MiddleButton: 2, Qt.RightButton: 3}.get(event.button())
        if button:
            self.input.send({"type": "button", "button": button, "pressed": True})

    def mouseReleaseEvent(self, event):
        button = {Qt.LeftButton: 1, Qt.MiddleButton: 2, Qt.RightButton: 3}.get(event.button())
        if button:
            self.input.send({"type": "button", "button": button, "pressed": False})

    def wheelEvent(self, event):
        point = self._coords(event.pos())
        if not point:
            return
        delta = event.angleDelta().y() or event.pixelDelta().y()
        if not delta:
            return
        if self.args.wheel_mode == "right-drag":
            pixels = max(8, min(80, round(abs(delta) / 120 * self.args.wheel_pixels)))
            self.input.send({"type": "zoom", **point, "pixels": -pixels if delta > 0 else pixels})
        else:
            self.input.send({"type": "move", **point})
            button = 4 if delta > 0 else 5
            for _ in range(max(1, abs(delta) // 120)):
                self.input.send({"type": "button", "button": button, "pressed": True})
                self.input.send({"type": "button", "button": button, "pressed": False})
        event.accept()

    def keyPressEvent(self, event):
        self.input.send({"type": "key", "keysym": event.nativeVirtualKey(), "name": event.text(), "pressed": True})

    def keyReleaseEvent(self, event):
        self.input.send({"type": "key", "keysym": event.nativeVirtualKey(), "name": event.text(), "pressed": False})

    def closeEvent(self, event):
        self.timer.stop()
        self.pipeline.set_state(Gst.State.NULL)
        self.input.close()
        try:
            self.video_process.stdin.close()
        except OSError:
            pass
        for process in (self.video_process, self.input_process):
            if process.poll() is None:
                process.terminate()
                try:
                    process.wait(timeout=2)
                except subprocess.TimeoutExpired:
                    process.kill()
        self.video_log.close()
        self.input_log.close()
        super().closeEvent(event)


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--name", required=True)
    parser.add_argument("--title", required=True)
    parser.add_argument("--remote", required=True)
    parser.add_argument("--local-ip", required=True)
    parser.add_argument("--display", required=True)
    parser.add_argument("--xid", required=True)
    parser.add_argument("--port", type=int, required=True)
    parser.add_argument("--fps", type=int, default=12)
    parser.add_argument("--bitrate", type=int, default=8000)
    parser.add_argument("--width", type=int, default=1280)
    parser.add_argument("--height", type=int, default=960)
    parser.add_argument("--wheel-mode", choices=("normal", "right-drag"), default="normal")
    parser.add_argument("--wheel-pixels", type=int, default=20)
    args = parser.parse_args()
    app = QApplication([])
    viewer = Viewer(args)
    viewer.show()
    app.exec_()


if __name__ == "__main__":
    main()
```
