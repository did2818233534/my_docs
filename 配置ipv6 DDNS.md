# 配置 IPv6 DDNS：Dynu + Ubuntu 自动更新

## 目的与当前状态

让手机通过固定域名 SSH 连接家中电脑；电脑公网 IPv6 变化时自动更新域名 AAAA 记录。DDNS 不会固定运营商分配的 IPv6 前缀，也不能替代光猫和电脑防火墙放行。

2026-09-29 实测环境：Ubuntu 24.04、用户 `guga`、无线网卡 `wlp0s20f3`。

- 使用域名：`did2818233534.camdvr.org`。
- 当时非临时公网 IPv6：`240e:306:2081:6e00:6b8a:b30c:49c4:621b`，这是历史值，复现时重新读取。
- Dynu 首次 HTTPS 更新返回 `good`。
- 整理本文时已确认安装到 `/opt/dynu-ddns`，`dynu-ddns.timer` 为 `enabled`、`active`。
- 不应重新运行首次安装脚本覆盖现有配置；脚本本身也会拒绝覆盖。
- 本文不包含账户密码或 IP 更新密码。

光猫 IPv6 端口放行过程见同目录《光猫配置ipv6端口.md》。

## 1. 申请域名与手动验证

1. 注册并登录 https://www.dynu.com/ 。
2. Control Panel → DDNS Services → Add。
3. 选择服务商提供的域名，输入主机名、选择后缀。本次最终使用 `camdvr.org`。
4. 管理页启用 IPv6 地址，填入电脑当前非临时公网 IPv6，TTL 使用 120 秒，保存。
5. 页面底部 IP Update Password 设置独立更新密码，不使用网站登录密码。输入凭据应由用户在本地完成，不要把凭据写进可分享文档。

电脑查看地址：

```bash
ip -6 addr show
```

选择公网 global 地址，排除 `temporary`、`deprecated`、`tentative`、`fe80::` 链路本地地址及 VPN/虚拟接口地址。不要把光猫自身 IPv6 或代理出口地址填进去。

手机流量测试：

```bash
ssh -6 -o ConnectTimeout=10 guga@did2818233534.camdvr.org
```

本次电脑 ED25519 主机指纹为：

```text
SHA256:F86Vwmsv+Njzy3h7Sg5sCHmLzdF2W4CXNVEhmpoEzDM
```

换电脑或重装后必须重新核验，不能把此值当作通用指纹。本机核验命令：

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

## 2. DNS 与 VPN 排查经验

本次 `did2818233534.mywire.org` 在 Google DNS 返回正确 IPv6，但阿里 DNS 返回无关的 `2a03:2880:…` 地址。随机不存在的 mywire.org 子域也出现相似异常，属于高度疑似 DNS 污染；未定位具体污染环节。

换成 `did2818233534.camdvr.org` 后，阿里与 Google DNS 都返回正确 IPv6。这仅证明当时两处查询一致，不保证后缀长期或所有国内网络都正常。

对照查询：

```bash
curl -q --noproxy '*' --max-time 15 'https://dns.alidns.com/resolve?name=did2818233534.camdvr.org&type=AAAA'
curl -q --noproxy '*' --max-time 15 'https://dns.google/resolve?name=did2818233534.camdvr.org&type=AAAA'
```

Google DNS 在某些网络可能不可达；失败不能直接等同于记录错误。`--noproxy` 仅绕过 HTTP 代理环境变量，不能绕过系统 TUN/VPN。

本次电脑曾解析到 `2001:2::f2` 且 SSH 指纹不匹配，疑似代理 Fake-IP/路由干扰。出现此类问题先确认目标，不要关闭 SSH 主机校验。手机 Clash Meta 可临时停用作为诊断，最终也可将该域名 SSH 流量直连、DNS 查询走可信解析器；必要时开启 IPv6 DNS 并排除 Fake-IP。应用分流绕过 Termux 会让该应用所有连接直连。

数字地址能登录、域名失败时，优先查 DNS。`No address associated with hostname` 表示没有得到可用 IPv6；`Permission denied` 属认证问题，但要先确认主机指纹正确。

截图里的 IPv4 曾是代理出口而非家庭地址。本方案用 `myip=no` 不改 IPv4，也不会删除旧 A 记录。连接必须保留 `ssh -6`；若希望 IPv6-only 域名，可在 Dynu 控制台另行清理错误 A 记录。

