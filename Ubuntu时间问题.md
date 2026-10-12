# Ubuntu 时间问题：双系统时钟统一与自动校时修复

记录日期：2026-10-12。适用环境：Windows 11 + Ubuntu 双系统；本次 Ubuntu 为 24.04.4 LTS，使用 `systemd-timesyncd` 校时，时区为 `Asia/Shanghai`。

本文用于换机后复现：让 Windows 和 Ubuntu 使用一致的硬件时钟规则，并解决自动校时服务器持续超时的问题。新机器要先检查实际环境，不能直接假定原因、用户名、代理配置和校时服务都与本机相同。

## 给 ChatGPT 的执行提示词

把本文交给能操作目标电脑的 ChatGPT，并附上下面这段提示词：

```text
请依据《Ubuntu时间问题.md》，直接在这台 Windows + Ubuntu 双系统电脑的 Ubuntu 中完成检查、备份、修复和验证。

目标：
1. Windows 和 Ubuntu 对硬件时钟采用一致的解释，切换系统不再出现 8 小时偏差。
2. Ubuntu 自动网络校时正常，服务开机启用，配置重启后仍有效。

执行要求：
1. 先检查时区、RTC 模式、实际硬件时钟、当前校时服务、有效配置、服务日志和网络连通性。用独立 NTP 服务器核对时间，不能把本机 date 或同一主机上的“当前时间”工具当成独立基准。
2. 本文默认 Windows 沿用本地硬件时钟。如果 Windows 已设置 RealTimeIsUniversal=1，就不能直接应用本地 RTC 方案；先明确两边要统一为哪种模式。
3. 优先复用已有校时服务。本文的 systemd-timesyncd 操作仅适用于它确实是当前校时服务的机器；不要同时启用 chrony、ntpd 和 systemd-timesyncd。
4. 写入前备份现有配置，记录原始 RTC、时区及服务启用状态。使用独立的配置片段，避免清空整个系统配置目录。
5. 从本文候选服务器中实测可用的端点，再配置自动校时。域名被 Clash/mihomo 分配 fake IP 不代表一定失败；以实际 NTP 响应和系统服务日志为准。
6. 先让系统时间通过网络校准成功，再设置本地 RTC 并将正确时间写入硬件时钟。不要先把错误的系统时间写进去，也不要重复机械地加减 8 小时。
7. 首次改变 RTC 模式后安排一次重启回 Ubuntu，完成实际 RTC 和持续运行验证，再验证 Windows → Ubuntu 的切换。重启会中断正在进行的工作，必须先由我确认；未重启的检查结果不能写成已经完成双系统验收。
8. 需要管理员密码时，让我在系统认证窗口或本机终端输入，不要要求我在聊天中发送密码，不要绕过权限认证。
9. 如果需要安装系统软件，优先使用 Ubuntu 官方软件仓库；默认安装在主目录的软件改装到 /opt/软件名。本文通常不需要安装 ntpdate 或 hwclock。
10. 结束后清理临时脚本、认证进程和测试后台，保留正式配置与备份。报告修改的文件、备份位置、验证结果及尚未完成的步骤。
```

## 问题背景

本次同时发现两个问题：

| 问题 | 本次证据 | 处理方向 |
| --- | --- | --- |
| 系统时间快约 8 小时 | 电脑显示约 19:09，独立 NTP 服务器返回的北京时间约 11:09；时区已经是 `Asia/Shanghai` | 检查 Windows 与 Ubuntu 的硬件时钟规则是否一致 |
| 自动校时一直未成功 | `NTP service: active`，但 `System clock synchronized: no`；日志不断显示 `ntp.ubuntu.com` 超时 | 改用当前网络能够连通的 NTP 端点 |

Windows 默认把硬件时钟当作本地时间，Ubuntu 通常按 UTC 解释。在北京时间环境中，这种解释差异可以造成 8 小时偏差。本文采用的方案是把 Ubuntu 的硬件时钟模式改为本地时间，与 Windows 默认行为一致。

