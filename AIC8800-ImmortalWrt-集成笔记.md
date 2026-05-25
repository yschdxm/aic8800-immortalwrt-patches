# 把 AIC8800 USB WiFi 网卡接入 ImmortalWrt：一段跌宕起伏的调试之旅

我有一块 AIC8800 芯片的 USB WiFi 适配器（ax900-wifi-adapter-linux-driver-V1.0.1.4），想把它插在跑了 ImmortalWrt 的 MT7981 路由器上，变成一个可用的无线接口。原以为只要有驱动、能加载模块就完事了，结果前前后后折腾了好几天，踩了七八个坑。这篇文章按时间顺序还原整个过程，包括每一次失败的根因、推理路径和最终的修复代码。

---

## 起点：驱动能加载，但 LuCI 看不到

AIC8800 的驱动源码结构是这样的：它包含一个固件加载模块 `aic_load_fw.ko` 和一个主驱动 `aic8800_fdrv.ko`，后者是标准的 cfg80211 全 MAC（fullmac）驱动。我仿照 OpenWrt 其他 USB 网卡的写法，写好了 Makefile 和 hotplug 脚本，把固件放到 `/lib/firmware/` 下，编译进固件刷入路由器。

驱动是加载成功了，`iwinfo` 也能看到 `phy0` 和 `wlan0`，但 LuCI 的无线页面里就是找不到它。打开 `logread` 一看，netifd 反复打印 `Wireless device 'radio0' setup failed, retry=3,2,1,0`，四次重试后放弃。

问题的入口在 `/lib/netifd/wireless/mac80211.sh`。这是 OpenWrt 为所有 mac80211 类型无线设备编写的统一管理脚本，netifd 通过它来发现 phy、配置 hostapd、启动 BSS。我要做的就是让这个脚本"认识"AIC8800。

## 第一道坎：BusyBox 的算术语法错误

打开 `logread` 细看，每次 radio0 尝试启动都有一行：`eval: line 123: arithmetic syntax error`。定位到 mac80211.sh 第 122-123 行：

```sh
[ "$(($4))" -gt 0 ] || continue
[ "$(((0x$2) & $3))" -gt 0 ] || {
```

这里 `$4` 来自 AIC8800 的 HE 能力字符串。用 `iw phy phy0 info` 可以看到，AIC8800 上报的 HE PHY Capabilities 是一个 22 字符的十六进制数，比如 `0x06e02b580dc0cf00023000`。当这个值被传入 `$(($4))` 时，BusyBox 的 ash 直接崩溃——它期望纯数字，但收到的可能是空字符串或者超出整数范围的值。

OpenWrt 用 BusyBox 而不是 Bash，BusyBox 的算术求值非常脆弱。修复方式是用 `eval` 包一层，加上 `2>/dev/null` 吃掉错误：

```sh
eval "test \"\$4\" -gt 0" 2>/dev/null || continue
eval "test \"\$((0x\$2 & \$3))\" -gt 0" 2>/dev/null || {
```

同样的模式在 `mac80211_add_capabilities` 函数里也存在（第 106-107 行），也一起修了。

## 第二道坎：grep 的 `\t` 不工作

修完算术错误后，脚本继续往前跑了一段，但在另一个地方又悄悄失败了。这次没有报错，只是 `mac80211_prepare_vif` 函数从未被调用到。一路加 `echo` 追踪，发现脚本在调用 `mac80211_iw_interface_add` 时，要检查已有接口的类型：

```sh
case "$(iw dev $ifname info | grep "^\ttype" | cut -d' ' -f2- 2>/dev/null)" in
```

BusyBox 的 grep 不支持 `\t` 转义。这行匹配永远失败，导致后续逻辑跳过。修复：用 sed 替代。

```sh
case "$(iw dev $ifname info | sed -n 's/^[[:space:]]*type //p' 2>/dev/null)" in
```

## 第三道坎：HT 能力解析被 HE 行污染

继续向前跑，下一个崩溃点在 `mac80211_hostapd_setup_base` 函数的 HT 能力解析：

```sh
ht_cap_mask=0
for cap in $(iw phy "$phy" info | grep 'Capabilities:' | cut -d: -f2); do
    ht_cap_mask="$(($ht_cap_mask | $cap))"
done
```

这行 `grep 'Capabilities:'` 的本意是匹配 HT 能力行（如 `Capabilities: 0x963`），但 AIC8800 的 `iw phy` 输出里还有一行 `HE PHY Capabilities: (0x06e02b580dc0cf00023000):`。grep 把 HE 行也抓进来了，`cut -d: -f2` 取出的是 `(0x06e02b580dc0cf00023000)`，带括号。括号进入 `$(())`，BusyBox 再次炸掉。