## 3. 自动更新设计

- Python 3 标准库，无第三方 Python 依赖。
- 从指定网卡的 `ip -j -6 addr` 获取地址，排除临时/失效/非公网地址。
- 只更新指定域名，HTTPS Basic Auth 携带专用更新密码，密码不放 URL、命令行参数或日志。
- `myipv6` 显式传入电脑地址，`myip=no` 禁止自动检测代理出口 IPv4。
- systemd 系统级定时器开机约 60 秒后启动，以用户 guga 运行；每 5 分钟检查，附带最多 15 秒随机延迟。
- 地址不变时跳过请求，每 24 小时重新同步；`--force` 可强制更新。
- 没有可用地址时保留旧记录；失败返回非零并等待后续定时重试。
- 电脑关机/休眠或断网时不能更新，DNS 可能暂时保留旧地址。
- 换网卡名后要修改 config.json。当前实现只关注配置的一个网卡。

官方接口：https://www.dynu.com/DynamicDNS/IP-Update-Protocol

## 4. 当前安装与日常检查

文件：

```text
/opt/dynu-ddns/update.py
/opt/dynu-ddns/config.json       # 敏感，600，guga 所有
/opt/dynu-ddns/state.json        # 上次成功同步状态
/etc/systemd/system/dynu-ddns.service
/etc/systemd/system/dynu-ddns.timer
```

检查运行状态：

```bash
systemctl is-enabled dynu-ddns.timer
systemctl is-active dynu-ddns.timer
systemctl list-timers dynu-ddns.timer
systemctl show dynu-ddns.service -p Result -p ExecMainStatus
journalctl -u dynu-ddns.service -n 20 --no-pager
```

oneshot 服务完成后显示 inactive 是正常的，timer 应保持 active。手动更新：

```bash
sudo systemctl start dynu-ddns.service
# 如需立即强制调用接口，避免与正在运行的服务同时执行：
python3 /opt/dynu-ddns/update.py --force
```

日志 `good`/`nochg` 表示更新成功/地址未变。`badauth` 检查更新密码、用户名和主机名；不要输出凭据排查。首次安装成功后，暂存目录的 config.json 已由安装脚本删除。

## 5. 在另一台电脑复现

先确认用户、网卡、域名与 Python/ip 路径。以下代码是本次实际使用版本，不要盲用 guga 和 wlp0s20f3。

准备一个仅自己可读的临时目录，把下面 update.py、service、timer、install.sh 保存进去。创建权限 600 的 config.json，用本地编辑器填写，格式如下：

```json
{
  "hostname": "did2818233534.camdvr.org",
  "username": "did2818233534",
  "interface": "wlp0s20f3",
  "password": "在本地填写专用IP更新密码"
}
```

目录权限设为 700，配置文件权限设为 600。不要把配置文件提交 Git。先运行 `python3 update.py --force` 验证，再执行 `sudo bash install.sh`；sudo 输入的是本机管理员密码，不是 Dynu 密码。

本机原暂存目录是 `/data/Documents/ChatGPT/sr3/dynu-ddns-setup`。新电脑可以使用自己的暂存目录，但最终安装目录保持 `/opt/dynu-ddns`。

### update.py

