# Ubuntu 换机配置清单

记录时间：2026-09-23。来源：本机 Ubuntu 24.04.4 LTS、GNOME Shell 46、Wayland 会话（ThinkPad E14 Gen 7）。

本文只记录查到的、相对本机 Ubuntu/GNOME **有效默认值**有差异的设置，以及明确添加的用户/系统配置。应用的最近使用记录、窗口大小、历史缓存、未改变默认行为的显式设置均已略去。新电脑若使用不同 Ubuntu/GNOME 版本，设置名称可能变化。

## 1. 桌面外观与操作

| 项目             | 本机设置                                                                                                                                                 | 换机时如何重做                                          |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| 左上角“活动”按钮      | 用 **Just Perfection** 扩展隐藏；`activities-button=false`                                                                                                 | 安装并启用 Just Perfection，在其设置中关闭“Activities Button” |
| 概览中的顶栏         | Just Perfection：`panel-in-overview=true`                                                                                                             | 在扩展设置中打开相应选项                                     |
| 扩展提示           | Just Perfection：`support-notifier-type=0`（扩展默认 1）                                                                                                    | 如需完全一致，在扩展设置中调整                                  |
| Ubuntu Dock 位置 | 屏幕**底部**，而非默认左侧；`dock-position='BOTTOM'`                                                                                                             | 设置 → Ubuntu 桌面 → Dock → 屏幕位置：底部                  |
| Dock 行为        | 非固定（自动隐藏）；多屏显示；图标最大 28 px；不显示垃圾桶和已挂载磁盘                                                                                                               | 设置 → Ubuntu 桌面 → Dock；详见下方命令                     |
| 桌面“主目录”图标      | 关闭；DING 扩展的 `show-home=false`                                                                                                                        | 桌面图标设置中隐藏主目录                                     |
| 深色与配色          | `prefer-dark`；GTK 主题 `Yaru-blue-dark`；图标 `Yaru-blue`                                                                                                 | 设置 → 外观：深色、蓝色强调色                                 |
| 顶栏电量百分比        | 显示；`show-battery-percentage=true`                                                                                                                    | 设置 → 电源：显示电量百分比                                  |
| 桌面壁纸           | `Fuwafuwa_nanbatto_san_by_amaral` 的明/暗两张壁纸；背景基色、次色均为 `#000000`                                                                                       | 设置 → 外观中选择对应 Ubuntu 壁纸                           |
| 鼠标指针速度         | `-0.82426778242677823`（默认 0）                                                                                                                         | 设置 → 鼠标和触摸板；精确值见命令                               |
| 文件管理器视图        | Nautilus 默认“列表视图”                                                                                                                                    | 文件管理器 → 首选项 → 默认视图                               |
| 应用网格           | 自建“编程”“办公”“Preferences”应用文件夹并调整过排列                                                                                                                   | 若仍需要，手动重建；应用网格位置依赖安装的软件                          |
| 平铺窗口           | Tiling Assistant 有自定义设置：`tiling-popup-all-workspace=true`、提示色 `rgb(211,70,21)`；GNOME 原生 `edge-tiling=false`，原生 `Super+Left` / `Super+Right` 平铺快捷键已清空 | 启用 Ubuntu Tiling Assistant 后在扩展设置中核对             |

本机启用的非 Ubuntu 标配扩展：**Just Perfection**（`just-perfection-desktop@just-perfection`）和 **Chinese Calendar**（`chinese-calendar@tigertall`）。Chinese Calendar 设为 `festival-region='CN'`，且 `show-lunar-in-panel=false`；它的默认值分别是 `auto`、`true`。该扩展当前来自 `/opt/chinese-calendar`。

在安装相应扩展后，可用下面的命令恢复主要 GNOME 设置（只运行需要的行）：

