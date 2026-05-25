# AIC8800 USB WiFi 适配器 ImmortalWrt 集成补丁

将 AIC8800 (ax900-wifi-adapter-linux-driver-V1.0.1.4) USB WiFi 适配器完整集成到 ImmortalWrt (MediaTek MT7981, kernel 5.4)。

## 修改文件

### 1. mac80211.sh — 无线管理脚本 (7 处修复)

**文件路径**: `package/kernel/mac80211/files/lib/netifd/wireless/mac80211.sh`

| 修复 | 说明 |
|------|------|
| BusyBox 算术安全 | `$(($4))` → `eval "test ..."` 避免非数值导致语法错误 |
| HT 能力 grep 精确匹配 | `grep 'Capabilities:'` → `grep 'Capabilities: 0x'` 排除 HE 行 |
| HE sed 移除空格 | `/HE Iftypes: AP/` → `/HE Iftypes:/` 修复 BusyBox sed 解析 |
| HE hostapd 能力检测 | 用 `hostapd -t` 测试 `ieee80211ax` 是否被支持 |
| grep `\t` → sed | BusyBox grep 不支持 `\t` 转义 |
| 移除手动 hostapd 启动 | wpad init 已处理，额外启动导致冲突 |

详见 `mac80211.sh.patch`。

### 2. rwnx_main.c — AIC8800 驱动主文件 (4 处修复)

**文件路径**: `package/kernel/aic8800/drivers/aic8800/aic8800_fdrv/rwnx_main.c`

| 修复 | 说明 |
|------|------|
| `get_tx_power` 实现 | 新增回调函数，初始化为 20 dBm，取消注释 `.get_tx_power` |
| 移除信道 14 | 将 CHAN(2484) 从常规信道列表移除 |
| 信号来源修复 | `sinfo->signal` 改用 `rx_vect1->rssi1`（RX 路径实测值）而非 `cfm.rssi`（固件返回恒为 0） |
| 信号标志位补充 | `sinfo->filled` 增加 `NL80211_STA_INFO_SIGNAL` |

### 3. rwnx_defs.h — 驱动数据结构

**文件路径**: `package/kernel/aic8800/drivers/aic8800/aic8800_fdrv/rwnx_defs.h`

在 `struct rwnx_hw` 中新增 `s8 tx_power` 字段，用于存储和读取发射功率。

### 4. .config — 构建配置

```
CONFIG_DRIVER_11AX_SUPPORT=y
CONFIG_WPA_MBO_SUPPORT=y
```

启用 hostapd 的 IEEE 802.11ax (Wi-Fi 6) 编译支持。

## 使用方式

1. 将 `modified/` 下的文件覆盖到 ImmortalWrt 源码树对应位置
2. 在 `.config` 中添加 `CONFIG_DRIVER_11AX_SUPPORT=y`
3. 运行 `make defconfig && make` 编译固件

## 详细过程

参见 `AIC8800-ImmortalWrt-集成笔记.md`。
