# root 用户下去广告

记录日期：2026-09-30。设备为 Redmi K70 Pro / HyperOS 2；Root 来源及限制见同目录《根据 GhostLock 进行 root 提权.md》。

## 1. 本次实际完成的范围

通过 ADB + Root 精确停用组件，Blocker 用于发现、归类和查看组件。没有卸载整个应用，也没有批量禁用某个 SDK 的全部匹配项。

- 第一轮：9 个应用、57 个广告展示 Activity。
- 第二轮：新增 81 个组件，扩大到出行应用、部分系统广告组件和应用自有广告页面。
- 合计：**28 个应用/系统包，138 个组件**；已逐项确认 Package Manager 中处于 disabled 状态。
- 第一轮 9 个应用、本轮 24 个用户应用通过启动检查，检查期间未发现对应启动崩溃。两轮测试对象有重叠，不能相加当成独立应用数。
- 系统广告组件只核对了停用状态；未重启手机测试，也未逐项验证登录、支付、NFC、购票、消息收发和全部业务流程。

当前方案针对广告页面、专用广告下载服务、明确的广告通知接收器。信息流广告可能由应用主页面直接绘制，不一定存在可独立停用的组件，因此不能保证广告全部消失。

## 2. Blocker 页面上的条目是什么

“SDK/跟踪器”中列出的主要是应用内嵌的功能库或识别规则，并不全是广告，也不全是系统组件。例如推送、地图、网页内核、统计、登录服务都可能出现在其中。

右侧匹配数量和规则名称只用于筛选候选对象，不能作为整组禁用的依据。本次数据库扫描包含 444 个应用、43,746 个组件；这不是建议停用的数量。

判断原则：

1. 同时看所属应用、完整组件名、组件类型和用途。
2. 优先选择明确的广告展示页面、广告下载服务、广告通知接收器。
3. 保留通用消息推送、登录、支付、地图、网页内核、普通下载器、系统核心服务。
4. 本次没有停用 ContentProvider，也没有按 `Ad` 字符串全局匹配后全部关闭。例如 Bluetooth advertising 是蓝牙广播，Advertising ID 是广告标识接口，都不能等同于广告展示组件。
5. 规则中的“safeToBlock”不能替代设备实测；一些规则本身注明副作用未知，甚至描述与名称不一致。

## 3. 先修复 Blocker 的初始化问题

安装版本：Blocker `2.0.6339`、Shizuku `13.6.0.r1086.2650830c`。

本次确认 Shizuku 服务以 Root 运行，Blocker 已获授权，但界面一直显示“初始化数据”。数据库完整性正常，应用表和组件表却都是 0 条；HyperOS 的读取应用列表权限被拒绝：

```text
MIUIOP(10022): ignore
```

处理：

```bash
adb shell "su -c 'cmd appops set com.merxury.blocker 10022 allow'"
adb shell am force-stop com.merxury.blocker
adb shell am start -n com.merxury.blocker/.MainActivity
```

之后列表扫描正常完成。**Root/Shizuku 授权与 HyperOS 的应用列表权限是两回事。** 数字 `10022` 是本次 MIUI/HyperOS 环境的权限编号，不要当成所有 Android 厂商通用编号。

读取日志：

```bash
adb shell "su -c 'tail -80 /data/user/0/com.merxury.blocker/files/logs/2026-09-30.log'"
adb shell cmd appops get com.merxury.blocker 10022
```

日志文件名随日期变化，应先查看日志目录。排查时不必清空全机 logcat，也不要因为初始化卡住就直接清除 Blocker 数据。

## 4. 修复规则同步的 master/main 不匹配

规则源：`https://gitlab.com/mercuryli/blocker-general-rules.git`。

本次远端默认分支为 `main`，本地缓存 `.git/HEAD` 却指向没有本地提交的 `master`，目录中只有 `.git`，实际规则文件未检出。日志反复出现：

```text
RefNotAdvertisedException: Remote GITLAB did not advertise Ref for branch master
```

处理前停止 Blocker 并备份原始 `HEAD` 和 `config`，然后修正以下缓存仓库的分支与跟踪配置：

```text
/data/user/0/com.merxury.blocker/files/blocker-general-rules/.git/
```

`HEAD` 内容：

```text
ref: refs/heads/main
```

`config` 中保留原有内容，并设置：

```ini
[branch "main"]
    remote = GITLAB
    merge = refs/heads/main
```

使用 Root 写私有目录时，重定向必须放在 `su -c` 的引号内部。不要先改分支再无条件写入提交号，更不要随意删除用户自己的规则。