修复：让 grep 更精确，只匹配 `Capabilities: 0x` 这种格式（HT 行以 `0x` 开头，HE 行以 `(0x` 开头）：

```sh
for cap in $(iw phy "$phy" info | grep 'Capabilities: 0x' | cut -d: -f2); do
```

## 第四道坎：BusyBox sed 遇到空格就罢工

HT 问题修完后，又卡在 HE 能力解析。mac80211.sh 用 sed 从 `iw phy` 输出中提取 HE 能力值：

```sh
he_phy_cap=$(iw phy "$phy" info | sed -n '/HE Iftypes: AP/,$p' | awk ...)
```

这行在 GNU sed 上完全正常，但在 BusyBox sed 上报 `sed: no address after comma`。原因竟然是正则表达式 `/HE Iftypes: AP/` 里面的**空格**！BusyBox sed 解析地址范围 `/regex/,$p` 时，如果 regex 里有空格且没有用 `\` 转义，就会把空格之后的内容当成地址的一部分，导致解析失败。

修复很简单——去掉 "AP"，只留 `/HE Iftypes:/`，因为 `iw phy` 输出里只有一行 HE Iftypes：

```sh
he_phy_cap=$(iw phy "$phy" info | sed -n '/HE Iftypes:/,$p' | awk ...)
```

## 第五道坎：hostapd 根本不认识 802.11ax

上面四个 BusyBox 兼容性问题全修完后，脚本终于完整跑通了。但 radio0 的 `hostapd` 启动失败，日志里出现了 30 条：

```
hostapd: Line XX: unknown configuration item 'ieee80211ax'
hostapd: Line XX: unknown configuration item 'he_bss_color'
hostapd: Line XX: unknown configuration item 'he_mu_edca_ac_be_aifsn'
...
hostapd: 30 errors found in configuration file
```

这个路由器的 hostapd 是在没有 `CONFIG_DRIVER_11AX_SUPPORT=y` 的情况下编译的，所有 IEEE 802.11ax（Wi-Fi 6）的配置项全都不认识。而 mac80211.sh 只要检测到网卡支持 HE 能力，就会往 hostapd 配置文件里写入 `ieee80211ax=1` 和一系列 `he_*` 选项。

`.config` 里是这样的：

```
# CONFIG_DRIVER_11AX_SUPPORT is not set
```

这个配置项控制 hostapd Makefile 是否定义 `HOSTAPD_IEEE80211AX:=y`，进而传递 `CONFIG_IEEE80211AX=y` 给编译。

修复分两步。第一步，在 `.config` 中启用：

```
CONFIG_DRIVER_11AX_SUPPORT=y
```

第二步，在 mac80211.sh 中添加 hostapd HE 能力检测，防止刷到旧版 hostapd 上再次爆炸：

```sh
if [ "$enable_ax" != "0" ]; then
    echo -e "driver=nl80211\ninterface=__he_test\nieee80211ax=1" > /tmp/.he_check.conf
    /usr/sbin/hostapd -t /tmp/.he_check.conf 2>&1 | grep -q "unknown configuration item 'ieee80211ax'" && enable_ax=0
    rm -f /tmp/.he_check.conf
