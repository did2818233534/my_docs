# TethrLink 安卓手机 USB 副屏部署记录

本文记录 2026-09-21 在 Ubuntu GNOME Wayland 环境中，将安卓手机通过 USB 网络共享配置为扩展副屏的完整部署结果、定制内容、使用方法、验证数据及回滚方式。

## 1. 最终结果

| 项目 | 当前配置 |
|---|---|
| Ubuntu 桌面 | Ubuntu GNOME |
| 显示协议 | Wayland |
| TethrLink 版本 | 2.0.1（Debian 软件包） |
| 安卓设备 | 23117RK66C，2400×1080 |
| 连接方式 | USB 网络共享 / RNDIS |
| USB 链路 | USB 2.0 High-Speed，480 Mbit/s |
| 虚拟显示器分辨率 | 1280×576，20:9 |
| 串流目标帧率 | 60 FPS |
| 实测帧率 | 稳定 60–61 FPS |
| 视频编码 | H.264，`vah264enc` 硬件编码 |
| 编码码率 | 8000 kbit/s |
| 触摸输入 | 已启用 |
| TCP 串流端口 | 51137 |
| UDP 自动发现端口 | 8765 |

最终持续运行统计为：

```text
fps=60~61
dropped=0
overflows=0
```

USB 2.0 的有效带宽远高于当前约 8 Mbit/s 的视频流需求，因此本部署中的主要流畅度限制不是 USB 带宽，而是分辨率、编码/解码开销和串流帧率。

## 2. 基础安装与连接方式

Ubuntu 端安装 TethrLink Debian 软件包：

```bash
sudo apt install ./tethrlink_all.deb
```

安卓端安装同一发行版本提供的 `tethrlink.apk`。

日常连接顺序：

1. 使用支持数据传输的 USB 线连接手机。
2. 在手机中开启“USB 网络共享”。
3. 从 Ubuntu 应用菜单启动 TethrLink。
4. 点击 **Start Server**。
5. 打开手机端 TethrLink；手机会通过 UDP 自动发现电脑。
6. 在 Ubuntu“设置 → 显示器”中安排副屏位置。

## 3. 本次定制内容

TethrLink 2.0.1 原版将 H.264 串流帧率固定为 30 FPS，且没有公开的帧率环境变量。本次在以下文件中加入了帧率和自动启动参数支持：

```text
/usr/lib/tethrlink/server/app/main.py
```

新增环境变量：

| 环境变量 | 作用 | 有效范围/示例 |
|---|---|---|
| `TETHRLINK_RES` | 覆盖虚拟显示器分辨率 | `1280x576` |
| `TETHRLINK_FPS` | 覆盖串流帧率 | 整数 `1`～`120` |
| `TETHRLINK_AUTOSTART` | 启动应用后自动启动服务 | `1`、`true`、`yes` 或 `on` |

上游默认值仍保留为 30 FPS。只有显式设置 `TETHRLINK_FPS` 时才会启用更高帧率，避免以后误将手机原生 2400×1080 分辨率直接运行在 60 FPS。

应用菜单启动项已修改为：

```ini
Exec=env TETHRLINK_RES=1280x576 TETHRLINK_FPS=60 tethrlink
```

文件位置：

```text
/usr/share/applications/tethrlink.desktop
```

因此，以后从应用菜单正常打开 TethrLink，就会使用 1280×576 和 60 FPS。应用菜单启动仍需手动点击 **Start Server**。

## 4. 手动启动命令

标准启动：

```bash
TETHRLINK_RES=1280x576 TETHRLINK_FPS=60 tethrlink
```

需要启动应用后自动开启服务时：

```bash
TETHRLINK_RES=1280x576 \
TETHRLINK_FPS=60 \
TETHRLINK_AUTOSTART=1 \
tethrlink
```

临时退回 30 FPS：

```bash
TETHRLINK_RES=1280x576 TETHRLINK_FPS=30 tethrlink
```

临时退回更低分辨率的 60 FPS：

```bash
TETHRLINK_RES=960x432 TETHRLINK_FPS=60 tethrlink
```

环境变量只在进程启动时读取。修改参数前必须停止服务并完全退出旧实例，再重新启动。

## 5. 验证方法

确认桌面环境：

```bash
echo "$XDG_CURRENT_DESKTOP / $XDG_SESSION_TYPE"
```

期望结果：

```text
ubuntu:GNOME / wayland
```