随后由 Blocker 的同步任务完成拉取，日志确认：

```text
Pulling changes on branch main
network_sync_successful
```

同步成功的提交为 `0f82b34e50857cc2e5fd4074376fb00fb9c7d2c3`，本地与当时远端一致，中文规则已能读取。此为本次修复记录；以后若仓库结构变化，应先重新核对远端，不能直接照抄覆盖。

## 5. 精确停用及原状态记录

本次使用 Package Manager 逐个停用，不是写入 IFW 拦截规则：

```bash
# 将示例包名、完整组件类名替换为已核实的实际值。
adb shell "su -c 'pm disable --user 0 包名/完整组件类名'"
```

每次执行前记录原始组件启用设置。本次 138 个对象原始设置均为 `0`（默认状态），不是明确的 `1`（强制启用）。因此恢复应使用 `default-state`，而不是无差别 `enable`：

```bash
adb shell "su -c 'pm default-state --user 0 包名/完整组件类名'"
```

PM 配置通常跨重启保留，但本次没有重启验证；应用更新也可能新增组件或改变行为。Root 失效与组件配置是否保留不是同一件事。

## 6. 已处理应用统计

| 应用/系统包 | 包名 | 停用组件数 |
| --- | --- | ---: |
| 北京一卡通 | `cn.com.bmac.nfc` | 10 |
| 易校园 | `cn.com.yunma.school.app` | 12 |
| 铁路12306 | `com.MobileTicket` | 2 |
| 起飞VPN | `com.ambrose.overwall` | 1 |
| bilibili | `com.bilibili.app.in` | 6 |
| 菜鸟 | `com.cainiao.wireless` | 6 |
| 酷安 | `com.coolapk.market` | 10 |
| Facebook | `com.facebook.katana` | 1 |
| BOSS直聘 | `com.hpbr.bosszhipin` | 1 |
| 京东 | `com.jingdong.app.mall` | 1 |
| 微店 | `com.koudai.weidian.buyer` | 10 |
| 酷狗概念版 | `com.kugou.android.lite` | 9 |
| Outlook | `com.microsoft.office.outlook` | 1 |
| 智能服务 | `com.miui.systemAdSolution` | 9 |
| 超级小爱 | `com.miui.voiceassist` | 1 |
| 夸克 | `com.quark.browser` | 9 |
| 搜狗输入法小米版 | `com.sohu.inputmethod.sogou.xiaomi` | 1 |
| 抖音 | `com.ss.android.ugc.aweme` | 2 |
| 抖省省 | `com.ss.android.ugc.lifeservices` | 2 |
| 闲鱼 | `com.taobao.idlefish` | 8 |
| QQ邮箱 | `com.tencent.androidqqmail` | 8 |
| 微信 | `com.tencent.mm` | 3 |
| 腾讯会议 | `com.tencent.wemeet.app` | 1 |
| 交管12123 | `com.tmri.app.main` | 1 |
| 小红书 | `com.xingin.xhs` | 4 |
| TikTok | `com.zhiliaoapp.musically` | 2 |
| 志愿汇 | `com.zzw.october` | 11 |
| 哔哩哔哩 | `tv.danmaku.bili` | 6 |

其中，小米“智能服务”仅处理广告包中的广告通知、广告推送接收器及部分广告展示服务，没有停用全机的小米通用推送服务。微信处理的是部分广告落地页，不是聊天服务，也不代表朋友圈广告本身全部被移除。

## 7. 已检查但保留的范围

- **淘宝**：没有确认能独立区分广告与订单通知的推送组件；营销消息列表页面也不等于消息发送源，因此未停用共享消息链路。
- **短信/RCS**：出现带广告字样的服务，但未充分验证与消息流程的独立性，没有修改短信核心包。
- **Google Play 服务、系统设置等**：未通过名称模糊匹配关闭广告标识、蓝牙广播或核心服务。
- **银行、支付和其他混合业务组件**：没有因为名称中包含 marketing/advert 就自动停用。
- **推送 SDK**：保留普通聊天、邮件、行程通知；只关闭能明确归属于广告用途的接收器/服务。

后续如果某个应用仍有广告，应记录具体应用、版本、广告出现位置和触发动作，再单独检查，不能以“还有广告”为由直接全禁 SDK。

## 8. 恢复方法与排障顺序

完整审计清单：

[ad-components-changes.json](/home/guga/Documents/Codex/2026-09-30/guga-adb-shell-uname-r-adb/outputs/ad-components-changes.json)

恢复脚本：

