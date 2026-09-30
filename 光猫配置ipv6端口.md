# TEWA-1006G：持久放行 IPv6 TCP 22 的复现手册

## 给接手 AI 的任务

在用户授权管理的中国电信天邑 TEWA-1006G 光猫上，允许互联网通过 IPv6 访问所有内网设备的 TCP 22，并使用固件原生配置持久保存。保留其他防火墙规则，完成后关闭临时 Telnet/FTP，不添加常驻脚本。先检查现有状态；如果目标配置已经存在且正确，直接验证，不要重复添加。

本文来自 2026-09-29 的一次实际操作。下面明确区分实测结果与尚未验证事项。操作可能因地区固件而不同，不能盲目套用对象编号、接口名或会话值。不得恢复出厂、修改宽带认证、关闭整个防火墙或修改运营商远程管理来实现本任务。

## 1. 实测环境、结果和限制

| 项目 | 本次实测值 |
|---|---|
| 型号 | TEWA-1006G，XG-PON 天翼网关（双频 WiFi6） |
| 软件版本 | V1.0.0；系统文件时间显示 2022 年构建痕迹 |
| 网关 IPv4 | 192.168.1.1 |
| 简化管理入口 | http://192.168.1.1/cgi-bin/luci/ |
| 独立管理入口 | http://192.168.1.1:8080/login.html |
| 互联网接口 | ppp1.2 |
| LAN 桥 | br0 |
| 实测互联网连接对象 | InternetGatewayDevice.WANDevice.1.WANConnectionDevice.2.WANPPPConnection.1 |
| 原生防火墙例外对象类型 | X_BROADCOM_COM_FirewallException，MDM OID 136 |
| 新规则名 | SSH22_IPv6 |
| 本次新增实例 | 上述互联网连接下的 X_BROADCOM_COM_FirewallException.1. |

已完成：原生例外配置创建、启用、`qoecmd save` 返回 `config saved.`、导出配置包含该对象；禁用再启用该对象能够删除再生成运行规则；旧手工规则已移除；临时 Telnet 已关闭，最终 TCP 21、22、23 均拒绝连接，8080 管理端口仍可用。

**未做光猫重启实测，也未在最终操作后独立验证手机外网 SSH 登录成功。** 持久化依据是固件原生配置保存成功，不应声称已完成重启验收。运营商后续重置、固件升级或配置下发能否覆盖规则也未验证。

**原生例外的实际范围：** 固件会同时在 `forward_firewall` 和 `input_firewall` 添加 TCP 22 允许规则。因此既允许转发到内网设备，也允许访问光猫自身的 TCP 22。本次光猫未运行 SSH，22 端口拒绝连接。若用户只授权 LAN 转发且禁止光猫本身的例外，不可直接应用本文方案，应先另行研究或取得范围确认。

本文件不包含真实管理密码、临时密码、会话令牌、宽带密码或完整配置备份。

## 2. 诊断前提

先确认电脑的 SSH 服务监听 IPv6，手机流量具有 IPv6 出网能力：

```bash
# 在待访问电脑上
ip -6 addr show
ss -lnt '( sport = :22 )'

# 在手机 Termux 上，关闭 Wi-Fi 后
curl -q --noproxy '*' -6 -v --connect-timeout 10 --max-time 15 https://api64.ipify.org
ssh -6 -vv -o ConnectTimeout=10 用户名@电脑当前公网IPv6
```

本次手机可成功访问 IPv6 HTTPS，但 SSH 建立 TCP 连接超时。光猫后来读出的规则证实存在 WAN 入站丢弃规则。单凭超时不能断言是光猫；必要时在电脑执行 `sudo timeout 30 tcpdump -ni any 'ip6 and tcp port 22'` 并让手机发起连接。

## 3. 管理入口与临时 Telnet

简化管理界面没有所需配置。8080 后台虽有防火墙页面，但总“启用”未勾选仍存在底层 IPv6 入站 DROP。

本次用用户提供的 `telecomadmin` 账号及密码可以登录；不要把用户名或任何默认密码当成跨设备通用凭据。`enableTelnet.html` 在本固件上访问后跳回登录页，未成功开启服务。

### 用户手动开启的成功方式