本次 Clash Verge/mihomo 接管了 DNS，`ntp.ubuntu.com` 被解析为 `198.18.0.37` 和 `2001:2::25`，请求超时。Cloudflare 的两个服务器 IP 和阿里云校时服务都能响应。这说明当时默认服务器的访问路径有问题，但仅凭这些信息不能断言超时完全由 fake IP 导致；本次 `ntp.aliyun.com` 在 fake IP 下也能正常响应。

## 适用场景与前提

- 在物理电脑上的 Ubuntu 中操作，电脑与 Windows 共用同一个硬件时钟。容器、WSL 和虚拟机不直接套用本文。
- Windows 使用默认的本地硬件时钟约定，两个系统的时区与实际所在地一致。
- Ubuntu 使用 `systemd-timesyncd` 自动校时。其他校时服务应按其自身配置方式处理。
- 本文示例使用北京时间 `Asia/Shanghai`；其他地区应调整两个系统的时区。

如果以前修改过 Windows 的 UTC 注册表设置，可在 Windows 终端检查：

```powershell
Get-TimeZone
reg query "HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal
```

`RealTimeIsUniversal` 直译为“实时时钟使用通用时间”。值不存在或为 `0` 时，通常沿用本地硬件时钟；值为 `1` 时，Windows 按 UTC 解释硬件时钟。后一种情况应让 Ubuntu 也使用 UTC，或先明确要调整 Windows，不能让两边模式相反。

## 整体思路

1. 检查当前时间、时区、实际 RTC、校时服务和超时日志。
2. 用独立 NTP 服务器验证偏差和端点连通性。
3. 备份配置，配置可用的自动校时服务器。
4. 等待网络校时成功，确认系统时间正确。
5. 将 RTC 改为本地时间，并把正确系统时间写入 RTC。
6. 首次修改后重启回 Ubuntu，验证 RTC 没有再次被写成 UTC。
7. 实测 Windows → Ubuntu 切换，并清理测试过程。

## 1. 检查 Ubuntu 当前状态

```bash
date --iso-8601=seconds
timedatectl status
timedatectl show -p Timezone -p LocalRTC -p NTP -p NTPSynchronized
timedatectl timesync-status
systemctl status systemd-timesyncd --no-pager
systemctl is-enabled systemd-timesyncd
systemd-analyze cat-config systemd/timesyncd.conf
journalctl -u systemd-timesyncd -b --no-pager -n 40
```

几个名称的含义：

| 名称 | 含义 |
| --- | --- |
| RTC / Real-Time Clock | 实时时钟，即断电后仍由电池维持的硬件时钟 |
| UTC / Coordinated Universal Time | 协调世界时；北京时间为 UTC+8 |
| NTP / Network Time Protocol | 网络时间协议，用于通过网络校准时间 |
| `LocalRTC=yes` | 本地（Local）硬件时钟（RTC）模式已启用 |
| `NTP=yes` | 自动网络校时已启用，不能单凭这一项判断同步成功 |
| `NTPSynchronized=yes` | NTP 时间同步（Synchronized）已经成功 |
| `Offset` | 最近一次校时测量的偏差，首次同步时可能仍显示修正前的 8 小时偏差 |
| `Packet count` | 已收到的校时数据包数量，可用于确认后续自动校时继续进行 |

确认是否存在其他校时服务：

```bash
systemctl list-unit-files 'systemd-timesyncd.service' 'chrony.service' 'chronyd.service' 'ntp.service' 'ntpd.service'
```

若机器已经使用 chrony 或 ntpd，应先检查现有服务，避免另外启用一个服务抢着调整系统时间。

检查实际硬件时钟，而不只看配置标志：

```bash
date '+%F %T %Z'
date -u '+%F %T UTC'
cat /sys/class/rtc/rtc0/date
cat /sys/class/rtc/rtc0/time
cat /etc/adjtime
```

