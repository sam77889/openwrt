# VS010 LAN Ping 修复实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修复 VS010 路由器 LAN 口无法 ping 通 192.168.1.1 的问题，根治双根因（board 脚本语法错误 + conduit 映射缺失）。

**Architecture:** 修改 board 网络初始化脚本，将 VS010 单独拆出 case 块并声明正确的 eth conduit 映射；同时新增 uci-defaults 脚本作为首次开机的二重保险，确保 `/etc/config/network` 布局正确。

**Tech Stack:** OpenWrt shell（ash/busybox），uci，netifd，DSA（QCA8337），IPQ5018 aarch64

## Global Constraints

- 目标平台：`target/linux/qualcommax/ipq50xx`，设备 `unicom,vs010`
- LAN 接口必须为 `lan1 lan2 lan3`（走 eth1 / QCA8337），WAN 接口必须走 `eth0`（IPQ5018 内置 GE PHY）
- 不得修改其他设备（`cmcc,mr3000d-ci`、`linksys,mx2000` 等）的分支逻辑
- uci-defaults 脚本命名遵循已有惯例：`99-vs010-<subsystem>`，无执行权限也可（OpenWrt 会 source 而非 exec）
- 修改后需能在不重新编译的情况下手工验证（串口 TTL 115200 8N1）

---

### Task 1：修复 `02_network`——语法错误 + conduit 映射

**Files:**
- Modify: `target/linux/qualcommax/ipq50xx/base-files/etc/board.d/02_network:20-26`

**Interfaces:**
- Consumes: 无（第一个任务）
- Produces: 开机时 `ipq50xx_setup_interfaces unicom,vs010` 正确执行，写入 `/etc/board.json`，netifd 据此生成含正确 conduit 映射的 `/etc/config/network`

---

- [ ] **Step 1: 查看当前文件，确认第 20 行内容**

  ```bash
  sed -n '17,30p' target/linux/qualcommax/ipq50xx/base-files/etc/board.d/02_network
  ```

  期望输出（有 bug 的当前状态）：
  ```
          unicom,vs010|
          cmcc,mr3000d-ci|\
          linksys,mx2000|\
          linksys,mx5500|\
          linksys,spnmx56|\
          xiaomi,ax6000)
                  ucidef_set_interfaces_lan_wan "lan1 lan2 lan3" "wan"
                  ;;
  ```

- [ ] **Step 2: 将 `unicom,vs010` 从共用分支中移出，单独添加 case 块**

  将文件第 20–26 行（`unicom,vs010|` 到 `;;`）替换为以下内容：

  ```sh
  	cmcc,mr3000d-ci|\
  	linksys,mx2000|\
  	linksys,mx5500|\
  	linksys,spnmx56|\
  	xiaomi,ax6000)
  		ucidef_set_interfaces_lan_wan "lan1 lan2 lan3" "wan"
  		;;
  	unicom,vs010)
  		ucidef_set_interfaces_lan_wan "lan1 lan2 lan3" "wan"
  		ucidef_set_network_device_conduit "lan1" "eth1"
  		ucidef_set_network_device_conduit "lan2" "eth1"
  		ucidef_set_network_device_conduit "lan3" "eth1"
  		ucidef_set_network_device_conduit "wan"  "eth0"
  		;;
  ```

  > 注意缩进为 TAB，不是空格，与文件其他行保持一致。