```bash
gsettings set org.gnome.shell.extensions.dash-to-dock dock-position 'BOTTOM'
gsettings set org.gnome.shell.extensions.dash-to-dock dock-fixed false
gsettings set org.gnome.shell.extensions.dash-to-dock multi-monitor true
gsettings set org.gnome.shell.extensions.dash-to-dock dash-max-icon-size 28
gsettings set org.gnome.shell.extensions.dash-to-dock show-trash false
gsettings set org.gnome.shell.extensions.dash-to-dock show-mounts false
gsettings set org.gnome.shell.extensions.ding show-home false
gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'
gsettings set org.gnome.desktop.interface gtk-theme 'Yaru-blue-dark'
gsettings set org.gnome.desktop.interface icon-theme 'Yaru-blue'
gsettings set org.gnome.desktop.interface show-battery-percentage true
gsettings set org.gnome.desktop.peripherals.mouse speed -0.82426778242677823
gsettings set org.gnome.nautilus.preferences default-folder-viewer 'list-view'
gsettings set org.gnome.calculator button-mode 'advanced'
```

Just Perfection 的 schema 随扩展安装，不一定在系统全局 schema 目录中；建议用扩展管理器界面设置。若安装位置与本机相同，可运行：

```bash
gsettings --schemadir "$HOME/.local/share/gnome-shell/extensions/just-perfection-desktop@just-perfection/schemas" set org.gnome.shell.extensions.just-perfection activities-button false
```

## 2. 电源与屏幕空闲

| 项目       | 本机非默认值                                                                | 恢复方式                                                   |
| -------- | --------------------------------------------------------------------- | ------------------------------------------------------ |
| 电源模式     | 当前为 `performance`（GNOME 也记住上次选“性能”）                                   | 设置 → 电源 → 电源模式：性能；或 `powerprofilesctl set performance` |
| 插电空闲后的动作 | `sleep-inactive-ac-type='nothing'`，因此插电空闲时不自动挂起                       | 设置 → 电源中关闭插电自动挂起；或下方命令                                 |
| 插电空闲超时配置 | `sleep-inactive-ac-timeout=3600` 秒；动作已设为 `nothing`，此值目前不触发挂起          | 仅追求数值一致时恢复                                             |
| 自动熄屏     | `org.gnome.desktop.session idle-delay=0`，即关闭自动熄屏；Ubuntu 的有效默认值为 300 秒 | 设置 → 电源 → 屏幕空白：从不；或下方命令                                |

```bash
powerprofilesctl set performance
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-timeout 3600
gsettings set org.gnome.desktop.session idle-delay 0
```

性能模式在这台机器上对应当前 `platform_profile=performance`、CPU 能效偏好 `performance`。这是**电源模式的当前效果**，无需另行写入 `/sys`。本机启动的是 `linux-image-realtime-hwe-24.04` 实时内核（当前 7.0.0-31-realtime），这也不是普通 Ubuntu 桌面默认内核；只有新机器确实需要实时内核时再安装。当前 **Ubuntu Pro** 已启用 `realtime-kernel`、`esm-apps`、`esm-infra` 服务；新机要使用相同实时内核，需要先关联自己的 Ubuntu Pro 订阅。

## 3. 常用用户目录统一放到 `/data`

本机 XDG 目录映射写在 `~/.config/user-dirs.dirs`：

| 类型        | 路径                |
| --------- | ----------------- |
| Desktop   | `/data/Desktop`   |
| Downloads | `/data/Downloads` |
| Documents | `/data/Documents` |
| Music     | `/data/Music`     |
| Pictures  | `/data/Pictures`  |
| Videos    | `/data/Videos`    |
| Templates | `/data/Templates` |
| Public    | `/data/Public`    |

**关键点：** 本机 `/data` 只是根分区 `/` 上的普通目录，未单独挂载分区。XDG 映射只改变“默认目录”的指向，不会自动移动旧文件；本机另有 `~/Documents`，它与 `/data/Documents` 是不同目录。

文件管理器还有两项非默认改动：`~/.hidden` 中列出 `Documents` 和 `snap`，因此文件管理器在主目录隐藏这两个条目；`~/.config/gtk-3.0/bookmarks` 把 `/data/Documents`、`/data/Music`、`/data/Pictures`、`/data/Videos`、`/data/Downloads` 加入侧栏书签。这与 XDG 目录映射是三处独立配置，换机时可分别恢复。