fi
```

原理：创建一个临时配置文件写入 `ieee80211ax=1`，用 `hostapd -t`（测试模式）检查 hostapd 是否认识这个选项。如果输出包含 "unknown configuration item"，说明不支持 HE，就关闭。如果支持，就正常往下走。

## 第六道坎：手动启动 hostapd 帮倒忙

之前在调试过程中我们加了一行 `hostapd -s -g /var/run/hostapd/global -B /dev/null` 在 `ubus wait_for hostapd` 前面。本意是当前系统没有全局 hostapd 守护进程时手动启动一个。但后来发现 `/etc/init.d/wpad` 已经在启动时运行了 `hostapd -s -g /var/run/hostapd/global`，不需要我们手动干预。更糟的是，我们传了 `/dev/null` 作为配置文件，它被 hostapd 当成了实际的配置文件去解析，反而导致出错。

这一行直接删掉就好，wpad 已经处理好了。

## 第七道坎：发射功率显示为 0

网卡终于能用了，但 LuCI 里 Tx-Power 显示 0 dBm。查驱动代码 `rwnx_main.c`，发现 `.get_tx_power` 回调被注释掉了：

```c
.set_tx_power = rwnx_cfg80211_set_tx_power,
//    .get_tx_power = rwnx_cfg80211_get_tx_power,
```

而且 `rwnx_cfg80211_get_tx_power` 函数根本没有实现。内核无法从驱动获取发射功率，iwinfo 只能显示 0。

修复：在 `rwnx_defs.h` 的 `struct rwnx_hw` 中加一个 `s8 tx_power` 字段，在 `set_tx_power` 中存储设定值，然后实现 `get_tx_power` 函数：

```c
static int rwnx_cfg80211_get_tx_power(struct wiphy *wiphy,
                                       struct wireless_dev *wdev,
                                       int *dbm)
{
    struct rwnx_hw *rwnx_hw = wiphy_priv(wiphy);

    if (rwnx_hw->tx_power == 0x7f)
        *dbm = 20; /* auto mode, report 20 dBm */
    else
        *dbm = rwnx_hw->tx_power;

    return 0;
}
```

最后把之前注释的回调取消注释：

```c
.get_tx_power = rwnx_cfg80211_get_tx_power,
```

初始化默认值为 20 dBm。这样 iwinfo 就能正确读取发射功率了。

## 第八道坎：信道 14 导致配置失败

用户在 LuCI 里切换信道测试时，选到了信道 14（2.484 GHz），保存后整个 radio 挂掉。logread 显示 hostapd 报 `Failed to prepare rates table`。

信道 14 只在日本合法，且仅支持 802.11b（DSSS），不允许 OFDM 速率。但 mac80211.sh 为 `hw_mode=g` 生成的 `supported_rates` 包含 OFDM 速率（6、9、12、18、24、36、48、54 Mbps）。当 hostapd 试图在信道 14 上使用这些 OFDM 速率时，直接失败。

追到驱动源码，发现 `rwnx_2ghz_channels` 数组把 2484 MHz（信道 14）列在了"常规信道"区，而非后面的"仅 PHY 测量的额外信道"区。把它删掉，让驱动只上报 1-13 信道，LuCI 里就不会再出现信道 14 这个选项。

## 第九道坎：客户端信号强度始终为 0

客户端连上后，BitRate 正常显示（HE-MCS 9, 114.7 Mbit/s），但 Signal 始终是 0 dBm。`iw dev wlan0 station dump` 也显示 `signal: 0 dBm`。

追到驱动的 `rwnx_fill_station_info` 函数，它通过 `rwnx_send_get_sta_info_req` 向芯片固件请求站点信息，然后读 `cfm.rssi`。加了错误日志后发现固件请求是成功的，只是芯片固件返回的 RSSI 字段永远为 0。这是 AIC8800 闭源固件的问题，无法从驱动侧修复固件响应。

但我注意到驱动在 RX 路径中，每收到一个数据包都会把硬件测量的 RSSI 存入 `stats->last_rx.rx_vect1.rssi1`。这个值是实打实的硬件测量结果，不依赖固件的额外请求。而且 radiotap 代码直接用它作为 `IEEE80211_RADIOTAP_DBM_ANTSIGNAL`，说明它本身就是 dBm 单位的有效值。

所以修复很简单——不用 `cfm.rssi`，改用 `rx_vect1->rssi1`。同时我发现 `sinfo->filled` 标志位里竟然缺少 `NL80211_STA_INFO_SIGNAL`，这意味着即使 `sinfo->signal` 有值，内核也会因为标志位缺失而忽略。一并修：

```c
sinfo->filled |= (BIT(NL80211_STA_INFO_TX_BITRATE) | BIT(NL80211_STA_INFO_TX_FAILED) |
                  BIT(NL80211_STA_INFO_SIGNAL));
// ...
sinfo->signal = rx_vect1->rssi1;  // 原来: sinfo->signal = (s8)cfm.rssi;
```

## 终态

所有修复完成后，AIC8800 网卡在 ImmortalWrt 上完全正常工作。radio0 被识别为 mac80211 类型，LuCI 可以像管理内置网卡一样管理它：切换信道、调整频宽、修改发射功率、更换 SSID 和加密方式，全都正常。客户端连接后，信号强度、速率都能正确显示。

这个过程反复了将近十轮编译-刷机-测试，每次都在看起来"马上就好了"的地方翻出新的坑。回过头看，这些坑基本上可以归为三类：BusyBox 和 GNU 工具链的微妙差异（`\t`、`$(())`、sed 的空格处理）、OpenWrt 构建配置的遗漏（HE 支持未启用）、以及 AIC8800 驱动本身的不完善（`get_tx_power` 缺失、RSSI 取错数据源）。修完后系统运行稳定，所有 LuCI 功能正常可用。