[restore-ad-components.sh](/home/guga/Documents/Codex/2026-09-30/guga-adb-shell-uname-r-adb/outputs/restore-ad-components.sh)

在连接该手机的电脑上执行；手机需要仍有 Root，并允许 ADB shell 使用 `su`。脚本指定本次设备序列号 `648718eb`。

```bash
# 恢复两轮全部组件
bash '/home/guga/Documents/Codex/2026-09-30/guga-adb-shell-uname-r-adb/outputs/restore-ad-components.sh'

# 只恢复某一个应用，例如铁路12306
bash '/home/guga/Documents/Codex/2026-09-30/guga-adb-shell-uname-r-adb/outputs/restore-ad-components.sh' com.MobileTicket

# 只恢复微信
bash '/home/guga/Documents/Codex/2026-09-30/guga-adb-shell-uname-r-adb/outputs/restore-ad-components.sh' com.tencent.mm
```

脚本恢复组件原状态，不卸载 APK、不清除应用数据、不修改用户登录信息。若后续另外调整过同一组件，执行恢复前应核对最新状态。

遇到闪退、白屏、卡开屏、广告奖励无法领取或业务异常时：

1. 先按应用恢复本次改动。
2. 重新打开该应用并复现原操作。
3. 若恢复后正常，说明需要缩小这个应用的停用范围；不要靠清数据或重装掩盖原因。
4. 若重启后 `su` 不可用，先按 GhostLock 文档重新获得临时 Root，再执行恢复。

检查脚本及临时数据库副本已清理，没有留下持续运行的诊断任务。保留审计清单和恢复脚本作为正式恢复资料。

## 9. 广告组件改动清单（完整 138 项）

以下是本次记录，不是适用于所有版本的通用黑名单。升级应用后应重新核对。原启用设置全部为默认状态 `0`；类型 ACTIVITY / SERVICE / RECEIVER 分别为页面、服务、广播接收器。

### 北京一卡通 — `cn.com.bmac.nfc`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoActivity` |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoLandscapeActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.AdWebViewActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.KsFullScreenVideoActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.KsFullScreenLandScapeVideoActivity` |
| ACTIVITY | `com.qq.e.ads.ADActivity` |
| ACTIVITY | `com.qq.e.ads.PortraitADActivity` |
| ACTIVITY | `com.qq.e.ads.LandscapeADActivity` |
| SERVICE | `com.bytedance.sdk.openadsdk.downloadnew.ApiDownloadHandlerService` |
| SERVICE | `com.qq.e.comm.DownloadService` |

### 易校园 — `cn.com.yunma.school.app`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.qq.e.ads.ADActivity` |
| ACTIVITY | `com.qq.e.ads.PortraitADActivity` |
| ACTIVITY | `com.qq.e.ads.LandscapeADActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.AdWebViewActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.KsFullScreenVideoActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.KsFullScreenLandScapeVideoActivity` |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoActivity` |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoLandscapeActivity` |
| ACTIVITY | `com.beizi.ad.v2.activity.BeiZiNewInterstitialActivity` |
| SERVICE | `com.qq.e.comm.DownloadService` |
| SERVICE | `com.bytedance.sdk.openadsdk.downloadnew.ApiDownloadHandlerService` |
| SERVICE | `com.beizi.ad.DownloadService` |

### 铁路12306 — `com.MobileTicket`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.beizi.ad.v2.activity.BeiZiNewInterstitialActivity` |
| SERVICE | `com.beizi.ad.DownloadService` |

### 起飞VPN — `com.ambrose.overwall`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.facebook.ads.AudienceNetworkActivity` |

### bilibili — `com.bilibili.app.in`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.bilibili.ad.core.click.AdCallUpActivity` |
| ACTIVITY | `com.bilibili.ad.core.click.AdCallUpActivityWeb` |
| ACTIVITY | `com.bilibili.ad.core.click.AdCallUpActivity2` |
| ACTIVITY | `com.bilibili.ad.web.AdWebActivity` |
| SERVICE | `com.bilibili.ad.core.apkdownload.ADDownloadService` |
| SERVICE | `com.bilibili.ad.web.preload.AdWebViewPreloadService4MainProcess` |

### 菜鸟 — `com.cainiao.wireless`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoActivity` |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoLandscapeActivity` |
| ACTIVITY | `com.qq.e.ads.ADActivity` |
| ACTIVITY | `com.qq.e.ads.PortraitADActivity` |
| ACTIVITY | `com.qq.e.ads.LandscapeADActivity` |
| SERVICE | `com.qq.e.comm.DownloadService` |

