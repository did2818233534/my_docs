# Ubuntu 文件管理与文件选择窗口定制：换机复现文档

记录日期：2026-10-12；更新日期：2026-10-13（增加复制绝对路径）。本文件包含完整源码；把这一份 Markdown 交给新机器上的 ChatGPT，即可按文档重建，无需原机器上的二进制文件或临时工作目录。

## 1. 要达到的效果

| 场景 | 目标行为 | 实现方式 |
| --- | --- | --- |
| Ubuntu“文件”（Nautilus）的文件夹空白处右键 | 一级菜单直接显示“新建文档”，没有二级菜单；创建空白、无后缀的文件 | 空白模板 + Nautilus 原生模板动作 + GResource 菜单覆盖 |
| 文件夹空白处或文件夹上右键 | 显示打开 Terminator 的菜单，在相应目录启动 Terminator | `nautilus-open-any-terminal` 扩展 |
| 选中文件或文件夹后右键 | 主菜单显示“复制绝对路径”，复制结果包含选中对象本身；多选时每行一个 | 自定义 Nautilus Python 菜单扩展 |
| 应用的“打开文件”窗口顶部目录栏 | 单击目录栏即可编辑完整路径；输入目录路径并按 Enter 进入目录；输入文件路径并按 Enter 将文件交给当前应用 | 自编译 GTK 3 模块 + GNOME 文件选择 portal 使用 GTK 后端 |

“打开文件”指完成当前应用的文件选择操作，例如打开编辑器文档或选中附件。文件选择窗口的“保存文件”模式保持原生行为。此方案没有修改全局 `Ctrl+Alt+T` 快捷键。

文件选择模块覆盖 GTK 3 的打开文件、选择目录控件；文件路径的确认由原生控件处理。通过 GNOME portal 调用的文件选择窗口也会使用 GTK 3 后端。应用自己实现的 Qt、网页界面、直接调用 GTK 4 的选择器，以及沙箱内独立的 GTK 环境，需要另行确认，不能保证全部适用。

## 2. 可以直接交给 ChatGPT 的任务说明

> 请在当前 Ubuntu GNOME 桌面上，按本文件实现第 1 节的四项效果。先确认发行版、Nautilus/GTK 版本、当前桌面用户和真实 XDG 目录，再提取文末源码，安装、配置并验证。不要把旧机器的用户名 guga、`/data/Templates`、UID 1000 或 x86_64 路径硬编码到新机器。自编写的软件安装到 `/opt/nautilus-direct-document` 和 `/opt/gtk-path-entry`；apt 软件采用系统默认目录。用户设置必须以当前桌面用户执行。保留现有模板和无关配置，保留首次修改前的备份。完成后测试中文和含空格的路径，并清理自己产生的测试进程和临时目录。若版本不兼容，先适配源码并验证，不要直接修改系统 Nautilus/GTK 二进制。只有四项验收全部通过后，才报告完成。

## 3. 已验证的环境与限制

本机为 Ubuntu 24.04.4 LTS、GNOME 46、Wayland，会话用户为 `guga`。

| 组件 | 已验证版本 |
| --- | --- |
| Nautilus | `46.4-0ubuntu0.2` |
| GTK 3 | `3.24.41-4ubuntu1.3` |
| GTK 4 | `4.14.5+ds-0ubuntu0.10`，文件选择 portal 原后端使用它 |
| Terminator | `2.1.3-1` |
| python3-nautilus | `4.0-1build4` |
| nautilus-open-any-terminal | `0.8.3` |
| xdg-desktop-portal-gnome | `46.2-0ubuntu1` |
| xdg-desktop-portal-gtk | `1.15.1-1build2` |

这不是任意 Ubuntu 版本都可直接套用的系统设置。Nautilus 菜单资源、GTK 内部控件名称、portal 配置规则可能随版本变化。新版本必须先检查本文使用的接口；GTK 模块应在新机器重新编译，不要复制旧机器的 `.so`。不同 CPU 架构使用新机器的 multiarch 路径。

Nautilus 启动器在无法识别菜单或模板时会退回正常 Nautilus，避免影响文件管理器启动。GTK 模块无法找到目标控件时会跳过修改；因此“安装成功”不能替代实际验收。

## 4. 准备、环境检查和提取源码

以下命令使用 Bash，按顺序执行，在当前已登录 GNOME 桌面的普通用户终端运行。下载校验、编译、安装或配置任一步失败时立即停止，排查后再继续；自动化脚本应检查退出码。不要整份文档都用 `sudo` 执行，也不要在 SSH 远程用户或 root 的 D-Bus 会话中改本地桌面设置。

```bash
cat /etc/os-release
printf '用户=%s UID=%s 桌面=%s 会话=%s\n' "$(id -un)" "$(id -u)" "$XDG_CURRENT_DESKTOP" "$XDG_SESSION_TYPE"
nautilus --version
xdg-user-dir TEMPLATES
dpkg-query -W -f='${Package}\t${Version}\n' nautilus libgtk-3-0t64 libgtk-4-1 xdg-desktop-portal-gnome xdg-desktop-portal-gtk 2>/dev/null
```

确认确实是 Nautilus + GNOME。检查 `/opt/nautilus-direct-document`、`/opt/gtk-path-entry` 和下文两个系统模块文件是否已有其他软件占用；已有本方案时先核对版本、原始备份，避免把已定制状态当作首次安装前状态。

把文档保存在本地，并在文档所在目录执行：

```bash
export REPRO_DOC="$PWD/Ubuntu文件管理与文件选择窗口定制.md"
export REPRO_TASK_DIR="$(mktemp -d /tmp/ubuntu-file-dialog-reproduce.XXXXXX)"
chmod 0755 "$REPRO_TASK_DIR"
export REPRO_STATE_DIR="${XDG_STATE_HOME:-$HOME/.local/state}/ubuntu-file-dialog-customization"
install -d -m 0700 "$REPRO_STATE_DIR"
python3 - <<'PYCODE'
import os
from pathlib import Path
import re
source = Path(os.environ['REPRO_DOC']).read_text()
output = Path(os.environ['REPRO_TASK_DIR'])
files = dict(re.findall(r'<!-- source-file: ([a-zA-Z0-9_.-]+) -->\n```(?:python|c)\n(.*?)\n```', source, re.S))
expected = {'launcher.py', 'configure-nautilus.py', 'path-entry.c', 'user-settings.py', 'copy_absolute_path.py'}
if set(files) != expected:
    raise SystemExit('文档源码块不完整，请检查文末五个附录。')
for name, code in files.items():
    (output / name).write_text(code + '\n')
    print('已提取：', output / name)
PYCODE
```

源码文件仅暂存在 `/tmp`；长期安装位置仍然是 `/opt`。用户配置、模板、备份放在相应 XDG 目录，这是配置数据，不是把软件安装进家目录。

首次安装前记录相关软件包的状态；若文件已经存在，保留原始记录：

