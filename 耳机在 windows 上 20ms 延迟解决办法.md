# 耳机在 windows 上 20ms 延迟解决办法

最终结论：20 秒延迟由 Windows 与耳机的 HFP 免提通话组件协商引起。禁用 `Hands-Free AG` 后，A2DP 音频端点可以快速启用。

## 下次换耳机或恢复出厂后的检查流程

### 1\. 先正常配对并测试

耳机开机连接 Windows，观察：

- 蓝牙是否立即显示“已连接”

- 声音输出设备是否要等约 20 秒才出现

如果新耳机没有延迟，不需要做任何修改。

### 2\. 排除其他设备争抢

测试时暂时关闭手机、平板的蓝牙或耳机的双设备连接功能，然后重新连接 Windows。

如果仍固定延迟，再继续下面的操作。

### 3\. 关闭“免提电话服务”

按 `Win + R`，输入：

```Plaintext
explorer.exe shell:::{A8A91A66-3A7D-4424-8D24-04E180695C7A}
```

然后：

1. 找到对应蓝牙耳机。

2. 右键 →“属性”→“服务”。

3. 取消“免提电话服务/Handsfree Telephony”。

4. 点击“应用”。

5. 耳机关机再开机测试。

### 4\. 完全禁用剩余的 Hands\-Free AG

恢复出厂或重新配对后，设备实例 ID 通常会改变，因此不要直接复用旧命令。

先连接耳机，然后以管理员身份打开 PowerShell，查找当前 HFP AG：

```PowerShell
# 只列出免提音频网关设备，不进行修改
Get-PnpDevice -PresentOnly |
    Where-Object { $_.FriendlyName -match 'Hands-Free AG' } |
    Format-List FriendlyName, Status, InstanceId
```

确认名称属于目标耳机后，复制它的 `InstanceId`，执行：

```PowerShell
# 把尖括号中的内容替换为刚才查到的完整 InstanceId
pnputil /disable-device "<耳机的 Hands-Free AG InstanceId>"
```

然后把耳机关机再开机测试。

不要使用模糊匹配自动批量禁用，以免同时禁用其他耳机。

### 5\. 需要恢复耳机麦克风时

先恢复 AG：

```PowerShell
pnputil /enable-device "<耳机的 Hands-Free AG InstanceId>"
```

再进入设备属性 →“服务”，重新勾选：

```Plaintext
免提电话服务/Handsfree Telephony
```

耳机关机再开机即可。

## 禁用后的影响

- A2DP 高音质播放：正常

- 听音乐、看视频：正常

- 耳机麦克风：不可用

- 微信、会议和游戏语音：需要使用电脑内置麦克风

- 如果重新启用 HFP：20 秒延迟可能重新出现

## 驱动和旧设备检查

如果所有耳机在 Windows 下都出现延迟，再检查：

- AX210 蓝牙驱动是否为英特尔官方新版本

- 设备管理器是否残留灰色的旧蓝牙适配器

- 不要删除状态为 `Started/OK` 的 Intel AX210 和 Enumerator

- 旧设备清理不是每次重新配对都需要做

每次恢复耳机出厂设置后，Windows 可能创建新的 `InstanceId`，所以正确做法是重新查询新的 `Hands-Free AG`，再对新的实例执行禁用。