- [ ] **Step 3: 验证语法正确（本地 shell 检查）**

  ```bash
  bash -n target/linux/qualcommax/ipq50xx/base-files/etc/board.d/02_network
  ```

  期望：无任何输出，退出码 0。若有报错，检查 TAB 缩进和 `\` 续行符。

- [ ] **Step 4: 确认 VS010 块内容正确**

  ```bash
  grep -A 8 "unicom,vs010" target/linux/qualcommax/ipq50xx/base-files/etc/board.d/02_network
  ```

  期望输出：
  ```
          unicom,vs010)
                  ucidef_set_interfaces_lan_wan "lan1 lan2 lan3" "wan"
                  ucidef_set_network_device_conduit "lan1" "eth1"
                  ucidef_set_network_device_conduit "lan2" "eth1"
                  ucidef_set_network_device_conduit "lan3" "eth1"
                  ucidef_set_network_device_conduit "wan"  "eth0"
                  ;;
  ```

- [ ] **Step 5: 确认其他设备分支未受影响**

  ```bash
  grep -n "cmcc,mr3000d-ci\|linksys,mx2000\|xiaomi,ax6000" target/linux/qualcommax/ipq50xx/base-files/etc/board.d/02_network
  ```

  期望：这三个仍在同一 case 块中，`cmcc,mr3000d-ci|\` 后有 `\`。

- [ ] **Step 6: Commit**

  ```bash
  git add target/linux/qualcommax/ipq50xx/base-files/etc/board.d/02_network
  git commit -m "ipq50xx: vs010: fix 02_network syntax error and add conduit mapping

  unicom,vs010 was missing the backslash line continuation in the case
  statement, causing a syntax error on boot that prevented QCA8337 from
  being initialized by netifd.

  Move vs010 into its own case block and add explicit conduit mappings:
  - lan1/lan2/lan3 via eth1 (QCA8337 uplink)
  - wan via eth0 (IPQ5018 built-in GE PHY)

  This matches the pattern used by cmcc,pz-l8 which has the same
  hardware topology.

  Fixes: boot-time 'syntax error: unexpected word (expecting \")\")'"
  ```

---

### Task 2：新增 `99-vs010-network` uci-defaults（二重保险）

**Files:**
- Create: `files/etc/uci-defaults/99-vs010-network`

**Interfaces:**
- Consumes: Task 1 完成后的硬件拓扑理解（eth0=WAN，eth1 conduit→lan1/2/3）
- Produces: 固件首次开机时 `/etc/config/network` overlay 被强制写为正确布局，无论 board 层历史状态如何

---

- [ ] **Step 1: 查看现有 uci-defaults 文件，确认格式惯例**

  ```bash
  ls files/etc/uci-defaults/
  cat files/etc/uci-defaults/99-vs010-system
  ```

  期望：看到以 `#!/bin/sh` 开头、用 `uci set/commit` 的简洁脚本，无 `exit 0` 也可。

- [ ] **Step 2: 创建 `99-vs010-network` 文件**

  新建文件 `files/etc/uci-defaults/99-vs010-network`，内容如下：

  ```sh
  #!/bin/sh
  # VS010 network layout:
  #   LAN = eth1 (QCA8337 DSA, lan1/lan2/lan3)
  #   WAN = eth0 (IPQ5018 built-in GE PHY)
  # Explicitly set in uci overlay as safety net for stale board.json state.
  uci set network.wan.device="eth0"
  uci -q delete network.@device[0].ports
  uci add_list network.@device[0].ports="lan1"
  uci add_list network.@device[0].ports="lan2"
  uci add_list network.@device[0].ports="lan3"
  uci commit network
  ```

- [ ] **Step 3: 验证文件内容**

  ```bash
  cat files/etc/uci-defaults/99-vs010-network
  ```

  确认：6 条 uci 命令完整，无拼写错误，`network.wan.device` 为 `eth0`，ports 列表为 `lan1 lan2 lan3`。

- [ ] **Step 4: 手工语法检查**

  ```bash
  bash -n files/etc/uci-defaults/99-vs010-network
  ```

  期望：无输出，退出码 0。

