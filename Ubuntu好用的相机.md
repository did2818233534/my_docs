# Ubuntu 好用的相机：BigCam 安装记录

记录日期：2026-09-23。适用环境：本机 Ubuntu 24.04.4 LTS（x86_64）。

## 安装结果

| 内容 | 版本或位置 |
| --- | --- |
| BigCam | 4.5.0，源码提交 `447bfb1eece06d311038c3692c4267300f09e753` |
| 程序目录 | `/opt/bigcam` |
| GTK4 GStreamer 视频插件 | `gtk4paintablesink` 0.14.0，私有存放在 `/opt/bigcam/lib/gstreamer-1.0/` |
| GStreamer `GstVideo` 类型信息 | 从 Ubuntu 24.04 的 `gir1.2-gst-plugins-base-1.0` 包提取，私有存放在 `/opt/bigcam/lib/girepository-1.0/` |
| 应用菜单入口 | `/usr/share/applications/br.com.biglinux.bigcam.desktop`，显示为“BigCam 镜像相机” |

BigCam 源码来自[官方仓库](https://github.com/biglinux/bigcam)。Ubuntu 24.04 软件源没有现成的 `gstreamer1.0-gtk4` 二进制包，所以本次从[Ubuntu 官方软件包档案](https://packages.ubuntu.com/questing/gstreamer1.0-gtk4)下载较新版本的插件，并仅放进 BigCam 的私有目录。没有添加新软件源，也没有替换系统的 GStreamer 文件。`gtk4paintablesink` 是 [GStreamer 官方的 GTK4 视频输出组件](https://gstreamer.freedesktop.org/documentation/gtk4/)。

## 本次安装流程

1. 确认系统是 Ubuntu 24.04，内置摄像头可被 `v4l2-ctl --list-devices` 识别。本机是 `/dev/video0`。
2. 从 BigCam 官方仓库获取源码，固定到上述提交。只复制应用代码、翻译和图标；没有运行仓库里的 Arch Linux 安装脚本。
3. 下载 `gstreamer1.0-gtk4_0.14.0-3ubuntu1_amd64.deb`，用 `dpkg-deb -x` 解包，取出 `libgstgtk4.so`。该包来自较新的 Ubuntu 版本；在本机 GTK 4.14 和 GStreamer 1.24 环境中验证了可加载。
4. 从本机 Ubuntu 24.04 软件源下载 `gir1.2-gst-plugins-base-1.0`，同样只解包，取出 `GstVideo-1.0.typelib` 等类型信息。
5. 将上述文件和 BigCam 应用组合到 `/opt/bigcam`，创建启动脚本和应用菜单入口。程序本体没有装进主目录。
6. 启动脚本单独设置 `GST_PLUGIN_PATH`、`GI_TYPELIB_PATH`，让 BigCam 找到私有组件；同时设置 `LC_MESSAGES=zh_CN.UTF-8` 和 `LANGUAGE=zh_CN:zh`，使界面优先显示中文。

当时用于获取源码和组件的关键命令如下。安装已完成，**不需要在本机重复执行**：

```bash
git clone https://github.com/biglinux/bigcam.git bigcam
git -C bigcam checkout 447bfb1eece06d311038c3692c4267300f09e753

curl -fL -o gstreamer1.0-gtk4_0.14.0-3ubuntu1_amd64.deb \
  https://mirrors.edge.kernel.org/ubuntu/pool/main/r/rust-gst-plugin-gtk4/gstreamer1.0-gtk4_0.14.0-3ubuntu1_amd64.deb
dpkg-deb -x gstreamer1.0-gtk4_0.14.0-3ubuntu1_amd64.deb gtk4-package

apt download gir1.2-gst-plugins-base-1.0
dpkg-deb -x gir1.2-gst-plugins-base-1.0_*.deb gst-types-package
```

完成解包后，实际复制到 `/opt/bigcam` 的目录结构是：

```text
/opt/bigcam/
├── bin/bigcam
├── lib/
│   ├── gstreamer-1.0/libgstgtk4.so
│   └── girepository-1.0/*.typelib
└── share/
    ├── biglinux/bigcam/       # BigCam 程序
    ├── locale/zh/LC_MESSAGES/bigcam.mo
    ├── icons/
    └── applications/br.com.biglinux.bigcam.desktop
```

`/opt/bigcam/bin/bigcam` 启动脚本的关键配置：

```sh
#!/bin/sh
app_root=$(CDPATH= cd -- "$(dirname -- "$0")/.." && pwd)
export GST_PLUGIN_PATH="$app_root/lib/gstreamer-1.0${GST_PLUGIN_PATH:+:$GST_PLUGIN_PATH}"
export GI_TYPELIB_PATH="$app_root/lib/girepository-1.0${GI_TYPELIB_PATH:+:$GI_TYPELIB_PATH}"
export LC_MESSAGES=zh_CN.UTF-8
export LANGUAGE=zh_CN:zh
export PYTHONDONTWRITEBYTECODE=1
exec /usr/bin/python3 "$app_root/share/biglinux/bigcam/main.py" "$@"
```

## 使用和验证

从应用菜单搜索 **BigCam 镜像相机**，或在终端运行：

```bash
/opt/bigcam/bin/bigcam
```

镜像按钮位于窗口底栏左侧，也可以按 **Ctrl+M** 切换。**BigCam 目前只镜像预览画面，保存的照片和视频仍按摄像头原始方向写入。**

以下命令可分别检查 GTK4 插件和中文翻译：

```bash
GST_PLUGIN_PATH=/opt/bigcam/lib/gstreamer-1.0 gst-inspect-1.0 gtk4paintablesink

LC_MESSAGES=zh_CN.UTF-8 LANGUAGE=zh_CN:zh \
PYTHONPATH=/opt/bigcam/share/biglinux/bigcam \
python3 -c "from utils.i18n import _; print(_('Mirror preview'))"
```

本机检查结果：插件可以加载，第二条命令输出“镜像预览”；BigCam 能识别并启动内置摄像头。当前 Ubuntu 的 Python GStreamer 接口缺少 BigCam 优先使用的缓冲区替换能力，因此启动日志显示它选择了兼容的 `appsink` 预览路径。这不影响 GTK4 插件已安装和可加载的检查结果。

如需排查启动问题，查看 `~/.local/state/bigcam/debug.log`，并确认没有其他程序正在占用摄像头：

```bash
fuser /dev/video0
```

## 卸载

如果以后不再使用 BigCam，可删除本次创建的程序目录和菜单入口：

```bash
sudo rm -r -- /opt/bigcam
sudo rm -- /usr/share/applications/br.com.biglinux.bigcam.desktop
```

BigCam 自己生成的用户设置和日志可能留在 `~/.config/bigcam/`、`~/.local/state/bigcam/`；需要彻底清除时再单独删除。系统原有的 GTK、OpenCV、GStreamer 等软件包不属于本次私有安装，卸载 BigCam 时无需移除。