1. 用户登录 `http://192.168.1.1:8080/login.html`。
2. 在该页面浏览器开发者工具 Console 中读取 `sessionKey`，得到当前会话值。只读取此变量，不需要粘贴整段未知脚本。
3. 用户自行生成临时强密码（例如至少 20 位随机大小写字母和数字），选择临时账号。下例用 `telcheck`。
4. 用户在新标签页地址栏打开以下模板，替换占位符：

```text
http://192.168.1.1:8080/telandftpcfg.cmd?action=add&telusername=telcheck&telpwd=TEMP_PASSWORD&telport=23&telenable=1&sessionKey=CURRENT_SESSION_KEY
```

请求会更改管理服务配置。不要把完整地址发送给第三方或写入文档；地址包含密码和令牌，可能留在浏览历史中。重新登录后要重新读取会话值。

本次返回页面显示 Telnet 23 已勾选，FTP 未勾选。仅传入 Telnet 参数在本固件上成功，但**不能保证其他固件省略 FTP 参数不会影响原 FTP 配置**，操作前后都要检查。本次后来读出 `TelnetdCfg.NetworkAccess=LAN`，WAN Telnet 开关为 FALSE；其他设备必须独立验证访问范围。

涉及新管理凭据时，接手 AI 应遵守所在工具的凭据创建/安全访问确认要求；本次创建和提交由用户完成。

## 4. 登录、权限和安全输出

从同一局域网使用 Telnet 客户端连接：

```text
telnet 192.168.1.1 23
Login: telcheck
Password: 用户设置的临时密码
```

本次普通提示符为 `$`。该会话具有命令过滤/代理行为：能执行 `ip6tables`，但普通进程 UID 为 100，`qoecmd` 和停止服务会报权限不足。不能仅凭能执行 ip6tables 就判断获得 root。

本固件实测：在 Telnet 会话输入单独一个分号并回车：

```sh
;
```

会输出 `sh: syntax error: unexpected ";"`，随后同一连接中执行：

```sh
cat /proc/self/status | grep Uid
```

实测四列 UID 从 100 变成 0。这是该固件的异常权限切换行为，**不保证适用于其他版本；必须只用于用户授权的设备，并核实输出**。新建连接需要重新检查权限。本次不需要 `su` 密码；尝试把网页密码当作 su 密码失败。不要重复猜密码。

Telnet 是明文协议，限可信局域网使用。读取完整配置会暴露宽带和管理凭据；尽量用 `getpv`/指定对象读取，自动化时在内存中筛选输出，绝不打印完整 `dumpcfg`/`dumpmdm`。本机缺少部分标准命令，BusyBox `grep` 不支持 `-A/-B`。

## 5. 只读探测与备份

在确认 root 的同一 Telnet 会话执行：

```sh
ip -6 route show
ip6tables -S forward_firewall
ip6tables -S input_firewall
qoecmd mdm -h
qoecmd mdm lookup | grep -i -e firewall -e filter
qoecmd dumpmdm | grep IPv6DNSWANConnection
```

本次得到 `ppp1.2` 和完整路径：

```text
InternetGatewayDevice.WANDevice.1.WANConnectionDevice.2.WANPPPConnection.1
```

核对其接口名：

```sh
qoecmd mdm getpv InternetGatewayDevice.WANDevice.1.WANConnectionDevice.2.WANPPPConnection.1.X_BROADCOM_COM_IfName
```

预期本次为 `ppp1.2`。若实际不同，后续所有路径都用实际值。若连接是 WANIPConnection，不能照搬 WANPPPConnection 路径和 OID。

原来的入站策略核心为：

```text
-A forward_firewall -i ppp1.2 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A forward_firewall -i ppp1.2 -j DROP
```

检查既有例外，避免重复添加：

```sh
qoecmd mdm dumpobj 136
```

备份配置到光猫持久分区，权限仅限管理者。先检查目标文件是否存在；不要覆盖原备份。以下是本次已创建的文件名：

```sh
umask 077
qoecmd dumpcfg > /data/codex-before-ssh22.cfg
ls -l /data/codex-before-ssh22.cfg
```

本次文件约 158 KB、权限 600。完整配置含敏感信息，不要上传。本次 `/data` 是可写 JFFS2，`/etc` 属于只读 squashfs；这也是没有采用 rc.local 脚本的原因。

