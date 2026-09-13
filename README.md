# Linksys WRT1900AC v2 (Cobra) 固件配置说明

> 目标设备：Marvell Armada 385 88F6820（2× Cortex-A9 @1.6GHz）/ 512MB DDR3 / 128MB NAND / 88W8864 双频 WiFi / 5× GbE（mv88e6176，DSA）
> 取向：**稳定 > 性能 > 体积**
> 基准源：Lean's LEDE 系（opkg + firewall3/iptables），与你现在 `jarodvip/linksys` 的 CI 保持一致

---

## 一、交付物

| 文件 | 作用 |
|---|---|
| `.config` | 编译配置本体，精简 diffconfig 格式，直接放源码根目录 |
| `files/etc/sysctl.d/99-wrt1900ac-tuning.conf` | 运行时内核参数调优，整目录拷进源码树的 `files/` 即可编进固件 |
| `README.md` | 本文件：决策依据、体积预算、运行期设置、构建与排错 |

`.config` 是**精简格式**，只写非默认值。`make defconfig` 会自动展开成完整配置，因此它不会因为源版本升级（6.6 → 6.12、gcc 换版本）而失效——这也是我刻意**不写** `CONFIG_LINUX_6_6`、`CONFIG_GCC_VERSION`、`CONFIG_BINUTILS_VERSION` 的原因。

---

## 二、硬件事实基线（决定后面所有取舍）

这部分来自官方构建产物，不是经验估计：

| 项 | 官方值 | 影响 |
|---|---|---|
| Target / Profile | `mvebu/cortexa9` · `DEVICE_linksys_wrt1900ac-v2` | 设备符号名必须精确匹配 |
| 官方 device_packages | `kmod-mwlwifi` `wpad-basic-mbedtls` `mwlwifi-firmware-88w8864` `kmod-dsa-mv88e6xxx` | 这 4 个是开机能联网的**最小必要集** |
| 内核（25.12.5） | 6.12.94 | 你现有 CI 是 6.6，都属 LTS，可不动 |
| **kernel 分区硬上限** | **6144 KiB** | 内核塞太多 built-in 模块会直接拒绝生成镜像 |
| rootfs | ~34 MiB UBI（双分区轮写，每区 40MB） | 体积是真实约束，不是理论值 |
| 官方默认 sysupgrade 体积 | 7.998 MB | 裸机基线 |
| 你现有固件体积 | 9.25 MB | 已含 OpenClash + ruby + smartdns |

