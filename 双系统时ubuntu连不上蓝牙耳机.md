# 双系统时ubuntu连不上蓝牙耳机

## 问题背景

Windows 和 Ubuntu 双系统共用同一个蓝牙适配器时，蓝牙耳机会分别和两个系统生成配对密钥。耳机通常只认最后一次成功配对的密钥，所以在 Windows 里重新连接成功后，切回 Ubuntu 可能会出现无法连接。

本次设备：

- 耳机：`ROSE OpenFeel`

- 耳机 MAC：`41:42:C9:FF:6D:3B`

- Ubuntu 蓝牙适配器 MAC：`BC:CD:99:B2:13:C5`

- Windows 注册表里对应的适配器目录名：`bccd99b213c5`

- Windows 注册表里对应的耳机值名：`4142c9ff6d3b`

注意：`LinkKey` 是蓝牙配对密钥，不建议写进公开仓库或截图外发。本文只记录方法，不记录本次真实密钥。

## 适用场景

这个方法适合下面这种情况：

- 同一副耳机在 Windows 能连上，切到 Ubuntu 连不上或者频繁断开。

- 耳机刚恢复出厂设置，旧的 Ubuntu 配对记录已经失效。

- 已经在 Windows 删除旧设备、重新配对，并确认能连接。

- 现在需要把 Windows 新生成的蓝牙配对密钥同步到 Ubuntu。

## 整体思路

1. 在 Windows 里重新配对耳机，让 Windows 生成新的 `LinkKey`。

2. 回到 Ubuntu，只读挂载 Windows 分区。

3. 从 Windows 的 `SYSTEM` 注册表 hive 导出蓝牙配对密钥。

4. 找到当前 Ubuntu 蓝牙适配器和耳机对应的那条 `LinkKey`。

5. 写入 Ubuntu 的 BlueZ 配置目录。

6. 重启蓝牙并直接连接耳机。

不要在 Ubuntu 里重新 `pair` 这副耳机，否则可能会覆盖刚同步好的密钥。

## Windows 侧操作

在 Windows 里先完成：

1. 删除 `ROSE OpenFeel` 这个蓝牙设备。

2. 让耳机进入配对模式。

3. 在 Windows 里重新配对。

4. 确认耳机能在 Windows 里正常连接。

5. 重启进入 Ubuntu。

如果耳机恢复出厂设置过，必须重新做这一步。恢复出厂设置前的旧 `LinkKey` 不能再用。

## Ubuntu 侧确认设备状态

查看 Ubuntu 当前蓝牙适配器：

```Bash
bluetoothctl show
```

本次看到的适配器是：

```Plaintext
Controller BC:CD:99:B2:13:C5 did [default]
```

查看已配对设备：

```Bash
bluetoothctl devices Paired
```

恢复出厂设置后，`ROSE OpenFeel` 一开始不在已配对列表里。即使 `bluetoothctl devices` 能看到它，也可能只是扫描缓存，不代表已经配对。

## 只读挂载 Windows 分区

先找到 Windows 分区：

```Bash
lsblk -f
```

本次 Windows 分区是：

```Plaintext
/dev/nvme0n1p3
```

只读挂载：

```Bash
udisksctl mount --block-device /dev/nvme0n1p3 --options ro
```

本次挂载位置是：

```Plaintext
/media/did/Windows
```

## 导出 Windows 蓝牙密钥

需要系统里有 `reged`，通常来自 `chntpw` 包。

确认工具存在：

```Bash
which reged
which chntpw
```

导出 Windows 注册表里的蓝牙密钥分支：

```Bash
reged -x /media/did/Windows/Windows/System32/config/SYSTEM \
  HKEY_LOCAL_MACHINE\\SYSTEM \
  \\ControlSet001\\Services\\BTHPORT\\Parameters\\Keys \
  /tmp/bluetooth_keys.reg
```

查看导出的内容：

```Bash
sed -n '1,220p' /tmp/bluetooth_keys.reg
```

本次关键结构类似这样：

```Plaintext
[HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Services\BTHPORT\Parameters\Keys\bccd99b213c5]
"4142c9ff6d3b"=hex:<这里是 16 字节的 Windows LinkKey>
```

含义：