新电脑上先创建目录，再设映射：

```bash
sudo install -d -o "$USER" -g "$USER" /data /data/Desktop /data/Downloads /data/Documents /data/Music /data/Pictures /data/Videos /data/Templates /data/Public
xdg-user-dirs-update --set DESKTOP /data/Desktop
xdg-user-dirs-update --set DOWNLOAD /data/Downloads
xdg-user-dirs-update --set DOCUMENTS /data/Documents
xdg-user-dirs-update --set MUSIC /data/Music
xdg-user-dirs-update --set PICTURES /data/Pictures
xdg-user-dirs-update --set VIDEOS /data/Videos
xdg-user-dirs-update --set TEMPLATES /data/Templates
xdg-user-dirs-update --set PUBLICSHARE /data/Public
xdg-user-dir DOWNLOAD
```

## 4. 终端与 Shell 个性化

- 系统 `x-terminal-emulator` 当前指向 **Terminator**（已安装 2.1.3），Dock 也固定了 Terminator。`update-alternatives` 处于自动模式，安装 Terminator 后因优先级较高被选中；若新电脑没有自动选中，可运行 `sudo update-alternatives --config x-terminal-emulator`。另一方面，`~/.config/xdg-terminals.list`、`ubuntu-xdg-terminals.list`、`GNOME-xdg-terminals.list` 都明确列为 `org.gnome.Terminal.desktop`，因此按 XDG 终端列表选程序的应用仍可能打开 GNOME Terminal。
- Terminator 的用户配置在 `~/.config/terminator/config`。与其默认配置相比，明确增加了 `suppress_multiple_term_dialog = True`，抑制多开终端时的提示；没有查到单独设置的配色、字体或布局。
- `~/.bashrc` 加载 `/opt/ros/jazzy/setup.bash`；将 `rm` 改成 `trash`（依赖 `trash-cli`）；将 `ls` 设为彩色显示并隐藏名为 `Documents`、`snap` 的条目。`shopt -s extglob` 也被开启。

用户追加的 Bash 配置核心如下（已有 ROS/SDK 才启用对应行）：

```bash
source /opt/ros/jazzy/setup.bash
alias rm='trash'
alias ls='ls --color=auto --hide=Documents --hide=snap'
shopt -s extglob
```

## 5. 输入法、快捷键与启动项

- 系统语言为 `zh_CN.UTF-8`，时区 `Asia/Shanghai`；系统键盘布局文件 `/etc/default/keyboard` 写为 `cn`，图形会话的 GNOME 输入源则是美式键盘布局 `us`。这两处值不同，换机时分别核对。
- 已通过 `im-config` 把输入法设为 **Fcitx 5**（`~/.xinputrc` 中 `run_im fcitx5`），Fcitx 用户配置在 `~/.config/fcitx5/`。输入法组包含 `keyboard-us` 和 **Rime**，默认输入法为 `rime`。`~/.config/fcitx5/rime/default.custom.yaml` 将 Rime 方案列表设为 **雾凇拼音** `rime_ice`；目录内还有对应词库和方案文件。换机应先安装 Fcitx 5 与 Rime，再迁移 Rime 方案及个人词库。
- 自定义快捷键（设置 → 键盘 → 查看及自定义快捷键）：`Ctrl+1` 执行 `/opt/Snipaste/Snipaste-2.11.3-x86_64.AppImage snip`；`Super+Space` 执行 `/usr/bin/fcitx5-remote -t`，切换中英文。GNOME 自身切换输入源的 `Super+Space` 已从绑定中移除，目前用 `XF86Keyboard`（反向为 `Shift+XF86Keyboard`），避免冲突。
- 用户自启动目录 `~/.config/autostart/` 有 Clash Verge、飞书、Snipaste 和 Fcitx 5 的启动项。安装对应程序后再按需恢复。
- 另有用户级 systemd 自启动：`deepseek-harness.service`（本机命令 `dsh web --no-open --host 127.0.0.1 --port 3080`）和 `opentabletdriver.service`。这两项是单独启用的服务，不在上述 `.desktop` 启动项中。