`/etc/adjtime` 不存在时，Ubuntu 通常按 UTC 处理。采用本文方案后，它的第三行应为 `LOCAL`，且实际 RTC 应接近本地时间。例如本地时间 11:30、UTC 时间 03:30 时，RTC 应为约 11:30。

## 2. 排查服务器超时与代理 DNS

```bash
getent ahosts ntp.ubuntu.com
resolvectl status
timedatectl show-timesync --all
```

如果 DNS 返回 `198.18.*` 等代理虚拟地址，应结合代理状态和 NTP 实测判断。不要把“能打开网页”当作“UDP 123 校时一定能通”。

以下脚本只发出 NTP 查询，不修改系统时间。只需要 Python 3 标准库，每个端点最多等待 4 秒：

```bash
python3 - <<'PY'
import concurrent.futures
import datetime
import socket
import struct
import time

def check(server):
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.settimeout(4)
    try:
        start = time.time()
        stamp = start + 2208988800
        request = bytearray(48)
        request[0] = 0x23
        struct.pack_into('!II', request, 40, int(stamp),
                         int((stamp - int(stamp)) * 2**32))
        sock.sendto(request, (server, 123))
        packet, source = sock.recvfrom(1024)
        end = time.time()
        if (len(packet) < 48 or packet[0] & 7 != 4
                or packet[0] >> 6 == 3 or not 1 <= packet[1] <= 15
                or packet[24:32] != request[40:48]):
            raise ValueError('无效或未同步的 NTP 响应')
        def timestamp(index):
            seconds, fraction = struct.unpack('!II', packet[index:index + 8])
            return seconds - 2208988800 + fraction / 2**32
        offset = ((timestamp(32) - start) + (timestamp(40) - end)) / 2
        remote = datetime.datetime.fromtimestamp(
            timestamp(40), datetime.timezone.utc)
        return {'server': server, 'source': source[0],
                'server_utc': remote.isoformat(),
                'server_minus_local_seconds': round(offset, 6),
                'round_trip_ms': round((end - start) * 1000, 2)}
    except Exception as error:
        return {'server': server, 'error': str(error)}
    finally:
        sock.close()

servers = ['ntp.ubuntu.com', '162.159.200.1',
           '162.159.200.123', 'ntp.aliyun.com']
with concurrent.futures.ThreadPoolExecutor(max_workers=4) as pool:
    for result in pool.map(check, servers):
        print(result)
PY
```

`server_minus_local_seconds` 意思是“服务器时间减去本机时间”。接近 `-28800` 表示本机快约 8 小时，接近 `0` 表示两者一致。换机后应重新实测端点，本文记录的连通性不代表未来所有网络环境都会相同。

## 3. 备份并配置自动校时

确认本机使用 `systemd-timesyncd`，且下面的候选端点已实测可用后，再执行这一段。若只有部分端点可用，应调整 `NTP=` 候选列表。