```bash
if [ ! -e "$REPRO_STATE_DIR/packages-before.txt" ]; then
    dpkg-query -W -f='${Package}\t${db:Status-Status}\t${Version}\n' \
        terminator nautilus-extension-gnome-terminal nautilus-extension-any-terminal \
        > "$REPRO_STATE_DIR/packages-before.txt" 2>/dev/null || true
fi
```

安装依赖。下面是已验证的 Ubuntu 24.04 包名，其他发行版先核对包名和模拟结果：

```bash
sudo apt-get update
sudo apt-get install --no-install-recommends \
    terminator python3-gi python3-nautilus gir1.2-nautilus-4.0 \
    xdg-user-dirs desktop-file-utils libglib2.0-bin libglib2.0-dev \
    gcc pkg-config dpkg-dev curl xdg-desktop-portal xdg-desktop-portal-gtk
```

`xdg-user-dir TEMPLATES` 必须返回独立的绝对目录，不能是家目录本身。如果模板目录功能被关闭或路径无效，先设置独立目录，例如：

```bash
mkdir -p "$HOME/Templates"
xdg-user-dirs-update --set TEMPLATES "$HOME/Templates"
```

已有有效目录就直接使用，不要把本机的 `/data/Templates` 强行搬到新机器。若该目录中已有名为“新建文档”的非空文件，先保留并改名；安装程序会拒绝覆盖它。

## 5. 一级菜单新建无后缀文档

```bash
sudo install -d -m 0755 -o "$(id -u)" -g "$(id -g)" /opt/nautilus-direct-document
install -m 0755 "$REPRO_TASK_DIR/launcher.py" /opt/nautilus-direct-document/launcher.py
install -m 0755 "$REPRO_TASK_DIR/configure-nautilus.py" /opt/nautilus-direct-document/configure-nautilus.py
python3 /opt/nautilus-direct-document/configure-nautilus.py
```

程序读取当前用户的 XDG 模板目录，创建 0 字节的“新建文档”，并配置以下三个用户入口：

- `${XDG_DATA_HOME:-~/.local/share}/applications/org.gnome.Nautilus.desktop`。
- 同一数据目录下 `dbus-1/services/org.gnome.Nautilus.service`。
- 同一数据目录下 `dbus-1/services/org.freedesktop.FileManager1.service`。

原始文件和模板状态保存在 `/opt/nautilus-direct-document/configuration-backup.json`，权限为 `0600`。重复安装保留首次备份；发现用户后来修改的配置会停止，等待检查。配置文件若为符号链接，工具会停止，先让 ChatGPT 根据实际托管方式适配，避免破坏原链接。

仅改 desktop 文件不够：GNOME 会通过 D-Bus 激活 Nautilus，两个服务文件都要覆盖。保留原 desktop 文件的 `DBusActivatable` 等属性，只替换其 Nautilus 启动命令。启动器从当前 `/usr/bin/nautilus` 动态提取菜单资源，在 `$XDG_RUNTIME_DIR/nautilus-direct-document` 生成覆盖文件，并设置 `G_RESOURCE_OVERLAYS`。

“新建文档”仍使用 Nautilus 原生模板动作，文件创建、重名处理、错误提示及撤销由 Nautilus 完成。其他模板文件会保留，但此定制的右键入口专用于空白文档，原来的模板子菜单不再显示。本机调试期间曾创建的空白 `新建文本文件.txt` 已清除；换机时不需要创建它，也不要自动删除新机器上的同名文件。

桌面和 D-Bus 入口可使用该启动器；直接启动 `/usr/bin/nautilus` 的全新进程会绕过覆盖。从终端需要相同效果时使用 `/opt/nautilus-direct-document/launcher.py`。

## 6. 右键打开 Terminator

固定使用上游 `0.8.3` 的 Debian 包，而不是下载“最新版”后沿用旧哈希：

```bash
curl --fail --location \
    https://github.com/Stunkymonkey/nautilus-open-any-terminal/releases/download/0.8.3/nautilus-extension-any-terminal_0.8.3-1_all.deb \
    --output "$REPRO_TASK_DIR/nautilus-extension-any-terminal_0.8.3-1_all.deb"
printf '%s  %s\n' \
    '939d85f0a176627a1781e7faec8d9c85408145e90f98f16558bce86481960c26' \
    "$REPRO_TASK_DIR/nautilus-extension-any-terminal_0.8.3-1_all.deb" | sha256sum -c -
sudo apt-get -s install --no-install-recommends \
    "$REPRO_TASK_DIR/nautilus-extension-any-terminal_0.8.3-1_all.deb" \
    nautilus-extension-gnome-terminal-
```

检查模拟结果，卸载项应只包含旧的 `nautilus-extension-gnome-terminal`；若还要卸载桌面、Nautilus 或其他无关软件，先解决依赖。确认后执行：

```bash
sudo apt-get install --no-install-recommends \
    "$REPRO_TASK_DIR/nautilus-extension-any-terminal_0.8.3-1_all.deb" \
    nautilus-extension-gnome-terminal-
python3 "$REPRO_TASK_DIR/user-settings.py" install terminal
```

上面命令最后的 `-` 是 apt 的指定卸载语法。这里只替换 GNOME Terminal 的 Nautilus 菜单扩展，不需要卸载 GNOME Terminal 应用，也不要执行 `autoremove`。

用户配置为：

```text
com.github.stunkymonkey.nautilus-open-any-terminal
terminal='terminator'
use-generic-terminal-name=false
new-tab=false
```

脚本保留原始用户设置，包括“原来没有显式值”的状态。菜单名称随系统语言变化，本机测试时显示 `Open Terminator Here`。该扩展使用选中目录作为进程工作目录，中文和空格路径已测试。

单独修改 `x-terminal-emulator` 的 alternatives 无法替换原 GNOME Terminal 扩展的菜单，因此不能用设置默认终端代替此步骤。全局快捷键属于另一项配置，不在本文复现范围内。

## 6.1 选中文件或文件夹后复制绝对路径

新增一级右键菜单“复制绝对路径”。例如选中 `c.urdf` 后复制 `/home/aaa/bbb/c.urdf`，而不是它的父目录；选中文件夹后复制包含该文件夹名称的路径。复制为纯文本，不加引号，不转换成 `file://` URI，也不编码中文、空格或 `%`。多选时每个路径一行；符号链接复制当前选中链接的路径，不解析为目标路径。没有本地路径的远程 URI 不显示此项。

完整源码在附录 E，已包含在第 4 节的自动提取中。它使用现有 `python3-nautilus`，不需要额外终端剪贴板工具，Wayland 和 X11 都通过 GDK 剪贴板接口复制。

如果已经完成第 5 节，`/opt/nautilus-direct-document` 已存在并由桌面用户管理。如果只复现本项，也需要 Nautilus 4/GTK 4、`python3-nautilus` 和 `gir1.2-nautilus-4.0`，并先按第 5 节的第一条命令创建该 `/opt` 目录。随后以普通桌面用户运行：