## 6. 创建原生持久例外

如果没有相同配置，创建实例：

```sh
qoecmd mdm addobj InternetGatewayDevice.WANDevice.1.WANConnectionDevice.2.WANPPPConnection.1.X_BROADCOM_COM_FirewallException.
```

**读取返回的新实例路径**。本次是 `.1.`，不能假定每台设备都是 1。新对象本次默认 `Enable=0`，先检查并保持禁用再设置属性：

```sh
qoecmd mdm getov InternetGatewayDevice.WANDevice.1.WANConnectionDevice.2.WANPPPConnection.1.X_BROADCOM_COM_FirewallException.1.
```

本次配置值：

| 属性 | 值 |
|---|---|
| Enable | 最后设为 1 |
| FilterName | SSH22_IPv6 |
| IPVersion | 6 |
| Protocol | TCP |
| DestinationPortStart | 22 |
| DestinationPortEnd | 22 |
| SourcePortStart / SourcePortEnd | 保留默认 0（不限制） |
| SourceIPAddress / SourceNetMask | 保留空 |
| DestinationIPAddress / DestinationNetMask | 保留空（所有目标） |

以下为本次验证成功的单条多参数命令；其他设备须替换全部对象路径：

```sh
qoecmd mdm setpv InternetGatewayDevice.WANDevice.1.WANConnectionDevice.2.WANPPPConnection.1.X_BROADCOM_COM_FirewallException.1.FilterName SSH22_IPv6 InternetGatewayDevice.WANDevice.1.WANConnectionDevice.2.WANPPPConnection.1.X_BROADCOM_COM_FirewallException.1.IPVersion 6 InternetGatewayDevice.WANDevice.1.WANConnectionDevice.2.WANPPPConnection.1.X_BROADCOM_COM_FirewallException.1.Protocol TCP InternetGatewayDevice.WANDevice.1.WANConnectionDevice.2.WANPPPConnection.1.X_BROADCOM_COM_FirewallException.1.DestinationPortStart 22 InternetGatewayDevice.WANDevice.1.WANConnectionDevice.2.WANPPPConnection.1.X_BROADCOM_COM_FirewallException.1.DestinationPortEnd 22
```

确认各字段后启用：

```sh
qoecmd mdm setpv InternetGatewayDevice.WANDevice.1.WANConnectionDevice.2.WANPPPConnection.1.X_BROADCOM_COM_FirewallException.1.Enable 1
ip6tables -S forward_firewall
ip6tables -S input_firewall
```

本次固件自动在两个链前部都生成：

```text
-i ppp1.2 -p tcp -m tcp --dport 22 -j ACCEPT
```

无需清空防火墙，也无需自行写开机脚本。

## 7. 保存与验证

```sh
qoecmd save
```

必须看到 `config saved.`。再读取 `qoecmd dumpcfg`，只提取名为 `SSH22_IPv6` 的 FirewallException XML 元素并核对，不输出其他配置。预期：

```xml
<X_BROADCOM_COM_FirewallException instance="1">
  <Enable>TRUE</Enable>
  <FilterName>SSH22_IPv6</FilterName>
  <IPVersion>6</IPVersion>
  <Protocol>TCP</Protocol>
  <DestinationPortStart>22</DestinationPortStart>
  <DestinationPortEnd>22</DestinationPortEnd>
</X_BROADCOM_COM_FirewallException>
```

注意同名 `nextInstance` 元素可能单行自闭合。不要使用宽泛的 sed 范围匹配，否则可能输出后面的完整敏感配置。建议完整输出仅在程序内存中解析 XML，最终只打印匹配的对象。

本次曾有手工临时规则，仅在**确认精确匹配存在且确系本次添加**时移除：

```sh
ip6tables -D forward_firewall -i ppp1.2 -o br0 -p tcp --dport 22 -j ACCEPT
```

本次进行了规则重新生成验证：把原生对象 Enable 设为 0，核实允许规则消失；再设为 1，核实自动出现；再次 `qoecmd save`。这会短暂影响 SSH 新连接，应在无重要会话时做。此验证不能代替重启验收。

让用户使用手机流量连接内网设备的当前公网 IPv6。完全验收需在用户可接受断网时重启光猫，重新确认地址、外网连接与配置仍存在。不要擅自宣称已通过重启测试。