本次直接使用 [Cloudflare 官方列出的两个 NTP IP](https://developers.cloudflare.com/time-services/ntp/usage/)，可以避开代理 DNS 虚拟地址映射；阿里云作为同一列表中的后续候选。IP 方式避开的是域名解析，不代表绕过整个代理或路由。

```bash
sudo bash <<'BASH'
set -euo pipefail

repair_backup=$(mktemp -d /var/backups/dual-boot-time.XXXXXXXX)
chmod 700 "$repair_backup"
repair_config=/etc/systemd/timesyncd.conf.d/90-dual-boot-time.conf

if [[ -e /etc/adjtime ]]; then
    cp -a /etc/adjtime "$repair_backup/adjtime"
else
    touch "$repair_backup/adjtime.was-absent"
fi
if [[ -e "$repair_config" ]]; then
    cp -a "$repair_config" "$repair_backup/90-dual-boot-time.conf"
else
    touch "$repair_backup/drop-in.was-absent"
fi
timedatectl status > "$repair_backup/status-before.txt"
timedatectl show -p Timezone -p LocalRTC -p NTP -p NTPSynchronized \
    > "$repair_backup/properties-before.txt"
systemctl is-enabled systemd-timesyncd \
    > "$repair_backup/service-enabled-before.txt" || true
systemd-analyze cat-config systemd/timesyncd.conf \
    > "$repair_backup/timesyncd-before.txt"
printf '备份目录：%s\n' "$repair_backup"

install -d -m 755 /etc/systemd/timesyncd.conf.d
install -m 644 /dev/stdin "$repair_config" <<'CONFIG'
# Use tested NTP endpoints. IP endpoints avoid proxy DNS mappings.
[Time]
NTP=
NTP=162.159.200.1 162.159.200.123 ntp.aliyun.com
FallbackNTP=
CONFIG

timedatectl set-ntp true
systemctl restart systemd-timesyncd

python3 - <<'WAIT_SYNC'
import subprocess
import sys
import time

deadline = time.monotonic() + 90
while time.monotonic() < deadline:
    state = subprocess.check_output(
        ['timedatectl', 'show', '-p', 'NTPSynchronized', '--value'],
        text=True).strip()
    if state == 'yes':
        print('网络时间同步成功。')
        break
    time.sleep(1)
else:
    print('90 秒内未同步成功，请检查有效配置和日志；暂不修改 RTC。',
          file=sys.stderr)
    sys.exit(2)
WAIT_SYNC

timedatectl status
timedatectl timesync-status
printf '请保留备份目录：%s\n' "$repair_backup"
BASH
```

这个等待脚本使用单调时间计时，因此系统时间回拨 8 小时不会影响等待上限。ChatGPT 调用工具时应让命令会话定期返回状态，避免长时间阻塞沟通。

配置中空的 `NTP=` 用于清除此前累积的系统服务器列表，再写入本次候选。空的 `FallbackNTP=` 清除编译默认的兜底服务器，避免继续保留有问题的 `ntp.ubuntu.com`。

**`FallbackNTP` 直译为“兜底 NTP”，仅在没有其他服务器信息时使用，不是所有 `NTP=` 端点超时后自动使用的备用列表。** 因此本文将阿里云也放进 `NTP=`，让服务能够实际尝试它。接口提供的 NTP 服务器也可能参与选择，必须通过 `timedatectl show-timesync --all` 核对最终结果。配置语义见 [systemd 官方说明](https://github.com/systemd/systemd/blob/v255/man/timesyncd.conf.xml)。

## 4. 设置本地硬件时钟

必须先确认：

- `NTPSynchronized=yes`，且系统显示时间与手机或独立 NTP 基准一致。
- Windows 按本地时间使用 RTC。
- 当前时区正确；北京时间应为 `Asia/Shanghai`。

如果时区确实设置错了，先调整为实际所在地。例如北京时间：

```bash
sudo timedatectl set-timezone Asia/Shanghai
```

然后设置本地 RTC：

```bash
sudo timedatectl set-local-rtc 1
timedatectl show -p LocalRTC -p NTPSynchronized
cat /etc/adjtime
```

`set-local-rtc` 直译为“设置本地硬件时钟”，`1` 表示启用。修改模式时，这个命令会把当前系统时间写入 RTC，并将 `/etc/adjtime` 第三行设为 `LOCAL`。写入方向见 [timedatectl 官方说明](https://github.com/systemd/systemd/blob/v255/man/timedatectl.xml)。

**若本来已经是 `LocalRTC=yes`，但实际 RTC 数值不对，同一命令可能直接返回而不重写 RTC。** 在确认系统时间已经准确后，可用以下两步强制重新写入；两步都不添加 `--adjust-system-clock`：

```bash
sudo timedatectl set-local-rtc 0
sudo timedatectl set-local-rtc 1
```

`--adjust-system-clock` 意思是“调整系统时钟”。添加它时，方向变成用 RTC 调整系统时间。本流程已经通过 NTP 校准了系统时间，应以正确系统时间为基准写 RTC，因此不使用这个参数。

## 5. 首次重启回 Ubuntu，并验证实际 RTC

**首次从 UTC RTC 改为本地 RTC 后，把一次重启回 Ubuntu 纳入完整修复流程。仅看到 `LocalRTC=yes` 不足以证明双系统切换已经稳定。**

本次复查发现：设置后 RTC 曾与本地时间一致，继续运行后却再次变成 UTC 数值，同时 `LocalRTC=yes` 和 `NTPSynchronized=yes` 仍然成立。这与当前启动周期中的内核 RTC 更新约定尚未重新初始化相符。systemd 在启动时读取 RTC 模式，Linux 内核的周期性 RTC 更新也依赖启动时建立的本地时钟标志；对应实现见 [systemd 启动时钟处理](https://github.com/systemd/systemd/blob/v255/src/core/main.c)及 [Linux RTC 更新代码](https://github.com/torvalds/linux/blob/master/kernel/time/ntp.c)。

操作顺序：

1. 保存正在编辑的文件，结束需要保留状态的工作。
2. 由用户确认后重启，启动菜单中先选择 **Ubuntu**。
3. 联网，确认自动校时成功。
4. 检查实际 RTC 与本地时间是否一致。
5. 正常使用至少 12 分钟后再次检查，确认 RTC 没有重新变成 UTC；不需要留下后台计时程序。

用户准备好后，可从桌面菜单重启，或执行：

```bash
sudo systemctl reboot
```

重启后检查：

```bash
timedatectl status
timedatectl timesync-status
systemctl is-enabled systemd-timesyncd
date '+%F %T %Z'
cat /sys/class/rtc/rtc0/date
cat /sys/class/rtc/rtc0/time
```

下面的只读脚本用于明确比较 RTC 与本地时间。如果机器的 RTC 不是 `rtc0`，先检查 `/sys/class/rtc/` 并调整路径：

```bash
python3 - <<'PY'
from datetime import datetime
from pathlib import Path

rtc_dir = Path('/sys/class/rtc/rtc0')
for attempt in range(3):
    first_date = (rtc_dir / 'date').read_text().strip()
    rtc_time = (rtc_dir / 'time').read_text().strip()
    second_date = (rtc_dir / 'date').read_text().strip()
    if first_date == second_date:
        break
else:
    raise SystemExit('读取时跨越日期边界，请重新检查。')
rtc = datetime.fromisoformat(first_date + 'T' + rtc_time)
local = datetime.now()
difference = (local - rtc).total_seconds()
print('本地时间：', local.isoformat(timespec='seconds'))
print('实际 RTC：', rtc.isoformat())
print('本地时间减 RTC，秒：', round(difference, 3))
if abs(difference) > 5:
    raise SystemExit('RTC 与本地时间不一致；不能判定双系统修复已完成。')
print('RTC 数值与本地时间一致。')
PY
```

## 6. 双系统切换验收

Ubuntu 重启后的自动校时与 RTC 持续运行检查都通过后，再进行：

1. 重启进入 Windows，确认时区与实际所在地一致，并与手机时间比较。
2. 再重启回 Ubuntu，检查本地时间和实际 RTC。
3. 如果刚进入系统时已经偏差 8 小时，后来联网才被纠正，说明硬件时钟约定仍有问题，不能只凭联网后的时间正常就判定成功。

Ubuntu 侧应满足：

```text
Timezone=Asia/Shanghai
LocalRTC=yes
NTP=yes
NTPSynchronized=yes
```

还应满足：

- `/etc/adjtime` 第三行为 `LOCAL`。
- 实际 RTC 数值接近本地时间，并在重启、持续运行后保持一致。
- `systemd-timesyncd` 为 `enabled`，日志显示成功联系可用服务器。
- 后续 `Packet count` 继续增加，`Offset` 不再是 8 小时量级。
- 独立 NTP 查询的偏差接近零；本机修复后曾测得约 12 毫秒，但换机后不要求达到同一个毫秒数。
- Windows → Ubuntu 实际切换通过。

## 与视频方案的对应关系

参考视频：[Windows11 安装 Ubuntu 避坑指南，第 18 节“系统时间同步问题”](https://www.bilibili.com/video/BV1Cc41127B9/?p=18)。以下比较依据提供的截图，未将完整视频当作已核实的操作记录。

| 项目 | 截图中的方案 | 本文方案 |
| --- | --- | --- |
| 网络校时工具 | 安装 `ntpdate` | 复用 `systemd-timesyncd` 自动校时服务 |
| 写入本地硬件时钟 | `sudo hwclock --localtime --systohc` | 校准后执行 `sudo timedatectl set-local-rtc 1`，必要时先切到 `0` 再切回 `1` |
| 自动校时服务器超时 | 截图未完整展示对应操作 | 实测服务器，配置可用端点，并验证服务实际同步 |

`hwclock` 意思是“硬件时钟”，`--localtime` 意思是“本地时间”，`--systohc` 表示“系统时间（sys）写入（to）硬件时钟（hc）”。二者的核心目的相同：统一本地 RTC 约定，并写入正确时间。

本机原本没有 `hwclock` 和 `ntpdate`，使用现有系统工具即可完成以上操作。为了避免额外安装，本文不把它们作为必需步骤。

## 常见问题

### 服务开启，但 `NTPSynchronized=no`

检查有效配置、当前选择的服务器、网络与日志：

```bash
systemd-analyze cat-config systemd/timesyncd.conf
timedatectl show-timesync --all
journalctl -u systemd-timesyncd -b --no-pager -n 40
```

重新运行第 2 节的 NTP 查询，区分域名解析、服务器响应和 UDP 123 连通性问题。如果所有候选服务器均无响应，应继续排查网络、代理和防火墙，不要把“配置已经写入”当作同步成功。

### `timesync-status` 仍显示 `Offset: -8h ...`

首次校时的测量值可能记录的是修正前偏差。查看当前实际时间，并等待下一次自动校时数据包，再检查 `Packet count` 和 `Offset`。不应为了清除这一显示而再次手动减 8 小时。

### 本地 RTC 模式出现警告

`timedatectl status` 会提示本地 RTC 在时区变化、夏令时切换方面存在限制。这是本地 RTC 模式的说明，不表示 NTP 同步失败。本文针对固定北京时间、与 Windows 默认模式一致的双系统使用方式；如果以后更换时区或希望两边统一为 UTC，需要同时考虑两个系统的约定。

### `LocalRTC=yes`，但 RTC 仍显示 UTC 数值

先完成第 5 节的首次 Ubuntu 重启与持续运行检查。若重启后仍发生，不要反复只写 `LOCAL` 或只看同步标志；检查启动日志、其他校时服务、RTC 写入程序及 Windows 的 UTC 注册表设置，继续查清实际写入方。

### `sudo` 要求密码

这是修改系统配置和 RTC 的正常权限要求。手动操作时在本机终端输入；有桌面认证代理时，助手也可以用 `pkexec` 执行准备好的修复脚本，由用户在 Ubuntu 认证窗口输入密码。临时脚本应先检查 Bash 语法，结束后删除，不保存密码。

## 回退方法

回退前根据目标决定：是恢复修改前状态，还是把 Windows 和 Ubuntu 一起改为 UTC。单独把 Ubuntu 改回 UTC、Windows 仍用本地 RTC，会重新产生两边约定不一致的问题。

1. 从备份目录的 `properties-before.txt` 或 `status-before.txt` 确认原始 `LocalRTC`、时区、NTP 状态。
2. 如果原先存在 `90-dual-boot-time.conf`，恢复它；如果备份里有 `drop-in.was-absent`，只删除本次新增的这一文件。
3. 在系统时间准确的前提下恢复原来的 RTC 模式，由 `timedatectl` 按原模式重写硬件时钟。
4. 如原先有 `/etc/adjtime`，恢复备份；如果有 `adjtime.was-absent` 且原先为 UTC，可在设置回 UTC 后删除本次生成的 `/etc/adjtime`。
5. 恢复记录的时区和校时服务启用状态。若继续使用原有自动校时，重启服务并重新验证。
6. 改变 RTC 模式后，安排重启与实际 RTC 验证。

以本次原状态为例：原先 RTC 模式为 UTC，NTP 已启用，目标配置片段和 `/etc/adjtime` 都不存在。确认备份记录匹配后，删除片段、保持自动校时、在时间准确时执行 `sudo timedatectl set-local-rtc 0`，再删除本次生成的 `/etc/adjtime`。不要把这个例子直接当作其他机器的原始状态。

只回退本次 NTP 配置片段、不改变 RTC 时，本次原先文件不存在的情况下可执行：

```bash
sudo rm -- /etc/systemd/timesyncd.conf.d/90-dual-boot-time.conf
sudo systemctl restart systemd-timesyncd
timedatectl status
```

若原先已有同名片段，应恢复备份而不是删除。原来的 `ntp.ubuntu.com` 超时也可能随配置回退再次出现。

## 本次实际记录与待完成验收

本次修复实际创建了：

```text
/etc/systemd/timesyncd.conf.d/90-dual-boot-time.conf
/etc/adjtime
```

本次备份目录：

```text
/var/backups/dual-boot-time.9rgpKuYp
```

该路径只对应当前机器，新机器应使用脚本实际输出的备份目录。本次备份记录在 `status-before.txt` 中；本文复现脚本另外增加了机器可读属性及服务启用状态记录。

截至记录时：

- 本地时间已从快约 8 小时恢复正确，自动校时成功。
- `/etc/adjtime` 第三行为 `LOCAL`，`LocalRTC=yes`。
- `systemd-timesyncd` 开机启用；已观察到连续校时，数据包数量达到 6，最近一次测量偏差约为 `-1.179ms`。
- 本机当前 `NTP=` 为两个 Cloudflare IP，`FallbackNTP=` 为阿里云。本文复现配置把阿里云加入 `NTP=`，使其成为实际会尝试的候选，而不是把 `FallbackNTP` 误当作连接失败后的备用列表。
- 后续复查观察到实际 RTC 又显示 UTC 数值：本地约 11:31，RTC 约 03:31。因此仍需首次重启回 Ubuntu、持续运行复查和 Windows → Ubuntu 切换验收，不能写成全部验收完成。
- 临时修复脚本和认证进程已清理，正式配置与备份保留。

## 参考资料

- [systemd：timedatectl 命令说明](https://github.com/systemd/systemd/blob/v255/man/timedatectl.xml)，说明 RTC 模式、写入方向及 `--adjust-system-clock`。
- [systemd：timesyncd.conf 配置说明](https://github.com/systemd/systemd/blob/v255/man/timesyncd.conf.xml)，说明 `NTP=` 与 `FallbackNTP=` 的实际语义。
- [Cloudflare：NTP 服务使用说明](https://developers.cloudflare.com/time-services/ntp/usage/)，列出本文使用的两个服务器 IP。
- [mihomo：DNS 配置说明](https://wiki.metacubex.one/config/dns/)，说明 fake IP 模式与过滤配置。
- [systemd：启动时初始化时钟](https://github.com/systemd/systemd/blob/v255/src/core/main.c)，参见 `initialize_clock()` 对本地 RTC 的启动处理。
- [Linux：周期性 RTC 更新代码](https://github.com/torvalds/linux/blob/master/kernel/time/ntp.c)，参见 `persistent_clock_is_local` 对 RTC 写入值的影响。