### 酷安 — `com.coolapk.market`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.qq.e.ads.ADActivity` |
| ACTIVITY | `com.qq.e.ads.PortraitADActivity` |
| ACTIVITY | `com.qq.e.ads.LandscapeADActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.AdWebViewActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.KsFullScreenVideoActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.KsFullScreenLandScapeVideoActivity` |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoActivity` |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoLandscapeActivity` |
| ACTIVITY | `com.coolapk.market.view.splash.SplashAdActivity` |
| SERVICE | `com.qq.e.comm.DownloadService` |

### Facebook — `com.facebook.katana`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.facebook.ads.AudienceNetworkActivity` |

### BOSS直聘 — `com.hpbr.bosszhipin`

| 类型 | 完整组件类名 |
| --- | --- |
| SERVICE | `com.hpbr.bosszhipin.service.ScreenAdvertService` |

### 京东 — `com.jingdong.app.mall`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.jingdong.app.mall.ad.ADActivity` |

### 微店 — `com.koudai.weidian.buyer`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.sigmob.sdk.base.common.AdActivity` |
| ACTIVITY | `com.sigmob.sdk.base.common.PortraitAdActivity` |
| ACTIVITY | `com.sigmob.sdk.base.common.LandscapeAdActivity` |
| ACTIVITY | `com.qq.e.ads.ADActivity` |
| ACTIVITY | `com.qq.e.ads.PortraitADActivity` |
| ACTIVITY | `com.qq.e.ads.LandscapeADActivity` |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoActivity` |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoLandscapeActivity` |
| SERVICE | `com.qq.e.comm.DownloadService` |
| SERVICE | `com.bytedance.sdk.openadsdk.downloadnew.ApiDownloadHandlerService` |

### 酷狗概念版 — `com.kugou.android.lite`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.qq.e.tg.ADActivity` |
| ACTIVITY | `com.qq.e.tg.PortraitADActivity` |
| ACTIVITY | `com.qq.e.tg.LandscapeADActivity` |
| ACTIVITY | `com.qq.e.tg.ADActivitySingleTop` |
| ACTIVITY | `com.qq.e.tg.WebAdActivity` |
| ACTIVITY | `com.qq.e.tg.TransPortraitADActivity` |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoActivity` |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoLandscapeActivity` |
| SERVICE | `com.qq.e.comm.DownloadService` |

### Outlook — `com.microsoft.office.outlook`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.facebook.ads.AudienceNetworkActivity` |

### 智能服务 — `com.miui.systemAdSolution`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.miui.app.lockscreenad.LockScreenAdClickActivity` |
| SERVICE | `com.miui.systemAdSolution.splashAd.SystemSplashAdService` |
| SERVICE | `com.miui.systemAdSolution.splashAd.ExternalMediaSplashAdService` |
| SERVICE | `com.miui.systemAdSolution.splashAd.SystemSplashExtMessageService` |
| SERVICE | `com.miui.systemAdSolution.splashscreen.SplashScreenService` |
| SERVICE | `com.miui.systemAdSolution.splashscreen.SplashScreenServiceV2` |
| SERVICE | `com.miui.zeus.mimo.msa.RemoteAdViewService` |
| RECEIVER | `com.miui.broadcast.MiPushMessageReceiver` |
| RECEIVER | `com.miui.broadcast.NotificationAdReceiver` |

### 超级小爱 — `com.miui.voiceassist`

| 类型 | 完整组件类名 |
| --- | --- |
| SERVICE | `com.bytedance.sdk.openadsdk.downloadnew.ApiDownloadHandlerService` |

### 夸克 — `com.quark.browser`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.qq.e.ads.ADActivity` |
| ACTIVITY | `com.qq.e.ads.PortraitADActivity` |
| ACTIVITY | `com.qq.e.ads.LandscapeADActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.AdWebViewActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.KsFullScreenVideoActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.KsFullScreenLandScapeVideoActivity` |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoActivity` |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoLandscapeActivity` |
| SERVICE | `com.qq.e.comm.DownloadService` |

### 搜狗输入法小米版 — `com.sohu.inputmethod.sogou.xiaomi`

| 类型 | 完整组件类名 |
| --- | --- |
| RECEIVER | `com.sogou.inputmethod.oem.xiaomi.miadvertise.MiAdNetWorkReceiver` |

### 抖音 — `com.ss.android.ugc.aweme`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.facebook.ads.AudienceNetworkActivity` |
| ACTIVITY | `com.bytedance.ies.ugc.aweme.commercialize.splash.show.SplashAdActivity` |