## 6. 默认应用与网络

- **默认浏览器**是 `google-chrome.desktop`，HTTP、HTTPS、`about`、`unknown` 链接也关联到 Chrome。`~/.config/mimeapps.list` 另外把本地 `text/html` 文件关联到 `chatgpt.desktop`，这是不同于网页链接的单独设置；换机时可用 `xdg-mime query default text/html` 核对。
- 用户文件关联还包含 `clash` / `clash-verge` → Clash Verge、`ccswitch` → CC Switch、`codex` → ChatGPT、`baiduyunguanjia` → 百度网盘。这些保存在 `~/.config/mimeapps.list`，安装对应应用后如需复现可参考该文件。
- GNOME 系统代理当前模式为 `none`，即**未启用**。但设置中留有非默认预设：HTTP/HTTPS/SOCKS 主机均为 `127.0.0.1`、端口 `7897`；PAC 地址 `http://127.0.0.1:44649/commands/pac`；绕过地址为 `localhost`、`127.0.0.1`、`192.168.0.0/16`、`10.0.0.0/8`、`172.16.0.0/12`、`::1`。只有重新启用代理时这些预设才生效。
- `/etc/sysctl.conf` 与发行版包提供的文件不同，末尾新增了以下 3 行；当前内核值也与文件一致。若需复现，可在新机写入 `/etc/sysctl.d/99-personal.conf`：

```ini
net.core.rmem_max = 10485760
net.core.wmem_max = 10485760
net.ipv4.conf.default.rp_filter = 0
```

## 7. 蓝牙、开发环境与系统服务

| 项目               | 本机非默认配置                                                                                 | 换机提示                                                |
| ---------------- | --------------------------------------------------------------------------------------- | --------------------------------------------------- |
| 蓝牙输入设备           | `/etc/bluetooth/input.conf` 的 `UserspaceHID=true`；该选项默认 `false`                         | 如果继续需要用户态 HID 处理，在新机设置相同选项                          |
| Docker 数据位置      | `/etc/docker/daemon.json` 为 `{"data-root":"/opt/docker/data"}`                          | 安装 Docker 后，在存放容器数据前设定目录；当前 `docker.service` 已启用并运行 |
| SSH 远程登录         | 安装了 `openssh-server`，`ssh.service` 已启用并运行                                               | 需要从其他设备 SSH 登录新机时安装并启用                              |
| Clash Verge 系统服务 | `/etc/systemd/system/clash-verge-service.service` 已启用并运行                                | 安装 Clash Verge 后核对其服务及第 5 节自启动项                     |
| AppArmor 配置文件    | 软件包校验显示 `/etc/apparmor.d/element-desktop` 内容已修改，`/etc/apparmor.d/nautilus` 这个包提供的配置文件缺失 | 这是实际文件差异，但未确认是否有意调整；如需复现，先对照新机对应软件包的配置              |

Ubuntu 软件源还有一项人工配置：`/etc/apt/sources.list` 添加了中科大镜像 `https://mirrors.ustc.edu.cn/ubuntu/` 的 `noble`、`noble-updates`、`noble-backports`、`noble-security`；系统原有的 `/etc/apt/sources.list.d/ubuntu.sources` 官方源仍在。另有 Chrome、VS Code、ChatGPT、Element、Typora、OBS Studio PPA、Android CLI、ROS 2（中科大镜像）以及 Ubuntu 实时内核的软件源文件。安装相应软件时通常会重新创建对应源，可按需要恢复。

## 核对命令

```bash
powerprofilesctl get
gsettings get org.gnome.shell.extensions.dash-to-dock dock-position
gsettings get org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type
gsettings get org.gnome.desktop.session idle-delay
xdg-user-dir DOWNLOAD
update-alternatives --display x-terminal-emulator
im-config -m
xdg-settings get default-web-browser
xdg-mime query default text/html
rg '^UserspaceHID' /etc/bluetooth/input.conf
systemctl is-enabled ssh docker clash-verge-service
```