来源：[OpenWrt 25.12.5 mvebu/cortexa9 profiles.json](https://downloads.openwrt.org/releases/25.12.5/targets/mvebu/cortexa9/profiles.json)、[config.buildinfo](https://downloads.openwrt.org/releases/25.12.5/targets/mvebu/cortexa9/config.buildinfo)

---

## 三、稳定性决策（这一节是重点）

### 3.1 必须做的事

| 决策 | 理由 |
|---|---|
| **显式禁用 `kmod-mwifiex-sdio`** | 这是老 radio2 驱动。官方 wiki 明确指出它与 mwlwifi 争抢 DFS 信道，导致 5GHz 反复 up/down——**这是这批机器"不稳定"最常见的根因**，而很多人压根不知道自己编进去了 |
| **关闭 `MAC80211_MESH`** | mwlwifi 根本不实现 802.11s。开着只是白白吃掉体积 |
| **装 `luci-app-advanced-reboot`** | WRT AC 系列是**双固件分区轮写**：每次 sysupgrade 写到"另一个"分区并切换启动。没有这个插件，你不知道下次重启落哪个分区，刷机就是开盲盒。官方 wiki 也把它列为推荐项 |
| **保留 `kmod-dsa-mv88e6xxx`** | mv88e6176 交换芯片驱动。mvebu 自 2020 年就彻底移除了 swconfig，旧配置里的 `swconfig` / `kmod-swconfig` 千万别带过来 |
| **保留 `CONFIG_KERNEL_KALLSYMS`** | mwlwifi/mac80211 出 oops 时，没有符号表等于两眼一抹黑 |

### 3.2 一个重要的纠偏：别再手动关 A-MSDU 了

网上大量老教程让你在 `/etc/rc.local` 里写：

```sh
echo "0" >> /sys/kernel/debug/ieee80211/phy0/mwlwifi/tx_amsdu
```

**在 23.05.3 及以后不要再这么做。** mwlwifi 10.4.10（2023-11-20）已经修掉了 88W8864 的高延迟问题，官方 wiki 明确写着"the fix below should no longer be used"。继续关 A-MSDU 只会白丢峰值吞吐。

### 3.3 我删掉的东西（相对你现有配置）

| 移除项 | 原因 |
|---|---|
| `zram-swap` + `kmod-zram` | 512MB DDR3 根本不缺内存。zram 只是让双核 CPU 持续做 lz4 压缩，纯负收益 |
| `kernel DEBUG_INFO` | 只影响构建产物体积与编译时间，不影响 rootfs；要调试就留着 KALLSYMS 足够 |
| `KEXEC` / `CRASH_DUMP` / `PROC_VMCORE` | mvebu 上无实用价值，还占保留内存 |
| `KERNEL_IP_MROUTE` / `PIMSM` / `MPTCP` | 家用用不到，省代码与内存。要看 IPTV 需连同 `igmpproxy` 一起打开（`.config` 第 15 节有注释片段） |
| `GDB` | 省体积，交叉调试用 SDK 更合适 |
| `autosamba` / `ksmbd` | 体积换功能，默认关。需要 USB 共享时取消 `.config` 里的注释 |

### 3.4 我保留但默认"运行期关闭"的东西：flow offloading

这里跟你的既有判断有分歧，我把依据摆出来，你自己定：

- **官方 wiki 的立场**：Linksys WRT AC 系列 CPU **没有** 任何硬件卸载（无 HFO、无 WED），所以**推荐开软件流量分载（SFO）**。
- **你的顾虑的由来**：21.02 时代 DSA 与 SFO 有冲突（hidden VLAN 导致每包在快慢路径之间反复 bounce，吞吐直接腰斩到 ~300Mbps），这在论坛里有大量案例。
- **现状**：该问题在 21.02 后期已修复，你现在的内核是 6.6/6.12，不再是那个环境。

**我的做法是工程折中，而不是二选一**：`kmod-ipt-offload` + `kmod-nf-flow` + `luci-app-turboacc` **都编译进固件**，但**默认不开**。这样你想试的时候在 LuCI 里点一下就能开，出问题点一下就能关，**完全不用重刷固件**。

> 编译期决定"有没有能力"，运行期决定"开不开"。这才是可控的做法。

验证方法（开 SFO 后跑 48 小时）：

```sh
# 看 5GHz 是否反复掉线
logread | grep -iE "mwlwifi|phy0|deauth"
# 看 offload 是否真的命中（cnt 要持续增长）
conntrack -L | head -1; cat /proc/net/nf_conntrack | wc -l
# 吞吐实测（LAN→WAN 而不是跑 WiFi）
iperf3 -c <server> -P 4
```

**FullCone NAT 与 flow offloading 在 fw3 下互斥**，只能取一个。有 NAT1 / 游戏联机需求就选 FullCone、关 offload；追求吞吐就反过来。

---

## 四、性能决策

| 手段 | 做法 | 预期收益 | 代价 |
|---|---|---|---|
| 应用编译档位 | `-Os -pipe` | — | 换 `-O2` 吞吐提升不足 3%，但 rootfs 涨明显，**不划算** |
| 内核编译档位 | `CONFIG_KERNEL_CC_OPTIMIZE_FOR_PERFORMANCE=y`（-O2） | 软转发、conntrack、nftables 全在内核态，收益直接 | 内核增量经 squashfs 压缩后很小 |
| ARM 专项 | `-fno-caller-saves -fno-plt` | 减少寄存器保存与 PLT 间接跳转，转发路径确定受益 | 无 |
| 符号裁剪 | `USE_SSTRIP` | rootfs 小 10%~15% | 无 |
| BBR | `kmod-tcp-bbr` + `net.ipv4.tcp_congestion_control=bbr` | 高延迟/丢包链路上传输提升明显 | 无 |
| conntrack | 65536 条 + 缩短超时 | 多设备/P2P 不再丢新连接 | 约 20MB 内存 |
| 收包队列 | `netdev_max_backlog=2048` | 高 PPS 不再溢出丢包 | 略增延迟 |
| Packet Steering | `network.globals.packet_steering='1'` | 双核分摊软中断 | 无（**必开**） |
| irqbalance | 默认**不开** | 双核上把 mwlwifi 中断挪 CPU1 | 官方 wiki 明确指出双核设备启用后 **WiFi 延迟反而升高** |

### 吞吐预期区间（需你实测校准）

| 场景 | 经验区间 |
|---|---|
| WAN NAT，纯软转发，无 offload | 300 ~ 500 Mbps |
| WAN NAT，开 SFO | 600 ~ 900 Mbps |
| 开 SQM/cake | 250 ~ 350 Mbps（换极低延迟，值不值看你容忍度） |
| 5GHz 80MHz 实测 | 典型 300 ~ 450 Mbps |

> 交换芯片内部的 LAN↔LAN 是线速 1Gbps，不走 CPU，别拿它当基准。

---

## 五、体积预算

| 组成 | 估算 |
|---|---|
| 基础系统 + LuCI + 内核 | ~8.0 MB |
| OpenClash + ruby 全家桶 | **+6 ~ 8 MB（最大头）** |
| smartdns、nlbwmon、ddns、upnp、htop 等 | +1.5 MB |
| 预计 sysupgrade 总量 | **约 9 ~ 10 MB** |
| rootfs 可用 | ~34 MB |

**结论**：空间充裕，但 ruby 是绝对大头。如果你哪天不需要 OpenClash，直接删掉 `.config` 第 13 节，固件立刻瘦一半以上。

需要精确审计时用这两个命令：

```sh
make -j$(nproc) V=s
./scripts/diffconfig.sh > my-diffconfig     # 生成便于版本控制的精简配置
ls -la bin/targets/mvebu/cortexa9/*.bin     # 看实际产物体积
```

---

## 六、运行期配置（编好固件后执行）

```sh
# 1) Packet Steering —— 双核分摊，必开
uci set network.globals.packet_steering='1'
uci commit network

# 2) 无线（按官方 wiki 的推荐基线）
uci set wireless.radio0.country='CN'          # 5GHz，信道 36 或 auto，80MHz，WPA2
uci set wireless.radio1.country='CN'          # 2.4GHz，20MHz，WPA2
uci set wireless.radio2.disabled='1'          # 关掉 mwifiex radio2
uci set wireless.default_radio0.dtim_period='2'
uci commit wireless

# 3) Turbo ACC：默认只开 BBR，offload 与 fullcone 先关着试
uci set turboacc.config.sw_flow='0'
uci set turboacc.config.hw_flow='0'
uci set turboacc.config.fullcone_nat='0'
uci set turboacc.config.bbr_cca='1'
uci commit turboacc

# 4) 应用内核参数后重启
sysctl -p /etc/sysctl.d/99-wrt1900ac-tuning.conf
reboot
```

UAS（USB3.0 硬盘）兼容问题：个别硬盘盒在 UAS 下会反复掉盘，给该设备加 quirks 即可：

```sh
# 在 /etc/modules.d/ 或内核 cmdline 加：usb-storage.quirks=XXXX:YYYY:u
# XXXX:YYYY 用 lsusb 查
```

---

## 七、构建

```bash
git clone <你的 LEDE 源>
cd lede

cp /path/to/.config .
mkdir -p files/etc/sysctl.d
cp /path/to/99-wrt1900ac-tuning.conf files/etc/sysctl.d/

./scripts/feeds update -a
./scripts/feeds install -a
make defconfig        # 关键：展开精简配置
make -j$(nproc) V=s
```

产物在 `bin/targets/mvebu/cortexa9/`：

- `*-squashfs-factory.img` — 从原厂 WebUI 首次刷入
- `*-squashfs-sysupgrade.bin` — 已有 OpenWrt 时升级
- `*-initramfs-kernel.bin` — UART/kwboot 救砖用，**务必留一份**

### 与你的 GitHub Actions 对接

`.config` 直接放仓库根目录即可（对应 `CONFIG_FILE` 环境变量）。三点提醒：

1. `diy-part1.sh` 负责挂 feed（OpenClash / helloworld 等），`.config` 里的 `luci-app-openclash` 只有在 feed 装好之后才能解析；缺 feed 时 `make defconfig` 会警告并忽略该符号，不会构建失败。
2. `diy-part2.sh` 里那个 IEI WT61P803 PUZZLE 9xx 内核补丁与 `.config` 无关，但 feeds 更新后仍可能再次冲突，CI 变红时优先查它。
3. 若 `make defconfig` 报 unknown symbol，多半是 Lean 源里没有某个包（例如 `luci-app-cpufreq` 是 Lean 专属，`luci-app-advanced-reboot`/`irqbalance`/`sqm`/`watchcat`/`uhttpd`/`ttyd` 在官方 luci feed 里）。删掉对应行即可。

---

## 八、参考来源

- [OpenWrt ToH — Linksys WRT AC Series](https://openwrt.org/toh/linksys/wrt_ac_series)：双分区机制、mwlwifi 能力边界（无 MU-MIMO / 802.11s / 802.11w）、SFO 推荐、A-MSDU 修复说明、DFS 与 mwifiex 冲突
- [OpenWrt ToH — Linksys WRT1900AC](https://openwrt.org/toh/linksys/wrt1900ac)：Flash Layout、WPA3 支持范围（23.05.3+，仅 88W8864 机型）
- [OpenWrt 25.12.5 mvebu/cortexa9 官方构建产物](https://downloads.openwrt.org/releases/25.12.5/targets/mvebu/cortexa9/)：device_packages、kernel 6MB 上限、默认包集合
- [OpenWrt Forum — mvebu mv88e6xxx 模块化讨论](https://forum.openwrt.org/t/mvebu-switch-to-use-a-module-for-mv88e6xxx/223289)：`kmod-dsa-mv88e6xxx` 必须显式选中
- [OpenWrt Forum — WRT1900AC 吞吐问题讨论](https://forum.openwrt.org/t/linksys-wrt1900ac-v1-throughput-issues/197525)：DSA 与 flow offload 的历史冲突
- 你的仓库 `jarodvip/linksys` main 分支 `.config`：现有配置基准（Lean 源、内核 6.6、fw3、OpenClash+ruby）