### 抖省省 — `com.ss.android.ugc.lifeservices`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.facebook.ads.AudienceNetworkActivity` |
| ACTIVITY | `com.bytedance.ies.ugc.aweme.commercialize.splash.show.SplashAdActivity` |

### 闲鱼 — `com.taobao.idlefish`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.beizi.ad.v2.activity.BeiZiNewInterstitialActivity` |
| ACTIVITY | `com.qq.e.ads.ADActivity` |
| ACTIVITY | `com.qq.e.ads.PortraitADActivity` |
| ACTIVITY | `com.qq.e.ads.LandscapeADActivity` |
| ACTIVITY | `com.taobao.fleamarket.splashad.SplashAdActivity` |
| SERVICE | `com.qq.e.comm.DownloadService` |
| SERVICE | `com.beizi.ad.DownloadService` |
| SERVICE | `com.bytedance.sdk.openadsdk.downloadnew.ApiDownloadHandlerService` |

### QQ邮箱 — `com.tencent.androidqqmail`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.qq.e.tg.ADActivity` |
| ACTIVITY | `com.qq.e.tg.ADActivitySingleTop` |
| ACTIVITY | `com.qq.e.tg.WebAdActivity` |
| ACTIVITY | `com.qq.e.tg.PortraitADActivity` |
| ACTIVITY | `com.qq.e.tg.LandscapeADActivity` |
| ACTIVITY | `com.qq.e.tg.TransPortraitADActivity` |
| SERVICE | `com.qq.e.downloader.DownloadService` |
| SERVICE | `com.qq.e.comm.DownloadService` |

### 微信 — `com.tencent.mm`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.tencent.mm.plugin.sns.ad.landingpage.SnsAdNativeLandingPagesMMUI` |
| ACTIVITY | `com.tencent.mm.plugin.sns.ad.landingpage.ui.activity.AdHalfScreenPageUI` |
| ACTIVITY | `com.tencent.mm.plugin.sns.ad.landingpage.ui.activity.AdGeneralPageUI` |

### 腾讯会议 — `com.tencent.wemeet.app`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.tencent.wemeet.module.advertisements.activity.AdvertisementActivity` |

### 交管12123 — `com.tmri.app.main`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.tmri.app.ui.activity.main.AdvertActivity` |

### 小红书 — `com.xingin.xhs`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.xingin.advert.intersitial.ui.InterstitialAdsActivity` |
| ACTIVITY | `com.xingin.advert.intersitial.ui2.view.SplashAdActivity` |
| SERVICE | `com.xingin.advert.download.libary.service.DownloadForegroundService` |
| RECEIVER | `com.xingin.advert.download.libary.service.DownloadNotificationActionReceiver` |

### TikTok — `com.zhiliaoapp.musically`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.facebook.ads.AudienceNetworkActivity` |
| ACTIVITY | `com.bytedance.ies.ugc.aweme.commercialize.splash.show.NormalSplashAdActivity` |

### 志愿汇 — `com.zzw.october`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.qq.e.ads.ADActivity` |
| ACTIVITY | `com.qq.e.ads.PortraitADActivity` |
| ACTIVITY | `com.qq.e.ads.LandscapeADActivity` |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoActivity` |
| ACTIVITY | `com.bytedance.sdk.openadsdk.core.component.reward.activity.TTFullScreenVideoLandscapeActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.AdWebViewActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.KsFullScreenVideoActivity` |
| ACTIVITY | `com.kwad.sdk.api.proxy.app.KsFullScreenLandScapeVideoActivity` |
| ACTIVITY | `com.beizi.ad.v2.activity.BeiZiNewInterstitialActivity` |
| SERVICE | `com.qq.e.comm.DownloadService` |
| SERVICE | `com.beizi.ad.DownloadService` |

### 哔哩哔哩 — `tv.danmaku.bili`

| 类型 | 完整组件类名 |
| --- | --- |
| ACTIVITY | `com.bilibili.ad.core.click.AdCallUpActivity` |
| ACTIVITY | `com.bilibili.ad.core.click.AdCallUpActivityWeb` |
| ACTIVITY | `com.bilibili.ad.core.click.AdCallUpActivity2` |
| ACTIVITY | `com.bilibili.ad.web.AdWebActivity` |
| SERVICE | `com.bilibili.ad.core.apkdownload.ADDownloadService` |
| SERVICE | `com.bilibili.ad.web.preload.AdWebViewPreloadService4MainProcess` |

