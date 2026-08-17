# VS010 LAN Ping 修复设计文档

**日期**：2026-08-17  
**适用固件**：OpenWrt SNAPSHOT r0-20d94d5，IPQ5018 + QCA8337，unicom,vs010  
**根因来源**：`VS010_LAN_PING_FIX.md`  
**状态**：待实现

---

## 1. 问题描述

PC 网线直连路由器 LAN 口，`ping 192.168.1.1` 不通。表现为：

- `br-lan` 持有 `192.168.1.1/24` 地址，但 operstate = DOWN，无 carrier
- 所有 LAN 口（lan1/lan2/lan3）`NO-CARRIER`
- `brctl addif br-lan lan1` 报 `Invalid argument`（DSA 口不能 brctl 硬塞）
- ARP `who-has 192.168.1.1` 广播发出，0 个 reply

**双根因叠加：**

1. **语法错误**：`/etc/board.d/02_network` 中 `unicom,vs010|` 缺续行符 `\`，导致 board 网络初始化脚本在启动时崩溃，QCA8337 交换芯片从未被 netifd 初始化。
2. **conduit 映射缺失**：VS010 的 LAN/WAN 对应关系（eth1->LAN，eth0->WAN）未在 `02_network` 中声明，netifd 无法生成正确的 `/etc/config/network`，LAN/WAN 布局反转。

---

## 2. 硬件拓扑（VS010 实机）

| 物理接口 | MAC 地址 | 对应芯片 | 正确功能 |
|---------|---------|---------|---------|
| `eth0` | D8:3D:CC:94:61:8B | IPQ5018 内置 GE PHY | **WAN** |
| `eth1` | D8:3D:CC:94:61:8A | QCA8337 交换芯片上行 | LAN 上行 conduit |
| `lan1/lan2/lan3` | — | QCA8337 Phy2/1/0（DSA） | **LAN 口** |
| `wan`（丝印） | — | QCA8337 Phy3（DSA） | 板面丝印 WAN，实际是交换口 |

> 注意：板面丝印"WAN"物理上属于 QCA8337 交换芯片，不是 eth0。真正的 WAN 上行是 eth0（内置 PHY）。

---

## 3. 修复方案

### 方案选择

| 方案 | 说明 | 结论 |
|-----|-----|-----|
| A：只加 `\` 续行符 | 修语法但 conduit 仍缺，布局仍会出错 | 不完整 |
| B：VS010 单独 case 块 + conduit 映射 | 彻底修两个根因，与同芯片 cmcc,pz-l8 对齐 | 采用 |
| C：只改 uci-defaults overlay | 不改源码，每次刷机需手动修 | 不治本 |

**采用方案 B**，同时新增 uci-defaults 作为二重保险。

---

## 4. 具体改动

### 4.1 修改 `02_network`

**文件**：`target/linux/qualcommax/ipq50xx/base-files/etc/board.d/02_network`

**改动**：将 `unicom,vs010` 从共用分支中移出，单独成一个 case 块，并添加完整 conduit 映射。

```diff
-	unicom,vs010|\
-	cmcc,mr3000d-ci|\
+	cmcc,mr3000d-ci|\
 	linksys,mx2000|\
 	linksys,mx5500|\
 	linksys,spnmx56|\
 	xiaomi,ax6000)
 		ucidef_set_interfaces_lan_wan "lan1 lan2 lan3" "wan"
 		;;
+	unicom,vs010)
+		ucidef_set_interfaces_lan_wan "lan1 lan2 lan3" "wan"
+		ucidef_set_network_device_conduit "lan1" "eth1"
+		ucidef_set_network_device_conduit "lan2" "eth1"
+		ucidef_set_network_device_conduit "lan3" "eth1"
+		ucidef_set_network_device_conduit "wan"  "eth0"
+		;;
```

**效果**：
- 语法崩溃消除，QCA8337 正常初始化
- netifd 知道 lan1/2/3 的上行 conduit 是 eth1，wan 的 conduit 是 eth0
- 生成正确的 `/etc/config/network`

### 4.2 新增 `99-vs010-network` uci-defaults

**文件**：`files/etc/uci-defaults/99-vs010-network`（新建）

**内容**：

```sh
#!/bin/sh
# VS010 network layout fix:
#   LAN = eth1 (QCA8337, lan1/lan2/lan3)
#   WAN = eth0 (IPQ5018 built-in GE PHY)
# Safety net in case board.d/02_network output is stale in overlay.
uci set network.wan.device="eth0"
uci -q delete network.@device[0].ports
uci add_list network.@device[0].ports="lan1"
uci add_list network.@device[0].ports="lan2"
uci add_list network.@device[0].ports="lan3"
uci commit network
```

**效果**：首次开机执行一次，直接写 overlay，保证 `/etc/config/network` 中 LAN/WAN 布局正确，即使 board 层有遗留脏数据也能覆盖。

---

## 5. 不涉及的变更

- 不修改内核驱动：QCA8337 DSA 驱动正常，问题纯粹在用户态配置脚本
- 不修改其他设备分支：仅隔离 VS010，不影响 `cmcc,mr3000d-ci` 等同组设备
- 不改 `br-lan` 名称或 IP：维持 `192.168.1.1/24` 不变

---

## 6. 验证标准

改完重新编译、刷机后，路由器侧执行：

```sh
ls /sys/class/net/br-lan/brif/       # 期望：lan1 lan2 lan3
cat /sys/class/net/br-lan/operstate  # 期望：up
cat /sys/class/net/lan1/carrier      # 期望：1（PC 在线时）
ip addr show br-lan | grep inet      # 期望：192.168.1.1/24
```

PC 侧：

```sh
ping 192.168.1.1   # 期望：0% loss，avg ~1ms
```

---

## 7. 已知陷阱（来自诊断记录）

1. **DSA 口不能 `brctl addif` 硬加**：QCA8337 的 DSA 口必须经 netifd bridge 配置，强行 `brctl` 会报 `Invalid argument`。
2. **中途误操作可能导致 eth0/Phy4 链路崩**：大量 ip link/brctl/restart 操作可能把 CPU 到 QCA8337 的内部上行搞崩，重启路由器即自愈，不要去改内核。

---

## 8. 参考

- 诊断记录：`VS010_LAN_PING_FIX.md`（`/home/san/vs010/vs010-openwrt/`）
- 同芯片参考实现：`cmcc,pz-l8` 分支（`02_network` 第 33-39 行）