确认 USB 网络共享工作在 USB 2.0 High-Speed：

```bash
lsusb -t
```

手机对应接口应包含：

```text
Driver=rndis_host, 480M
```

确认服务正在监听：

```bash
ss -lptn | grep ':51137'
```

确认实际运行进程：

```bash
ps -eo pid,ppid,stat,user,comm,args \
  | grep '[p]ython3 -m server.app.main'
```

确认应用菜单的持久参数：

```bash
grep '^Exec=' /usr/share/applications/tethrlink.desktop
```

当前会话日志位于：

```text
/tmp/tethrlink-1280x576-60fps.log
```

查看实时统计：

```bash
tail -f /tmp/tethrlink-1280x576-60fps.log
```

正常日志应包含：

```text
Using configured resolution: 1280×576
Starting — 1280×576 @ 60 FPS (H.264)
video/x-raw,framerate=60/1
H.264 framerate is fixed at 60 FPS by the pipeline
fps=60 ... dropped=0 ... overflows=0
```

## 6. 旧实例无法退出的处理方法

TethrLink 的实际进程名称是 `python3`，因此仅按程序名执行 `pkill tethrlink` 可能无法结束它。不要使用 `killall python3`，否则可能误杀 ROS 或其他 Python 程序。

优先在界面点击 **Stop Server** 并关闭窗口。如果仍未退出，先定位精确 PID：

```bash
ss -lptn | grep ':51137'
```

或：

```bash
ps -eo pid,ppid,stat,user,comm,args \
  | grep '[p]ython3 -m server.app.main'
```

然后只结束对应 PID：

```bash
kill <PID>
```

若数秒后仍存在：

```bash
kill -KILL <PID>
```

当应用由 GNOME 启动时，也可能位于类似以下用户作用域中：

```text
app-gnome-tethrlink-<PID>.scope
```

可先查询：

```bash
systemctl --user list-units --all | grep -i tethr
```

再停止精确匹配的作用域：

```bash
systemctl --user stop app-gnome-tethrlink-<PID>.scope
```

## 7. 常见现象与排查

### 界面仍显示 2400×1080

说明旧进程仍在运行，新的 `TETHRLINK_RES` 没有应用到实际服务。完全退出旧实例后再启动，并检查日志中的 `Using configured resolution`。

### 第一次连接立即断开一次

切换虚拟显示器分辨率时，Mutter/PipeWire 可能重建显示节点，硬件编码器也会随之重新初始化。手机通常会自动重连。应以第二次连接后连续数秒的 `fps`、`dropped` 和 `overflows` 数据判断稳定性。

### 画面仍有明显延迟

检查：

1. 视频编码器是否为 H.264，而不是 JPEG。
2. 日志是否持续出现 `dropped` 或 `overflows`。
3. 手机 USB 接口是否为 `rndis_host, 480M`，而不是 `12M`。
4. 与手机处于同一 USB 2.0 总线的摄像头是否正在高负载工作。
5. 必要时临时退回 `960×432 @ 60 FPS` 对比。

### 软件更新后恢复到 30 FPS

重新安装或升级 TethrLink 可能覆盖 `/usr/lib/tethrlink/server/app/main.py` 和桌面启动项。升级后检查 `TETHRLINK_FPS` 支持及桌面文件的 `Exec=` 行。

## 8. 备份与回滚

修改前的原始文件已备份为：

```text
/usr/lib/tethrlink/server/app/main.py.codex-backup-20260921-1745
/usr/lib/tethrlink/tethrlink.desktop.codex-backup-20260921-1751
```

恢复原始 Python 程序：

```bash
cp /usr/lib/tethrlink/server/app/main.py.codex-backup-20260921-1745 \
   /usr/lib/tethrlink/server/app/main.py
```

恢复原始桌面启动项：

```bash
cp /usr/lib/tethrlink/tethrlink.desktop.codex-backup-20260921-1751 \
   /usr/share/applications/tethrlink.desktop
```

恢复后完全退出并重新打开 TethrLink。上游默认配置将重新使用设备自动分辨率和约 30 FPS。

## 9. 当前推荐配置

当前手机屏幕比例为 20:9，因此 `1280×576` 能保持正确宽高比。它比 `960×432` 更清晰，同时本机实测仍能通过 `vah264enc` 稳定保持 60–61 FPS，无持续丢帧或队列溢出，适合作为当前默认配置。
