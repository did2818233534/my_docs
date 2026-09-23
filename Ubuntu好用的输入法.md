# Ubuntu 24.04 安装雾凇拼音提示词

把下面这段提示词交给另一台 Ubuntu 24.04 电脑上的助手：

```text
请直接在这台 Ubuntu 24.04 电脑上安装并配置 Fcitx 5 + Rime + 雾凇拼音，并完成验证。

目标：Fcitx 5 默认使用 Rime；Rime 默认使用雾凇拼音全拼；雾凇拼音默认处于中文模式，输出简体字，同时使用英文半角标点（例如 , 和 .）。

要求：
1. 先检查当前桌面环境、输入法及已有 Rime 配置，并备份现有配置和用户词库。不要直接清空或覆盖用户词库。
2. 优先通过 Ubuntu 官方软件仓库安装 Fcitx 5、fcitx5-rime、配置工具及必要的输入法前端；确认 librime 版本满足雾凇拼音要求，并且 librime-lua 可用。如果需要手动下载或安装软件，把软件放到 /opt/软件名，不要在家目录留下软件安装目录。
3. 从 iDvel/rime-ice 官方仓库获取完整的雾凇拼音配置。Fcitx 5 的 Rime 用户配置必须放在 ~/.local/share/fcitx5/rime/；不要误放到 ~/.config/fcitx5/rime/。用户配置目录不受上述 /opt 软件安装规则限制。
4. 用 im-config 启用 Fcitx 5，在 Fcitx 5 中添加 Rime 并设为默认。将 Rime 的默认方案设为 rime_ice（雾凇拼音全拼）。
5. 合并而不是盲目覆盖已有自定义配置。确认当前雾凇版本的 switches 顺序后，通过 custom.yaml 设置 ascii_mode=0（中文）、traditionalization=0（简体）、ascii_punct=1（英文标点）；调整相关状态记忆设置，使这些默认值在切换窗口或重新登录后仍然生效。
6. 重新部署 Rime。不要仅凭文件已复制或 Fcitx 5 已重启就判断成功；确认 rime_ice 已编译且正在被当前输入法进程加载。
7. 实际测试：能输入简体中文；在中文模式下输入逗号和句号时，得到英文半角标点 , 和 .。最后报告安装的软件版本、实际配置目录、默认输入法、测试结果，以及是否需要重新登录。
```

参考：

- 雾凇拼音官方仓库：https://github.com/iDvel/rime-ice
- Rime 用户目录说明：https://github.com/rime/home/wiki/UserData
- Fcitx 5 设置说明：https://fcitx-im.org/wiki/Setup_Fcitx_5