```bash
python3 - <<'PYCODE'
from pathlib import Path
import os
import shutil
source = Path(os.environ['REPRO_TASK_DIR']) / 'copy_absolute_path.py'
target = Path('/opt/nautilus-direct-document/copy_absolute_path.py')
share = Path(os.environ.get('XDG_DATA_HOME', str(Path.home() / '.local/share')))
entry = share / 'nautilus-python/extensions/copy_absolute_path.py'
if target.exists() and target.read_bytes() != source.read_bytes():
    raise SystemExit('已有同名扩展源码，请先备份检查，未覆盖。')
if entry.is_symlink():
    if entry.readlink() != target:
        raise SystemExit('扩展入口已被其他文件占用，未覆盖。')
elif entry.exists():
    raise SystemExit('扩展入口已被其他文件占用，未覆盖。')
shutil.copyfile(source, target)
target.chmod(0o644)
entry.parent.mkdir(parents=True, exist_ok=True)
if not entry.is_symlink():
    entry.symlink_to(target)
print('源码已安装到：', target)
print('Nautilus 扩展入口：', entry)
PYCODE
```

软件源码在 `/opt`；用户扩展目录只放一个软链接，保留其中其他扩展。按第 8 节安全重启 Nautilus 后菜单生效；不要在复制/移动文件进行中退出文件管理器。