```python
#!/usr/bin/env python3
import base64
import ipaddress
import json
import os
from pathlib import Path
import subprocess
import sys
import time
import urllib.parse
import urllib.request

BASE = Path(__file__).resolve().parent

def main():
    config = json.loads((BASE / 'config.json').read_text())
    interfaces = json.loads(subprocess.check_output(
        ['/usr/sbin/ip', '-j', '-6', 'addr', 'show', 'dev', config['interface']], text=True))
    candidates = []
    for interface in interfaces:
        for entry in interface.get('addr_info', []):
            if entry.get('scope') != 'global' or any(entry.get(k) for k in
                    ('temporary', 'tentative', 'deprecated', 'dadfailed')):
                continue
            address = ipaddress.IPv6Address(entry['local'])
            if address.is_global and entry.get('preferred_life_time', 0) != 0:
                candidates.append(str(address))
    if not candidates:
        print('No preferred non-temporary global IPv6; retaining DNS record.')
        return
    statefile = BASE / 'state.json'
    try:
        state = json.loads(statefile.read_text())
    except (FileNotFoundError, ValueError):
        state = {}
    address = state.get('address') if state.get('address') in candidates else sorted(candidates)[0]
    if (state.get('address') == address and state.get('hostname') == config['hostname']
            and 0 <= time.time() - state.get('updated', 0) < 86400 and '--force' not in sys.argv):
        print('IPv6 unchanged; no update needed.')
        return
    query = urllib.parse.urlencode({'hostname': config['hostname'], 'myip': 'no', 'myipv6': address})
    request = urllib.request.Request('https://api.dynu.com/nic/update?' + query,
        headers={'Authorization': 'Basic ' + base64.b64encode(
            (config['username'] + ':' + config['password']).encode()).decode(),
            'User-Agent': 'guga-dynu-ipv6/1.0'})
    # Explicit address selection prevents VPN egress addresses from being published.
    opener = urllib.request.build_opener(urllib.request.ProxyHandler({}))
    try:
        with opener.open(request, timeout=25) as response:
            result = response.read(2048).decode().strip().split()
    except Exception as error:
        print('Dynu HTTPS request failed (' + type(error).__name__ + '); retry on next timer.', file=sys.stderr)
        sys.exit(1)
    status = result[0] if result else 'empty'
    if status not in ('good', 'nochg'):
        safe = status if status in ('badauth', 'abuse', 'nohost', 'notfqdn', 'servererror', '911', 'unknown', 'numhost') else 'unexpected'
        print('Dynu update failed: ' + safe, file=sys.stderr)
        sys.exit(1)
    os.umask(0o077)
    temporary = BASE / 'state.json.tmp'
    temporary.write_text(json.dumps({'hostname': config['hostname'], 'address': address, 'updated': time.time()}))
    temporary.replace(statefile)
    print(f'Dynu {status}: {config["hostname"]} -> {address}')

if __name__ == '__main__':
    main()

```

### dynu-ddns.service

```ini
[Unit]
Description=Update computer IPv6 AAAA record on Dynu
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
User=guga
Group=guga
ExecStart=/usr/bin/python3 /opt/dynu-ddns/update.py
TimeoutStartSec=45
UMask=0077
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/dynu-ddns


```

### dynu-ddns.timer

```ini
[Unit]
Description=Check Dynu IPv6 every five minutes

[Timer]
OnBootSec=60
OnUnitActiveSec=5min
RandomizedDelaySec=15
Unit=dynu-ddns.service

[Install]
WantedBy=timers.target

```

### install.sh

```bash
#!/bin/bash
set -euo pipefail
if [ "$EUID" -ne 0 ]; then
    echo 'Run this installer with sudo.' >&2
    exit 1
fi
setup_dir=$(cd -- "$(dirname -- "$0")" && pwd)
if [ -e /opt/dynu-ddns/config.json ]; then
    echo '/opt/dynu-ddns/config.json already exists; refusing to overwrite.' >&2
    exit 1
fi
install -d -m 700 -o guga -g guga /opt/dynu-ddns
install -m 600 -o guga -g guga "$setup_dir/config.json" /opt/dynu-ddns/config.json
install -m 755 -o root -g root "$setup_dir/update.py" /opt/dynu-ddns/update.py
install -m 644 "$setup_dir/dynu-ddns.service" /etc/systemd/system/dynu-ddns.service
install -m 644 "$setup_dir/dynu-ddns.timer" /etc/systemd/system/dynu-ddns.timer
systemctl daemon-reload
systemctl start dynu-ddns.service
systemctl enable --now dynu-ddns.timer
rm -- "$setup_dir/config.json"
systemctl --no-pager status dynu-ddns.timer

```

## 6. 停用与交接

停止自动更新：

```bash
sudo systemctl disable --now dynu-ddns.timer
sudo systemctl stop dynu-ddns.service
```

停用不会删除 Dynu 已存在的记录，也不会关闭 SSH。若迁移域名，修改本地 config.json 后执行强制更新并验证新域名；确认用户需求后再删除旧域名。

接手 AI 应先读取安装状态而非重复安装。不得在回复中输出 config.json 内容或真实密码。测试产生的临时进程应及时结束，保留用户明确需要的 systemd 定时器。复现完成需核实 API 返回、DNS AAAA、手机外网 SSH 和定时器状态；未完成的验证必须明确标注。
