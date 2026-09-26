# Ubuntu 24.04 顶栏中国节假日日历复现指南

记录日期：2026-09-23；更新日期：2026-09-26。适用环境：Ubuntu 24.04、GNOME Shell 46。本文只记录本次对话中与日历有关的配置，不包含 Webcamoid。

## 最终效果与组成

点击顶栏时间后，月历中的法定放假日显示「休」和淡蓝色方格，调休补班日显示「班」和淡红色方格。方格为 6px 小圆角，相邻方格有适度间隙；「今天」的蓝色背景和选中日期的蓝色边框也改为 6px 圆角方形。农历和节日名称仍显示，原有字号、字重不变。具体颜色和圆角是扩展样式的手工修改，并非扩展设置界面的内置主题。

这个效果由两套**独立**数据组成：

1. **GNOME 日历订阅**：让顶栏日历读到放假、补班事件，日期下出现事件圆点，选中日期可查看事件。订阅源为 [ShuYZ 中国节假日 ICS](https://github.com/lanceliao/china-holiday-calender)。
2. **Chinese Calendar 扩展**：在日期格显示农历、节日和「休／班」，并用本指南的 CSS 加上淡色背景。扩展的休班标记读取自己的 [JSON 数据](https://tigertall.github.io/chinese-calendar/data/holidays_cn.json)，**不读取上面的 ICS**。[扩展项目](https://github.com/tigertall/chinese-calendar)。

本机使用 GNOME 日历 46.1、[Chinese Calendar 1.7](https://extensions.gnome.org/extension/9586/chinese-calendar/)（扩展 UUID：`chinese-calendar@tigertall`）。

## 在另一台 Ubuntu 24.04 上复现

### 1. 安装 GNOME 日历

```bash
sudo apt update
sudo apt install -y gnome-calendar curl unzip
```

这些 apt 软件包装在系统目录；扩展文件按本机习惯放到 `/opt/chinese-calendar`。

### 2. 订阅法定节假日事件

创建 Evolution Data Server 的网页日历源。以下是本机实际使用的地址；复制整段命令即可：

```bash
mkdir -p "$HOME/.config/evolution/sources"
cat > "$HOME/.config/evolution/sources/china-holidays-shuyz.source" <<'SOURCE'
[Data Source]
DisplayName=中国法定节假日与补班
Enabled=true
Parent=webcal-stub

[Calendar]
BackendName=webcal
Color=#ff3b30
Selected=true

[Authentication]
Host=www.shuyz.com
Method=none
Port=443
ProxyUid=system-proxy

[Security]
Method=tls

[WebDAV Backend]
ResourcePath=/githubfiles/china-holiday-calender/master/holidayCal.ics
ResourceQuery=

[Refresh]
Enabled=true
IntervalMinutes=30

[Offline]
StaySynchronized=true
SOURCE
chmod 600 "$HOME/.config/evolution/sources/china-holidays-shuyz.source"
```

完整订阅地址：`https://www.shuyz.com/githubfiles/china-holiday-calender/master/holidayCal.ics`。这是**订阅**，来源更新后会由系统重新获取；单独下载并导入 `.ics` 文件不能保持更新。

也可打开「日历」应用，进入「☰ → 管理日历 → 添加日历」，粘贴上述完整订阅地址。两种方法选一种即可，避免重复事件。

### 3. 安装 Chinese Calendar 扩展到 /opt

下面固定使用本次安装的 1.7 版，适配 GNOME Shell 46。若以后扩展网站更换版本，可到[扩展页面](https://extensions.gnome.org/extension/9586/chinese-calendar/)选择兼容 GNOME 46 的版本。

```bash
curl -LfsS 'https://extensions.gnome.org/download-extension/chinese-calendar@tigertall.shell-extension.zip?version_tag=75318' -o /tmp/chinese-calendar-1.7.zip
sudo install -d -m 755 /opt/chinese-calendar
sudo unzip -oq /tmp/chinese-calendar-1.7.zip -d /opt/chinese-calendar
sudo chmod -R a+rX /opt/chinese-calendar
sudo glib-compile-schemas --strict /opt/chinese-calendar/schemas
mkdir -p "$HOME/.local/share/gnome-shell/extensions"
ln -s /opt/chinese-calendar "$HOME/.local/share/gnome-shell/extensions/chinese-calendar@tigertall"
rm /tmp/chinese-calendar-1.7.zip
```

下载的 1.7 版压缩包在本次安装时的 SHA-256：`9a1e57e210df05d4d328a858e88d80c4fb9e8e450e40eecd9c985497941d4358`。如新电脑上已有同名扩展目录，先备份它，再创建指向 `/opt` 的链接。

`chmod -R a+rX` 是必需的：曾因 `/opt/chinese-calendar/metadata.json` 权限为 `600`，重启后 GNOME Shell 无法读取扩展，定制效果随之消失。

### 4. 加入本次定制的背景、圆角和间隙

**只追加下面的 CSS**，无需改扩展的 JavaScript。它保留原有「休／班」文字大小和字重，并将「今天」及选中日期改为圆角方形。

```bash
cat <<'CSS' | sudo tee -a /opt/chinese-calendar/stylesheet.css >/dev/null

/* 本机定制：休/班方格的背景、圆角与间隙，以及今天/选中日期的圆角 */
.lunar-badge {
    border-radius: 6px;
    margin: 2px;
}
.lunar-badge-rest {
    background-color: rgba(72, 145, 235, 0.22);
}
.lunar-badge-work {
    background-color: rgba(232, 92, 105, 0.24);
}
.lunar-calendar .calendar-day {
    border-radius: 6px !important;
}
CSS
```

蓝色对应「休」，红色对应「班」；`0.22` 和 `0.24` 是背景不透明度。`margin: 2px` 让相邻方格之间保留不大的间隙。
最后一条规则让「今天」和选中日期使用同样的 6px 圆角，不改变它们原有的蓝色填充或边框。

### 5. 启用扩展并更新休班数据

手动放入的 GNOME Shell 扩展通常要**注销桌面会话并重新登录一次**才能被发现。重新登录后执行：

```bash
gnome-extensions enable chinese-calendar@tigertall
GSETTINGS_SCHEMA_DIR=/opt/chinese-calendar/schemas gsettings set org.gnome.shell.extensions.chinese-calendar festival-region CN
GSETTINGS_SCHEMA_DIR=/opt/chinese-calendar/schemas gsettings set org.gnome.shell.extensions.chinese-calendar show-statutory-holidays true
GSETTINGS_SCHEMA_DIR=/opt/chinese-calendar/schemas gsettings set org.gnome.shell.extensions.chinese-calendar show-lunar-in-panel false
gnome-extensions prefs chinese-calendar@tigertall
```

在打开的扩展设置中，找到「法定假日 → 更新数据」，**第一次使用要点一次**。这样会下载中国大陆休班数据。`show-lunar-in-panel false` 只是不在顶栏时钟旁再加一段农历文字，不影响弹出的日历格。

### 6. 检查效果

点击顶栏时间，翻到 **2026 年 9 月**：

- **9 月 20 日**：国庆调休补班，应显示「班」和淡红色背景。
- **9 月 25–27 日**：中秋假期，应显示「休」和淡蓝色背景。
- **10 月 1–7 日**：国庆假期，也应显示「休」和淡蓝色背景。

再选中一个非今天的日期，检查蓝色选中边框为圆角方形；回到今天，检查蓝色背景也为圆角方形。

若已改 CSS 而画面尚未刷新，可执行：

```bash
gnome-extensions disable chinese-calendar@tigertall
gnome-extensions enable chinese-calendar@tigertall
```

如果重启后扩展消失，先检查 `/opt/chinese-calendar/metadata.json` 是否可供普通用户读取。不可读取时执行 `sudo chmod -R a+rX /opt/chinese-calendar`，然后注销并重新登录桌面会话，再运行 `gnome-extensions info chinese-calendar@tigertall` 检查是否启用。

GNOME 45 及以后会缓存扩展的 JavaScript；若修改的是 `extension.js` 而不是本指南的 CSS，通常需要注销并重新登录，单纯关闭再打开扩展不足以加载新代码。本次最终方案**没有修改 JavaScript**。

## 以后年度如何更新

国务院公布下一年度安排、两个第三方数据源完成更新后：

- ICS 订阅会由系统重新获取放假、补班事件。
- 在 `gnome-extensions prefs chinese-calendar@tigertall` 中再次点「更新数据」，让扩展刷新「休／班」标记。不要假定它会读取 ICS 订阅。
- 安排若有疑问，以[国务院办公厅的正式放假通知](https://www.gov.cn/zhengce/content/202511/content_7047090.htm)为准；此链接对应 2026 年安排，往后应查当年的通知。

扩展更新可能覆盖手工 CSS。更新后若淡色背景或今天、选中日期的圆角消失，重新执行第 4 步即可。本文没有记录或保存任何管理员密码。