实现使用 [Nautilus MenuProvider](https://gnome.pages.gitlab.gnome.org/nautilus-python/nautilus-python-overview-example.html) 提供选中对象的菜单，使用 [Gio.File.get_path](https://docs.gtk.org/gio/method.File.get_path.html) 取得本地绝对路径，包含对象本身。调用 GDK 剪贴板后，由正在运行的 Nautilus 提供文本给其他应用，不会启动驻留的 `xclip`/`wl-copy` 测试进程。

## 7. 文件选择窗口：单击目录栏编辑路径

先编译附录 C 的源码：

```bash
gcc -shared -fPIC -O2 -Wall -Wextra -Werror -Wl,-z,relro \
    -o "$REPRO_TASK_DIR/libpath-entry.so" "$REPRO_TASK_DIR/path-entry.c" \
    $(pkg-config --cflags --libs gobject-2.0 gmodule-2.0) -ldl
```

这里使用 GTK 3 的运行时接口，编译阶段依赖 GLib 开发包。安装编译产物、源码和用户配置工具：

```bash
export REPRO_MULTIARCH="$(dpkg-architecture -qDEB_HOST_MULTIARCH)"
export REPRO_MODULE_LINK="/usr/lib/$REPRO_MULTIARCH/gtk-3.0/modules/libpath-entry.so"
export REPRO_MODULE_DESCRIPTOR='/usr/lib/gnome-settings-daemon-3.0/gtk-modules/path-entry.desktop'
if [ -e "$REPRO_MODULE_LINK" ] || [ -L "$REPRO_MODULE_LINK" ]; then
    if [ "$(readlink "$REPRO_MODULE_LINK")" != '/opt/gtk-path-entry/libpath-entry.so' ]; then
        printf '模块名称已被占用：%s\n' "$REPRO_MODULE_LINK"
        exit 1
    fi
fi
cat > "$REPRO_TASK_DIR/path-entry.desktop" <<'DESKTOP'
[GTK Module]
Name=Editable file chooser directory bar
X-GTK-Module-Name=path-entry
DESKTOP
if [ -e "$REPRO_MODULE_DESCRIPTOR" ] && ! cmp -s "$REPRO_MODULE_DESCRIPTOR" "$REPRO_TASK_DIR/path-entry.desktop"; then
    printf '描述文件已被其他配置占用，停止安装。\n'
    exit 1
fi
sudo install -d -m 0755 /opt/gtk-path-entry
sudo install -m 0644 "$REPRO_TASK_DIR/libpath-entry.so" /opt/gtk-path-entry/libpath-entry.so
sudo install -m 0644 "$REPRO_TASK_DIR/path-entry.c" /opt/gtk-path-entry/path-entry.c
sudo install -m 0644 "$REPRO_TASK_DIR/user-settings.py" /opt/gtk-path-entry/user-settings.py
sudo install -d -m 0755 "$(dirname "$REPRO_MODULE_LINK")"
sudo install -d -m 0755 "$(dirname "$REPRO_MODULE_DESCRIPTOR")"
sudo ln -sfn /opt/gtk-path-entry/libpath-entry.so "$REPRO_MODULE_LINK"
sudo install -m 0644 "$REPRO_TASK_DIR/path-entry.desktop" "$REPRO_MODULE_DESCRIPTOR"
python3 /opt/gtk-path-entry/user-settings.py install chooser
```

GTK 模块描述文件让 GNOME Settings Daemon 的模块目录监视器发现模块；模块名经 `Gtk/Modules` 传播到 GTK 3 应用。这两个系统文件必不可少，注册作用于此机器的 GNOME 用户；用户 portal 和设置文件只修改当前用户。

用户配置工具会保留现有 GTK 模块列表、主题、注释和 portal 的其他配置，仅完成以下修改：

- 在 `org.gnome.settings-daemon.plugins.xsettings enabled-gtk-modules` 列表中追加 `path-entry`。
- 在 `${XDG_CONFIG_HOME:-~/.config}/gtk-3.0/settings.ini` 的 `gtk-modules` 字符串中追加 `path-entry`。
- 在 `${XDG_CONFIG_HOME:-~/.config}/xdg-desktop-portal/gnome-portals.conf` 中设置 `org.freedesktop.impl.portal.FileChooser=gtk;`。如果原用户文件不存在，基于系统的 GNOME 配置创建，保留其他 portal 路由。

本机最终 portal 配置如下；它是示例，不要用它整段覆盖新机器已有的无关配置：

```ini
[preferred]
org.freedesktop.impl.portal.FileChooser=gtk;
default=gnome;gtk;
org.freedesktop.impl.portal.Secret=gnome-keyring;
```

重启文件选择 portal：

```bash
systemctl --user restart xdg-desktop-portal-gtk.service
systemctl --user restart xdg-desktop-portal.service
systemctl --user is-active xdg-desktop-portal-gtk.service xdg-desktop-portal.service
```

GNOME 46 的 `org.gnome.SettingsDaemon.XSettings.service` 禁止手动启动/停止，直接执行 `systemctl --user restart` 会被拒绝。这里使用模块描述文件的目录监视器；若新会话或应用缓存未更新，关闭并重开相关应用，必要时注销并重新登录。

只设置 `org.gtk.Settings.FileChooser location-mode=filename-entry` 无法完成本需求：已检查的 GTK 3/GTK 4 版本未在打开选择器时按该值恢复路径栏，也不会添加单击目录栏行为。只追加 `enabled-gtk-modules` 也不足以保证当前 GNOME 会话立即更新，不能省略描述文件和系统模块链接。

模块把原路径栏放入接收单击的 GTK 容器，点击后调用原生 `location-popup`，显示并选中路径输入框。Enter 的目录导航、文件选择和错误提示继续由 GTK 处理。点击路径栏中的目录段会切换到编辑模式；需要浏览上级目录时可使用键盘或侧边栏。

## 8. 使 Nautilus 新启动入口生效

完成第 5、6、6.1 节后，已有的 Nautilus 进程必须退出并经启动器重新启动。先等文件复制、移动等任务结束，可查询：

```bash
gdbus call --session --dest org.gnome.Nautilus --object-path /org/gnome/Nautilus \
    --method org.freedesktop.DBus.Properties.Get org.gtk.Application Busy
```

已有进程时返回 `<false>` 才可退出；没有进程时服务查询可能触发激活。可先记录当前窗口目录：

```bash
gdbus call --session --dest org.freedesktop.FileManager1 --object-path /org/freedesktop/FileManager1 \
    --method org.freedesktop.DBus.Properties.Get org.freedesktop.FileManager1 OpenWindowsWithLocations
```

让 ChatGPT 保存这些 URI 并在重启后重新打开，或手动记下需要保留的目录。恢复目录不保证原窗口/标签分组完全一致。随后：

```bash
nautilus --quit || true
gdbus call --session --dest org.freedesktop.DBus --object-path /org/freedesktop/DBus \
    --method org.freedesktop.DBus.NameHasOwner org.gnome.Nautilus
```

本机 `nautilus --quit` 曾在成功退出时返回非零值；应以 `NameHasOwner` 返回 `false` 判断退出，不能只看退出码。若还为 `true`，等退出完成并排查忙碌任务，不要用强制杀进程作为常规步骤。然后从桌面重新打开“文件”，或运行：

```bash
/opt/nautilus-direct-document/launcher.py --new-window
```

自动化恢复目录时，可通过会话 D-Bus 的 `org.freedesktop.Application.Open` 向 `/org/gnome/Nautilus` 传入 URI 列表，由会话服务激活启动器。重启后再次读取 `OpenWindowsWithLocations`，核对记录的目录确实已经恢复；仅取得 D-Bus 服务名不能证明窗口已打开。

关闭此前打开的文件选择窗口，再重新打开。portal 客户端若仍缓存旧后端，重启该应用或注销登录。

## 9. 验收与排查

### 9.1 必须完成的实际操作

在自己创建的临时测试目录中验收，不要用已有文档做删除或覆盖测试。

| 测试 | 应看到的结果 |
| --- | --- |
| Nautilus 文件夹空白处右键 | 主菜单直接有“新建文档”，没有模板子菜单 |
| 点击“新建文档” | 创建 0 字节文件，无 `.txt` 等后缀；原生重命名/重名处理可用 |
| 再次创建 | 按 Nautilus 原生规则处理重名，不覆盖首次文件 |
| 在中文、含空格目录右键打开 Terminator | Terminator 启动；其中 `pwd` 输出对应目录 |
| 选中文件后右键“复制绝对路径”，粘贴到文本框 | 完整绝对路径，包含文件名和后缀，例如 `/home/aaa/bbb/c.urdf` |
| 选中文件夹后复制 | 路径包含选中文件夹名称，不只是父目录 |
| 多选文件/目录后复制 | 每行一个完整路径；中文、空格、`%` 保留原样 |
| 选中符号链接后复制 | 得到链接本身的绝对路径 |
| “打开文件”窗口单击顶部路径栏 | 显示当前完整路径，输入框获得焦点并选中内容 |
| 输入真实目录绝对路径，不加末尾 `/`，按 Enter | 进入该目录，窗口继续用于选择文件 |
| 输入真实文件绝对路径，按 Enter | 原生 Open/Accept 确认该文件，由调用应用接收 |
| 从“最近使用”页面点顶部路径栏再输入目录 | 能进入指定目录 |
| “保存文件”窗口 | 原文件名、保存行为保持正常 |

“选择目录”模式也附加点击编辑行为，但目录确认继续遵循原生选择目录模式；不要把“打开文件”模式的文件确认规则套到选择目录窗口上。

### 9.2 查看模块是否加载

在新启动的普通 GTK 3 进程里检查；不输出整个进程环境：

```bash
python3 - <<'PYCODE'
from pathlib import Path
import gi
gi.require_version('Gtk', '3.0')
from gi.repository import Gtk
Gtk.init([])
print('GTK modules:', Gtk.Settings.get_default().get_property('gtk-modules'))
print('path-entry loaded:', '/opt/gtk-path-entry/libpath-entry.so' in Path('/proc/self/maps').read_text())
PYCODE
```

预期模块列表包含 `path-entry`，且 `path-entry loaded: True`。对 portal 选择器，还应检查 `xdg-desktop-portal-gtk.service` 的 `MainPID` 对应 `/proc/<PID>/maps` 是否包含该库；调用一次文件选择窗口后再核对。

Nautilus 验证时查看实际会话进程的 `/proc/<PID>/environ` 中是否有本方案的 `G_RESOURCE_OVERLAYS`，只输出这一项即可。库、配置文件存在但菜单未变时，重点检查是否仍是旧进程、D-Bus 两个启动入口是否遗漏，以及是否直接绕过启动器打开了 `/usr/bin/nautilus`。

### 9.3 原机器已经完成的验证

原机器的隔离 Xvfb/D-Bus 测试已通过：真实 Nautilus 一级菜单创建了无后缀空文件；真实 Terminator shell 的工作目录为包含中文和空格的选中目录；真实 GTK 3 窗口单击进入编辑、目录导航、文件 Accept、“最近使用”以及保存模式回归测试通过。安装库与已测试库一致，GTK portal 后端加载了模块。

本文件新增的配置/恢复脚本另在隔离临时 XDG 目录、内存 GSettings 中验证过：重复安装不覆盖首次备份，已有模板保留，GTK 主题/模块/注释及其他 portal 路由保留，能恢复原用户文件和原终端选择。本文件的源码块已重新提取，并通过 Python 语法检查、GTK 编译及全部 Bash 命令块的语法检查。换机仍须执行 9.1 的实际验收，因为新版本、客户端和会话缓存可能不同。

2026-10-13 新增扩展也已在隔离 Xvfb/D-Bus 中验证：真实 Nautilus 选中文件后的菜单出现该项，点击后另一进程读取剪贴板，得到包含中文、空格和 `%` 的完整文件路径；文件夹、多选及符号链接的剪贴板结果均通过验证。安装采用的软链接入口也在真实 Nautilus 中加载成功。

### 9.4 清理测试后台和临时文件

结束自己启动的 Xvfb、测试 D-Bus、测试 Nautilus/Terminator 和测试应用，移除隔离运行目录及测试文件。只清理本次测试创建的进程，不要终止用户原有窗口或正常 portal 服务。

临时构建目录中的文件都已安装且验收完毕后，可清理：

```bash
case "$REPRO_TASK_DIR" in
    /tmp/ubuntu-file-dialog-reproduce.*) rm -rf -- "$REPRO_TASK_DIR" ;;
    *) printf '临时目录变量不正确，不执行删除。\n' ;;
esac
```

保留 `/opt` 下源码/工具、实际模板和 XDG 状态目录中的备份，它们用于运行与恢复。

## 10. 恢复原行为

先等文件任务结束、关闭文件选择窗口。

如果安装了复制路径扩展，先移除它的用户入口。只删除指向本方案源码的软链接：

```bash
python3 - <<'PYCODE'
from pathlib import Path
import os
share = Path(os.environ.get('XDG_DATA_HOME', str(Path.home() / '.local/share')))
entry = share / 'nautilus-python/extensions/copy_absolute_path.py'
if entry.is_symlink() and entry.readlink() == Path('/opt/nautilus-direct-document/copy_absolute_path.py'):
    entry.unlink()
    print('已移除复制绝对路径菜单入口，重启 Nautilus 后生效。')
elif entry.exists() or entry.is_symlink():
    raise SystemExit('入口后来有修改，已保留，请人工检查。')
PYCODE
```

源码可暂时保留在 `/opt`，不影响关闭该扩展。只恢复此功能时移除这个入口，然后按第 8 节重启 Nautilus即可；无需修改另外三项功能。

恢复其余定制的当前用户配置（普通用户执行）：

```bash
python3 /opt/nautilus-direct-document/configure-nautilus.py restore
python3 /opt/gtk-path-entry/user-settings.py restore terminal
python3 /opt/gtk-path-entry/user-settings.py restore chooser
```

备份位置：

- Nautilus：`/opt/nautilus-direct-document/configuration-backup.json`。
- Terminator：`${XDG_STATE_HOME:-~/.local/state}/ubuntu-file-dialog-customization/terminal-backup.json`。
- GTK/portal：同一状态目录中的 `chooser-backup.json`。
- 初始软件包状态：同一状态目录中的 `packages-before.txt`。

恢复脚本只恢复仍与本次安装内容一致的文件或设置；若后来手动改过，会提示并保留，需人工合并。原本没有显式值的 GSettings 会 `reset`。恢复后再次安装时先核对并归档旧备份；不要删除唯一的原始备份来掩盖冲突。

然后注销 GTK 模块的系统注册。确认描述文件内容与第 7 节一致、链接仍指向本方案，才删除：

```bash
module_link="/usr/lib/$(dpkg-architecture -qDEB_HOST_MULTIARCH)/gtk-3.0/modules/libpath-entry.so"
# 先检查这两个文件确实属于本方案。
cat /usr/lib/gnome-settings-daemon-3.0/gtk-modules/path-entry.desktop
readlink "$module_link"
# 核对后执行：
sudo rm -- /usr/lib/gnome-settings-daemon-3.0/gtk-modules/path-entry.desktop "$module_link"
systemctl --user restart xdg-desktop-portal-gtk.service
systemctl --user restart xdg-desktop-portal.service
```

只恢复 GSettings 或 `settings.ini` 不能彻底停用模块，系统描述文件仍会注册它；已加载模块的进程还需要退出，通常注销再登录最直接。

最后按 `packages-before.txt` 决定是否恢复原终端菜单。如果原机器装有 `nautilus-extension-gnome-terminal`，模拟后恢复它并移除新扩展：

```bash
sudo apt-get -s install nautilus-extension-gnome-terminal nautilus-extension-any-terminal-
sudo apt-get install nautilus-extension-gnome-terminal nautilus-extension-any-terminal-
```

若原来就没有 GNOME Terminal 菜单扩展，不要额外装它。恢复终端设置必须在卸载新扩展之前执行，以便 schema 仍可读取。Terminator 应用和新装依赖是否卸载依据实际使用需要决定，不做无关自动清理。

注销并重新登录，验证恢复。`/opt` 的软件文件可以暂时保留用于排查；若要彻底删除，先归档备份和源码，再移除本方案创建的两个目录。

## 11. 参考与维护

- [GNOME 模板文档](https://teams.pages.gitlab.gnome.org/websites/help.gnome.org/gnome-help/files-templates.html)：普通模板机制的背景。
- [GLib GResource 文档](https://docs.gtk.org/gio/struct.Resource.html)：`G_RESOURCE_OVERLAYS` 的机制。
- [Nautilus 46.4 文件视图源码](https://raw.githubusercontent.com/GNOME/nautilus/46.4/src/nautilus-files-view.c)：模板动作名与菜单更新。
- [nautilus-open-any-terminal 上游](https://github.com/Stunkymonkey/nautilus-open-any-terminal)：扩展来源；本文固定版本 0.8.3。
- [GTK 3.24.41 文件选择器源码](https://raw.githubusercontent.com/GNOME/gtk/3.24.41/gtk/gtkfilechooserwidget.c)：原生路径输入与确认行为。
- [GNOME 46 GTK 模块注册源码](https://raw.githubusercontent.com/GNOME/gnome-settings-daemon/46.0/plugins/xsettings/gsd-xsettings-gtk.c)：描述文件发现及模块发布。
- [portal 路由配置文档](https://flatpak.github.io/xdg-desktop-portal/docs/portals.conf.html)：按接口指定文件选择后端。

下面五段是本文件的完整配套源码。A、C、E 来自已通过实际 UI 测试的实现；B、D 为换机复现整理的可移植配置及恢复工具。后续更新代码时，要同步更新安装说明并重新验证。

## 附录 A：Nautilus 启动器

<!-- source-file: launcher.py -->
```python
#!/usr/bin/python3
"""Expose Nautilus's native blank-document action in the main context menu."""

import os
from pathlib import Path
import subprocess
import sys
import tempfile
import xml.etree.ElementTree as ET

import gi

gi.require_version("Gio", "2.0")
from gi.repository import Gio, GLib


RESOURCE = "/org/gnome/nautilus/ui/nautilus-files-view-context-menus.ui"


def make_menu(source, template):
    root = ET.fromstring(source)
    background = root.find("./menu[@id='background-menu']")
    if background is None:
        raise RuntimeError("Nautilus menu layout has changed")
    matches = [
        item for item in background.findall("item")
        if item.find("./attribute[@name='nautilus-menu-item']") is not None
        and item.find("./attribute[@name='nautilus-menu-item']").text == "templates-submenu"
    ]
    if len(matches) != 1:
        raise RuntimeError("Nautilus template menu cannot be identified")
    item = matches[0]
    link = item.find("./link[@name='submenu'][@id='templates-submenu']")
    if link is None:
        raise RuntimeError("Nautilus template menu layout has changed")
    item.remove(link)
    # Nautilus still populates this model internally; retain it without a menu link.
    ET.SubElement(root, "menu", {"id": "templates-submenu"})
    uri = Gio.File.new_for_path(str(template)).get_uri()
    action = "view.template_" + GLib.uri_escape_string(uri, None, True)
    ET.SubElement(item, "attribute", {"name": "action"}).text = action
    return ET.tostring(root, encoding="utf-8", xml_declaration=True)


def main():
    try:
        directory = Path(subprocess.check_output(
            ["/usr/bin/xdg-user-dir", "TEMPLATES"], text=True
        ).strip())
        if not directory.is_absolute() or directory == Path.home():
            raise RuntimeError("A dedicated Templates directory is required")
        directory.mkdir(parents=True, exist_ok=True)
        template = directory / "新建文档"
        try:
            with template.open("xb"):
                pass
        except FileExistsError:
            if not template.is_file() or template.stat().st_size != 0:
                raise RuntimeError("The blank-document template is no longer empty")
        source = subprocess.check_output([
            "/usr/bin/gresource", "extract", "/usr/bin/nautilus", RESOURCE
        ])
        xml = make_menu(source, template)
        runtime = os.environ.get("XDG_RUNTIME_DIR")
        if runtime:
            overlay = Path(runtime) / "nautilus-direct-document"
            overlay.mkdir(mode=0o700, exist_ok=True)
        else:
            overlay = Path(tempfile.mkdtemp(prefix="nautilus-direct-document-"))
        ui = overlay / Path(RESOURCE).name
        temporary = overlay / (ui.name + "." + str(os.getpid()))
        temporary.write_bytes(xml)
        temporary.replace(ui)
        mapping = "/org/gnome/nautilus/ui=" + str(overlay)
        previous = os.environ.get("G_RESOURCE_OVERLAYS")
        os.environ["G_RESOURCE_OVERLAYS"] = mapping + (":" + previous if previous else "")
    except Exception as error:
        # A future incompatible Nautilus upgrade must not prevent Files opening.
        print("nautilus-direct-document:", error, file=sys.stderr)
    os.execv("/usr/bin/nautilus", ["/usr/bin/nautilus", *sys.argv[1:]])


if __name__ == "__main__":
    main()
```

## 附录 B：Nautilus 用户入口配置与恢复

<!-- source-file: configure-nautilus.py -->
```python
#!/usr/bin/python3
"""Install or restore the per-user Nautilus entry points."""
import base64
import json
import os
from pathlib import Path
import subprocess
import sys

TARGET = Path('/opt/nautilus-direct-document')


def write(path, data, mode=0o644):
    path.parent.mkdir(parents=True, exist_ok=True)
    temporary = path.with_name(path.name + '.tmp-' + str(os.getpid()))
    temporary.write_bytes(data)
    temporary.chmod(mode)
    temporary.replace(path)


def encode(data):
    return base64.b64encode(data).decode() if data is not None else None


def current(path):
    return path.read_bytes() if path.exists() else None


def main():
    if os.geteuid() == 0:
        raise SystemExit('请用当前桌面用户运行，不要使用 sudo。')
    state_path = TARGET / 'configuration-backup.json'
    if sys.argv[1:] == ['restore']:
        state = json.loads(state_path.read_text())
        for name, entry in state['files'].items():
            path = Path(name)
            if encode(current(path)) != entry['installed']:
                print('保留后来修改的文件：', path)
                continue
            if entry['original'] is None:
                path.unlink(missing_ok=True)
            else:
                write(path, base64.b64decode(entry['original']), entry.get('mode', 0o644))
        print('已恢复可安全恢复的文件，请重新启动文件管理器。')
    else:
        if sys.argv[1:]:
            raise SystemExit('用法：configure-nautilus.py [restore]')
        directory = Path(subprocess.check_output(['xdg-user-dir', 'TEMPLATES'], text=True).strip())
        if not directory.is_absolute() or directory == Path.home():
            raise SystemExit('请先配置独立的 XDG 模板目录。')
        template = directory / '新建文档'
        if template.exists() and (not template.is_file() or template.stat().st_size != 0):
            raise SystemExit('已有“新建文档”模板不是空文件，请先保存并重命名。')
        share = Path(os.environ.get('XDG_DATA_HOME', str(Path.home() / '.local/share')))
        launcher = TARGET / 'launcher.py'
        desktop = Path('/usr/share/applications/org.gnome.Nautilus.desktop').read_text()
        lines = []
        changed = 0
        for line in desktop.splitlines():
            if line.startswith('Exec=nautilus') and line[len('Exec=nautilus'):len('Exec=nautilus')+1] in ('', ' '):
                line = 'Exec=' + str(launcher) + line[len('Exec=nautilus'):]
                changed += 1
            lines.append(line)
        if not changed:
            raise SystemExit('Nautilus desktop 文件的启动格式已变化，需要适配。')
        changes = {share / 'applications/org.gnome.Nautilus.desktop': ('\n'.join(lines)+'\n').encode(), template: b''}
        for name in ['org.gnome.Nautilus', 'org.freedesktop.FileManager1']:
            changes[share / ('dbus-1/services/' + name + '.service')] = (
                '[D-BUS Service]\nName=' + name + '\nExec=' + str(launcher) + ' --gapplication-service\n'
            ).encode()
        state = json.loads(state_path.read_text()) if state_path.exists() else {'files': {}}
        for path, data in changes.items():
            if path.is_symlink():
                raise SystemExit('配置文件为符号链接，需先适配，未覆盖：' + str(path))
            name = str(path)
            if name not in state['files']:
                state['files'][name] = {'original': encode(current(path)), 'mode': path.stat().st_mode & 0o777 if path.exists() else 0o644}
            elif encode(current(path)) != state['files'][name]['installed']:
                raise SystemExit('配置后来有修改，先检查再安装：' + name)
            state['files'][name]['installed'] = encode(data)
        write(state_path, json.dumps(state, ensure_ascii=False, indent=2).encode(), 0o600)
        for path, data in changes.items():
            write(path, data)
        print('桌面、两个 D-Bus 入口和空白模板已配置。')
    share = Path(os.environ.get('XDG_DATA_HOME', str(Path.home() / '.local/share')))
    subprocess.run(['update-desktop-database', str(share / 'applications')], check=False)


if __name__ == '__main__':
    main()
```

## 附录 C：GTK 3 路径栏模块

<!-- source-file: path-entry.c -->
```c
/* Make GTK 3 open-dialog breadcrumbs switch to the native location entry. */
#define _GNU_SOURCE
#include <dlfcn.h>
#include <glib-object.h>
#include <gmodule.h>

/* Opaque public GTK/GDK interfaces; runtime libraries are already installed. */
typedef struct _GtkWidget GtkWidget;
typedef struct _GdkEvent GdkEvent;
extern GType gtk_widget_get_type(void);
extern GType gtk_file_chooser_widget_get_type(void);
extern guint gtk_get_major_version(void);
extern GtkWidget *gtk_widget_get_parent(GtkWidget *);
extern gboolean gtk_widget_in_destruction(GtkWidget *);
extern gboolean gtk_widget_get_visible(GtkWidget *);
extern const gchar *gtk_buildable_get_name(gpointer);
extern GList *gtk_container_get_children(gpointer);
extern GType gtk_container_get_type(void);
extern GType gtk_box_get_type(void);
extern void gtk_container_remove(gpointer, GtkWidget *);
extern void gtk_container_add(gpointer, GtkWidget *);
extern void gtk_container_child_get(gpointer, GtkWidget *, const gchar *, ...);
extern void gtk_box_query_child_packing(gpointer, GtkWidget *, gboolean *, gboolean *, guint *, gint *);
extern void gtk_box_pack_start(gpointer, GtkWidget *, gboolean, gboolean, guint);
extern void gtk_box_pack_end(gpointer, GtkWidget *, gboolean, gboolean, guint);
extern void gtk_box_reorder_child(gpointer, GtkWidget *, gint);
extern GtkWidget *gtk_event_box_new(void);
extern void gtk_event_box_set_above_child(gpointer, gboolean);
extern void gtk_event_box_set_visible_window(gpointer, gboolean);
extern gboolean gdk_event_get_button(const GdkEvent *, guint *);
extern gint gtk_file_chooser_get_action(gpointer);
extern gchar *gtk_file_chooser_get_current_folder(gpointer);
extern void gtk_widget_show(GtkWidget *);
extern void gtk_widget_grab_focus(GtkWidget *);
extern void gtk_widget_add_events(GtkWidget *, gint);
extern void gtk_editable_select_region(gpointer, gint, gint);
extern GList *gtk_window_list_toplevels(void);

static gboolean is_type(gpointer object, GType type)
{
    return object && G_TYPE_CHECK_INSTANCE_TYPE((GTypeInstance *)object, type);
}

static GtkWidget *find_widget(GtkWidget *root, const gchar *name, gboolean by_type)
{
    const gchar *candidate = by_type ? G_OBJECT_TYPE_NAME(root) : gtk_buildable_get_name(root);
    if (g_strcmp0(candidate, name) == 0)
        return root;
    if (!is_type(root, gtk_container_get_type()))
        return NULL;
    GList *children = gtk_container_get_children(root);
    GtkWidget *match = NULL;
    for (GList *node = children; node && !match; node = node->next)
        match = find_widget(node->data, name, by_type);
    g_list_free(children);
    return match;
}

static gboolean edit_path(GtkWidget *event_box, GdkEvent *event, GtkWidget *chooser)
{
    (void)event_box;
    guint button = 0;
    if (!gdk_event_get_button(event, &button) || button != 1)
        return FALSE;
    gint action = gtk_file_chooser_get_action(chooser);
    if (action != 0 && action != 2) /* OPEN / SELECT_FOLDER */
        return FALSE;
    gchar *folder = gtk_file_chooser_get_current_folder(chooser);
    if (!folder)
        folder = g_strdup(g_get_home_dir());
    gchar *path = g_strconcat(folder, g_str_has_suffix(folder, "/") ? "" : "/", NULL);
    g_signal_emit_by_name(chooser, "location-popup", path);
    GtkWidget *entry = find_widget(chooser, "GtkFileChooserEntry", TRUE);
    if (entry && gtk_widget_get_visible(entry)) {
        gtk_widget_grab_focus(entry);
        gtk_editable_select_region(entry, 0, -1);
    }
    g_free(path);
    g_free(folder);
    return TRUE;
}

static gboolean attach_path_click(gpointer data)
{
    GtkWidget *chooser = data;
    if (gtk_widget_in_destruction(chooser))
        return G_SOURCE_REMOVE;
    gint action = gtk_file_chooser_get_action(chooser);
    if (action != 0 && action != 2)
        return G_SOURCE_REMOVE;
    GtkWidget *bar = find_widget(chooser, "browse_path_bar", FALSE);
    if (!bar)
        return G_SOURCE_REMOVE;
    GtkWidget *parent = gtk_widget_get_parent(bar);
    if (!is_type(parent, gtk_box_get_type()))
        return G_SOURCE_REMOVE;
    gboolean expand, fill;
    guint padding;
    gint pack_type, position;
    gtk_box_query_child_packing(parent, bar, &expand, &fill, &padding, &pack_type);
    gtk_container_child_get(parent, bar, "position", &position, NULL);
    GtkWidget *event_box = gtk_event_box_new();
    gtk_event_box_set_above_child(event_box, TRUE);
    gtk_event_box_set_visible_window(event_box, FALSE);
    gtk_widget_add_events(event_box, 1 << 8); /* GDK_BUTTON_PRESS_MASK */
    g_object_ref(bar);
    gtk_container_remove(parent, bar);
    gtk_container_add(event_box, bar);
    g_object_unref(bar);
    if (pack_type == 1)
        gtk_box_pack_end(parent, event_box, expand, fill, padding);
    else
        gtk_box_pack_start(parent, event_box, expand, fill, padding);
    gtk_box_reorder_child(parent, event_box, position);
    g_signal_connect_object(event_box, "button-press-event", G_CALLBACK(edit_path),
                            G_OBJECT(chooser), 0);
    gtk_widget_show(event_box);
    return G_SOURCE_REMOVE;
}

static void schedule_chooser(GtkWidget *widget)
{
    if (!is_type(widget, gtk_file_chooser_widget_get_type()))
        return;
    if (g_object_get_data(G_OBJECT(widget), "codex-path-entry-attached"))
        return;
    g_object_set_data(G_OBJECT(widget), "codex-path-entry-attached", GINT_TO_POINTER(1));
    g_idle_add_full(G_PRIORITY_DEFAULT_IDLE, attach_path_click,
                    g_object_ref(widget), g_object_unref);
}

static gboolean mapped(GSignalInvocationHint *hint, guint count,
                       const GValue *values, gpointer data)
{
    (void)hint;
    (void)data;
    if (count)
        schedule_chooser(g_value_get_object(values));
    return TRUE;
}

static void scan_existing(GtkWidget *widget)
{
    schedule_chooser(widget);
    if (!is_type(widget, gtk_container_get_type()))
        return;
    GList *children = gtk_container_get_children(widget);
    for (GList *node = children; node; node = node->next)
        scan_existing(node->data);
    g_list_free(children);
}

G_MODULE_EXPORT void gtk_module_init(gint *argc, gchar ***argv)
{
    (void)argc;
    (void)argv;
    if (gtk_get_major_version() != 3)
        return;
    /* Emission hooks must remain valid if Gtk/Modules changes later. */
    Dl_info info;
    if (dladdr((gpointer)gtk_module_init, &info) && info.dli_fname) {
        GModule *self = g_module_open(info.dli_fname, G_MODULE_BIND_LOCAL);
        if (self) {
            g_module_make_resident(self);
            g_module_close(self);
        }
    }
    gpointer widget_class = g_type_class_ref(gtk_widget_get_type());
    guint signal = g_signal_lookup("map", gtk_widget_get_type());
    if (signal)
        g_signal_add_emission_hook(signal, 0, mapped, NULL, NULL);
    g_type_class_unref(widget_class);
    GList *windows = gtk_window_list_toplevels();
    for (GList *node = windows; node; node = node->next)
        scan_existing(node->data);
    g_list_free(windows);
}
```

## 附录 D：Terminator、GTK 和 portal 用户配置与恢复

<!-- source-file: user-settings.py -->
```python
#!/usr/bin/python3
"""Configure/restore Terminator and GTK chooser settings for the desktop user."""
import base64
import json
import os
from pathlib import Path
import sys
from gi.repository import Gio, GLib


def encode(data):
    return base64.b64encode(data).decode() if data is not None else None


def current(path):
    return path.read_bytes() if path.exists() else None


def write(path, data, mode=0o644):
    path.parent.mkdir(parents=True, exist_ok=True)
    temporary = path.with_name(path.name + '.tmp-' + str(os.getpid()))
    temporary.write_bytes(data)
    temporary.chmod(mode)
    temporary.replace(path)


def setting(schema, key):
    source = Gio.SettingsSchemaSource.get_default()
    if source.lookup(schema, True) is None:
        raise SystemExit('缺少 GSettings schema：' + schema)
    settings = Gio.Settings.new(schema)
    value = settings.get_user_value(key)
    return settings, value.print_(True) if value is not None else None


def main():
    if os.geteuid() == 0:
        raise SystemExit('请用当前桌面用户运行，不要使用 sudo。')
    if len(sys.argv) != 3 or sys.argv[1] not in ('install', 'restore') or sys.argv[2] not in ('terminal', 'chooser'):
        raise SystemExit('用法：user-settings.py install|restore terminal|chooser')
    action, part = sys.argv[1:]
    state_dir = Path(os.environ.get('XDG_STATE_HOME', str(Path.home() / '.local/state'))) / 'ubuntu-file-dialog-customization'
    state_path = state_dir / (part + '-backup.json')
    if action == 'restore':
        state = json.loads(state_path.read_text())
        for name, entry in state['files'].items():
            path = Path(name)
            if encode(current(path)) != entry['installed']:
                print('保留后来修改的文件：', path)
                continue
            if entry['original'] is None:
                path.unlink(missing_ok=True)
            else:
                write(path, base64.b64decode(entry['original']), entry.get('mode', 0o644))
        for entry in state['settings'].values():
            settings, actual = setting(entry['schema'], entry['key'])
            if actual != entry['installed']:
                print('保留后来修改的设置：', entry['schema'], entry['key'])
                continue
            if entry['original'] is None:
                settings.reset(entry['key'])
            else:
                settings.set_value(entry['key'], GLib.Variant.parse(None, entry['original'], None, None))
        Gio.Settings.sync()
        print('已恢复可安全恢复的用户配置：', part)
        return
    changes = {}
    settings_changes = []
    if part == 'terminal':
        schema = 'com.github.stunkymonkey.nautilus-open-any-terminal'
        settings_changes = [(schema, 'terminal', GLib.Variant('s', 'terminator')),
                            (schema, 'use-generic-terminal-name', GLib.Variant('b', False)),
                            (schema, 'new-tab', GLib.Variant('b', False))]
    else:
        config = Path(os.environ.get('XDG_CONFIG_HOME', str(Path.home() / '.config')))
        gtk_path = config / 'gtk-3.0/settings.ini'
        gtk = GLib.KeyFile()
        if gtk_path.exists():
            gtk.load_from_file(str(gtk_path), GLib.KeyFileFlags.KEEP_COMMENTS)
        try:
            modules = gtk.get_string('Settings', 'gtk-modules').split(':')
        except GLib.Error:
            modules = []
        modules = [item for item in modules if item]
        if 'path-entry' not in modules:
            modules.append('path-entry')
        gtk.set_string('Settings', 'gtk-modules', ':'.join(modules))
        changes[gtk_path] = gtk.to_data()[0].encode()
        portal_path = config / 'xdg-desktop-portal/gnome-portals.conf'
        portal = GLib.KeyFile()
        source = portal_path if portal_path.exists() else Path('/usr/share/xdg-desktop-portal/gnome-portals.conf')
        if not source.exists():
            raise SystemExit('未找到 GNOME portal 配置，需要先确认桌面和 portal 版本。')
        portal.load_from_file(str(source), GLib.KeyFileFlags.KEEP_COMMENTS)
        portal.set_string('preferred', 'org.freedesktop.impl.portal.FileChooser', 'gtk;')
        changes[portal_path] = portal.to_data()[0].encode()
        schema = 'org.gnome.settings-daemon.plugins.xsettings'
        settings, _ = setting(schema, 'enabled-gtk-modules')
        enabled = settings.get_strv('enabled-gtk-modules')
        if 'path-entry' not in enabled:
            enabled.append('path-entry')
        settings_changes = [(schema, 'enabled-gtk-modules', GLib.Variant('as', enabled))]
    state = json.loads(state_path.read_text()) if state_path.exists() else {'files': {}, 'settings': {}}
    for path, data in changes.items():
        if path.is_symlink():
            raise SystemExit('配置文件为符号链接，需先适配，未覆盖：' + str(path))
        name = str(path)
        actual = encode(current(path))
        if name not in state['files']:
            state['files'][name] = {'original': actual, 'mode': path.stat().st_mode & 0o777 if path.exists() else 0o644}
        elif actual != state['files'][name]['installed']:
            raise SystemExit('配置后来有修改，先检查再安装：' + name)
        state['files'][name]['installed'] = encode(data)
    for schema, key, value in settings_changes:
        _, actual = setting(schema, key)
        name = schema + '/' + key
        if name not in state['settings']:
            state['settings'][name] = {'schema': schema, 'key': key, 'original': actual}
        elif actual != state['settings'][name]['installed']:
            raise SystemExit('设置后来有修改，先检查再安装：' + name)
        state['settings'][name]['installed'] = value.print_(True)
    write(state_path, json.dumps(state, ensure_ascii=False, indent=2).encode(), 0o600)
    for path, data in changes.items():
        write(path, data)
    for schema, key, value in settings_changes:
        settings, _ = setting(schema, key)
        if not settings.set_value(key, value):
            raise SystemExit('无法写入设置：' + schema + '/' + key)
    Gio.Settings.sync()
    print('已配置并保留首次安装前的用户状态：', part)


if __name__ == '__main__':
    main()
```

## 附录 E：复制选中对象的绝对路径

<!-- source-file: copy_absolute_path.py -->
```python
"""Copy selected Nautilus files' own absolute local paths as plain text."""
import gi

gi.require_version('Nautilus', '4.0')
gi.require_version('Gdk', '4.0')
from gi.repository import Gdk, GObject, Nautilus


class CopyAbsolutePath(GObject.GObject, Nautilus.MenuProvider):
    def get_file_items(self, files):
        if not files:
            return []
        paths = []
        for file_info in files:
            path = file_info.get_location().get_path()
            if path is None:
                # A remote URI has no absolute local filesystem path.
                return []
            paths.append(path)
        item = Nautilus.MenuItem(
            name='CopyAbsolutePath::Copy',
            label='复制绝对路径',
            tip='复制选中文件或文件夹本身的完整路径；多选时每行一个',
        )
        item.connect('activate', self._copy_paths, '\n'.join(paths))
        return [item]

    def get_background_items(self, current_folder):
        return []

    def _copy_paths(self, item, text):
        display = Gdk.Display.get_default()
        if display is not None:
            display.get_clipboard().set(text)
```