- `bccd99b213c5`：电脑蓝牙适配器 MAC，来自 `BC:CD:99:B2:13:C5`，去掉冒号并转小写。

- `4142c9ff6d3b`：耳机 MAC，来自 `41:42:C9:FF:6D:3B`，去掉冒号并转小写。

- `hex:...` 后面的 16 字节就是要同步到 Ubuntu 的 `LinkKey`。

把 `hex:` 后面的字节去掉逗号并转成 32 位十六进制字符串，例如：

```Plaintext
hex:aa,bb,cc,dd,ee,ff,00,11,22,33,44,55,66,77,88,99
```

转换成：

```Plaintext
AABBCCDDEEFF00112233445566778899
```

## 写入 Ubuntu BlueZ 配置

Ubuntu 的 BlueZ 配对文件路径格式是：

```Plaintext
/var/lib/bluetooth/<电脑蓝牙MAC>/<耳机MAC>/info
```

本次路径是：

```Plaintext
/var/lib/bluetooth/BC:CD:99:B2:13:C5/41:42:C9:FF:6D:3B/info
```

写入前建议先备份：

```Bash
sudo systemctl stop bluetooth
sudo cp -a /var/lib/bluetooth /var/lib/bluetooth.bak.$(date +%Y%m%d-%H%M%S)
```

创建目录并写入配置：

```Bash
sudo mkdir -p /var/lib/bluetooth/BC:CD:99:B2:13:C5/41:42:C9:FF:6D:3B
sudo nano /var/lib/bluetooth/BC:CD:99:B2:13:C5/41:42:C9:FF:6D:3B/info
```

内容示例：

```Plaintext
[General]
Name=ROSE OpenFeel
Alias=ROSE OpenFeel
Class=0x240404
SupportedTechnologies=BR/EDR;
Trusted=true
Blocked=false

[LinkKey]
Key=<从 Windows 复制并转换后的 32 位十六进制 LinkKey>
Type=4
PINLength=0
```

保存后设置权限并启动蓝牙：

```Bash
sudo chmod 600 /var/lib/bluetooth/BC:CD:99:B2:13:C5/41:42:C9:FF:6D:3B/info
sudo systemctl start bluetooth
```

## 连接耳机

不要重新配对，直接连接：

```Bash
bluetoothctl connect 41:42:C9:FF:6D:3B
```

本次连接成功时输出里有：

```Plaintext
Connection successful
```

确认状态：

```Bash
bluetoothctl info 41:42:C9:FF:6D:3B
```

成功状态类似：

```Plaintext
Name: ROSE OpenFeel
Paired: yes
Bonded: yes
Trusted: yes
Connected: yes
```

## 卸载 Windows 分区

完成后卸载刚才只读挂载的 Windows 分区：

```Bash
udisksctl unmount --block-device /dev/nvme0n1p3
```

## 常见问题

### `bluetoothctl remove` 提示设备不可用

如果出现：

```Plaintext
Device 41:42:C9:FF:6D:3B not available
```

说明它可能只是扫描缓存中的设备，不是 Ubuntu 当前已配对设备。可以用下面命令确认：

```Bash
bluetoothctl devices Paired
bluetoothctl devices Bonded
bluetoothctl devices Trusted
bluetoothctl devices Connected
```

如果这些列表里都没有它，就没有正式配对记录可删。

### 导出的注册表里没有耳机 MAC

检查：

- Windows 里是否真的重新配对并连接成功。

- 耳机 MAC 是否一致。

- 是否使用了正确的电脑蓝牙适配器分支。

- 如果 `ControlSet001` 找不到，可以检查 `ControlSet002`，或者查看 `HKEY_LOCAL_MACHINE\SYSTEM\Select` 判断当前控制集。

### 写入后仍然连不上

可以检查：

- `Key=` 是否正好是 32 位十六进制字符。

- 是否去掉了逗号和 `hex:`。

- 是否写进了正确的适配器 MAC 目录。

- 是否在 Ubuntu 里误点了重新配对。

- 耳机是否还连接着 Windows 或手机。

## 本次最终结果

写入 Windows 新生成的 `LinkKey` 后，Ubuntu 中 `ROSE OpenFeel` 状态为：

```Plaintext
Paired: yes
Bonded: yes
Trusted: yes
Connected: yes
Battery Percentage: 100%
```

问题解决。