- [ ] **Step 5: Commit**

  ```bash
  git add files/etc/uci-defaults/99-vs010-network
  git commit -m "vs010: add uci-defaults to enforce correct LAN/WAN layout on first boot

  Add 99-vs010-network as a safety net: explicitly set network.wan.device
  to eth0 and br-lan ports to lan1/lan2/lan3 via uci overlay on first boot.

  This ensures correct layout even if board.d/02_network output is stale
  in the overlay from a previous (buggy) boot.

  VS010 topology:
  - eth0 = IPQ5018 internal GE PHY = WAN
  - eth1 = QCA8337 uplink conduit
  - lan1/lan2/lan3 = QCA8337 Phy2/1/0 (DSA) = LAN ports"
  ```

---

### Task 3：编译验证与刷机测试

**Files:**
- 无新文件，验证 Task 1 & 2 产物在固件中正确打包

**Interfaces:**
- Consumes: Task 1 修改的 `02_network`，Task 2 新建的 `99-vs010-network`

---

- [ ] **Step 1: 编译 base-files 包**

  ```bash
  make package/base-files/compile V=s -j$(nproc) 2>&1 | tail -20
  ```

  期望：`build_dir/target-aarch64_cortex-a53_musl/linux-qualcommax_ipq50xx/base-files/` 下生成新的 `.ipk`，无编译错误。

- [ ] **Step 2: 确认修改进入 build_dir**

  ```bash
  grep -A 8 "unicom,vs010" build_dir/target-aarch64_cortex-a53_musl/linux-qualcommax_ipq50xx/base-files/.pkgdir/base-files/etc/board.d/02_network
  ```

  期望：看到 VS010 独立 case 块含 4 条 `ucidef_set_network_device_conduit`。

- [ ] **Step 3: 编译完整固件（可选，若仅需增量验证可跳过）**

  ```bash
  make -j$(nproc) V=s 2>&1 | tail -50
  ```

  期望：`bin/targets/qualcommax/ipq50xx/` 下生成 `openwrt-qualcommax-ipq50xx-unicom_vs010-squashfs-sysupgrade.bin`。

- [ ] **Step 4: 刷机后串口验证（路由器侧）**

  通过 TTL 串口（115200 8N1）登录路由器，执行：

  ```sh
  # 确认 board 脚本不再报语法错误（dmesg 或 logread）
  logread | grep -i "syntax error\|board.d"

  # 确认 br-lan 有三个 LAN 口
  ls /sys/class/net/br-lan/brif/
  # 期望：lan1  lan2  lan3

  # 确认 br-lan 状态 up
  cat /sys/class/net/br-lan/operstate
  # 期望：up

  # 确认 LAN 口有 carrier（需 PC 网线在线）
  cat /sys/class/net/lan1/carrier
  # 期望：1

  # 确认 IP 地址正确
  ip addr show br-lan | grep inet
  # 期望：inet 192.168.1.1/24
  ```

- [ ] **Step 5: PC 侧 ping 验证**

  PC 设置静态 IP 192.168.1.100/24 后执行：

  ```bash
  ping -c 4 192.168.1.1
  ```

  期望：
  ```
  4 packets transmitted, 4 received, 0% packet loss, time ...ms
  rtt min/avg/max = .../0.9/... ms
  ```

- [ ] **Step 6: 确认 WAN 口布局正确**

  路由器侧：

  ```sh
  uci show network.wan.device
  # 期望：network.wan.device='eth0'

  uci show network.@device[0].ports
  # 期望：network.@device[0].ports='lan1' 'lan2' 'lan3'
  ```

---

## 自检（Self-Review）

**Spec coverage：**
- ✅ 语法错误（unicom,vs010 缺 `\`） → Task 1
- ✅ conduit 映射缺失（eth1→lan，eth0→wan） → Task 1
- ✅ 二重保险 uci-defaults → Task 2
- ✅ 验证标准（brif、operstate、carrier、ping） → Task 3

**Placeholder 扫描：** 无 TBD/TODO，所有步骤含具体命令和期望输出。

**类型一致性：** 接口名（lan1/lan2/lan3/wan/eth0/eth1）在三个任务中完全一致。