## 8. 关闭临时 Telnet 和清理

本次查到的服务控制字段：

```sh
qoecmd mdm getpv InternetGatewayDevice.DeviceInfo.X_CT-COM_ServiceManage.TelnetEnable
```

关闭 Telnet 可能直接结束当前会话，因此需让保存命令在断连后完成。本次以下短时后台子 shell 成功执行；不会建立常驻服务：

```sh
(trap '' HUP; sleep 2; qoecmd mdm setpv InternetGatewayDevice.DeviceInfo.X_CT-COM_ServiceManage.TelnetEnable 0; qoecmd save) </dev/null >/var/codex-telnet-cleanup.log 2>&1 &
```

随后断开 Telnet，数秒后从本机检查 23 端口。本次已确认拒绝连接；21 端口也拒绝连接。关闭后的 `save` 日志未再次读取，所以不能将“端口已关闭”单独视为“Telnet 重启后仍关闭”的完整证明；重启验收时一并验证。

本次 `sh -c '...'` 形式曾被命令过滤返回 `Error!`，上面的括号子 shell 形式有效。普通 UID 100 下 `killall telnetd` 会报权限不足，而且仅杀进程不是持久关闭方法。

保留备份用于回滚；不要删除用户原有文件。关闭所有测试会话，不留下定时任务、守护进程或长期后台轮询。光猫 `/var/codex-telnet-cleanup.log` 是临时文件，`/var` 为 tmpfs。

## 9. 回滚

重新建立用户授权的维护访问后，先读取实例并确认 `FilterName=SSH22_IPv6`，仅删除本任务创建的实例：

```sh
qoecmd mdm delobj InternetGatewayDevice.WANDevice.1.WANConnectionDevice.2.WANPPPConnection.1.X_BROADCOM_COM_FirewallException.1.
qoecmd save
ip6tables -S forward_firewall
ip6tables -S input_firewall
```

上述删除路径应以实际新增实例为准。`delobj` 语法由设备帮助确认，但本次未执行最终删除回滚；执行后必须检查实际结果。不要直接恢复整个配置备份或恢复出厂，因为可能覆盖其他业务设置。回滚完成同样关闭临时 Telnet。

## 10. 接手 AI 的完成报告

明确报告：实际对象路径、规则范围、保存响应、规则重建验证结果、是否进行手机外网验证/光猫重启测试、Telnet/FTP 端口关闭状态、备份位置。遇到型号差异、路径不匹配、命令无效时停止依赖该假设的修改，保留已完成事实；不要把未验证步骤说成成功。

## 参考线索（不是跨型号保证）

- TEWA-1006G Telnet 讨论：https://www.chinadsl.net/forum.php?mod=viewthread&tid=177433
- 隐藏接口实现参考：https://github.com/invinciberry/tewa-modem-extractor/blob/main/modem_extract.py
- 命令行权限异常与 IPv6 规则讨论：https://github.com/jonirrings/xor/issues/24
- MDM/save 持久配置讨论：https://github.com/jonirrings/xor/issues/28#issuecomment-2380405814

最后一个讨论使用关闭整个防火墙的方法；本次没有采用，改用设备原生 FirewallException 精确放行 TCP 22。不要直接运行参考仓库脚本，它可能同时启用 FTP、改管理凭据或执行其他超出范围的操作。


## 后续新增：游戏与预留端口（2026-09-29）

同一 WANPPPConnection 下新增并启用原生 IPv6/TCP 例外：

| 实例 | 名称 | 目标端口 |
|---|---|---|
| 2 | GAME8088_IPv6 | 8088 |
| 3 | RESERVE18080_18089 | 18080–18089 |

均不限制目标设备，已核对 forward_firewall 规则且 `qoecmd save` 返回成功。未做重启或外网端口实测。原生对象也会生成光猫本身的 INPUT 例外。临时 Telnet 已关闭并验证 23 端口拒绝连接，FTP 未开启。预留端口不占用端口，但任何内网设备在该范围运行服务均可能被外网访问。

QQ 飞车静态页面运行在 TCP 8088，服务为用户级临时 systemd 单元 `qq-speed.service`；重启后不会自动恢复。停止命令：`systemctl --user stop qq-speed.service`。
