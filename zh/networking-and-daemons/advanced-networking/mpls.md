# MPLS 与标签分发

## 概述

本章介绍如何在 OpenBSD 上部署多协议标签交换（MPLS），使用基本系统自带的标签分发协议守护进程 [ldpd(8)](https://man.openbsdhandbook.com/ldpd.8/)。内容涵盖：用 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/) 在接口上启用 MPLS；由 [ldpd.conf(5)](https://man.openbsdhandbook.com/ldpd.conf.5/) 信令驱动、基于 LDP 的核心标签交换；用运营商边缘接口 [mpe(4)](https://man.openbsdhandbook.com/mpe.4/) 配合 [bgpd.conf(5)](https://man.openbsdhandbook.com/bgpd.conf.5/) 实现三层 MPLS VPN；以及用 [mpw(4)](https://man.openbsdhandbook.com/mpw.4/) 实现二层 VPLS 伪线。在 IGP 域内进行简单、可互通的标签分发时使用 LDP；需要跨 MPLS 核心承载多租户三层服务时，采用基于 BGP 的 VPN。MPLS 的启用、禁用及按接口控制均由 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/) 提供。

## 设计考量

- **底层与发现。** LDP 邻居在具有 IP 连通性且启用 MPLS 的接口上通过链路发现建立。先确保底层（静态/OSPF/IS-IS/BGP-IGP）提供可达性；LDP 不替代 IGP。
- **接口启用。** MPLS 按接口启用（例如 `ifconfig em0 mpls`）。用 `-mpls` 禁用。调整 MTU 时须为每个标签预留 4 字节垫片开销。
- **标签处置。** 默认情况下，直连路由使用隐式空标签；需要时可通过 `explicit-null yes` 通告显式空标签（用于保留 QoS/ECN）。
- **基于 rdomain 的 VRF。** 用路由域（类似 VRF）隔离客户路由表；将 VRF 接入 MPLS 核心时，使用绑定到该 rdomain 的 [mpe(4)](https://man.openbsdhandbook.com/mpe.4/) 接口。
- **二层服务。** 对于 VPLS，在各 PE 之间建立全互联伪线，并用 [bridge(4)](https://man.openbsdhandbook.com/bridge.4/) 桥接。使用受保护域以避免环路。
- **安全与互通。** LDP 会话使用 TCP 与 hello 发现；视情况启用 GTSM 和 MD5。OpenBSD 的 `ldpd` 对两者均提供调节选项。

## 配置

示例假定一个最小的双 PE 核心：

- **PE1 底层：** `em0` ↔ 核心，`em1` ↔ 核心
- **PE2 底层：** `em0` ↔ 核心，`em1` ↔ 核心

PE 之间经核心的 IP 可达性已存在（通过 IGP 或静态路由）。

### 1) 在中转接口上启用 MPLS 并启动 LDP（两个 PE）

按接口启用 MPLS，并配置 `ldpd` 在核心链路上运行 LDP。

```sh
# ifconfig em0 mpls
  # 在面向核心的接口上允许 MPLS
# ifconfig em1 mpls
  # 在第二个面向核心的接口上允许 MPLS

# rcctl enable ldpd
  # 开机时启动 LDP
# rcctl start ldpd
  # 立即启动
```

创建一个最小的 `/etc/ldpd.conf`，在核心接口上为 IPv4 运行 LDP：

```
## /etc/ldpd.conf — 在两个接口上通过 IPv4 运行最小化 LDP

router-id 192.0.2.1         # PE1 示例；每个 PE 须唯一设置

address-family ipv4 {
    interface em0
    interface em1
    # 可选：explicit-null yes
}
```

在第二个 PE 上，将 `router-id` 设为 `192.0.2.2`，接口列表保持一致。`ldpd` 在已启用的链路上发现邻居；链路本地会话无需指定邻居 IP。

### 2) 验证跨核心的标签分发

使用 [ldpctl(8)](https://man.openbsdhandbook.com/ldpctl.8/) 和 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)：

```sh
$ ldpctl show neighbors
  # 期望每条核心链路的邻居处于 Operational 状态
$ ldpctl show lib
  # 直连路由与 IGP 学到路由的标签信息库条目
$ ifconfig em0 | egrep -i 'mpls|mtu'
  # 确认 MPLS 已启用且 MTU 合适
```

### 3) 用 mpe(4) 与 OpenBGPD 实现 L3 MPLS VPN（PE-CE）

在非默认 rdomain 内用 [mpe(4)](https://man.openbsdhandbook.com/mpe.4/) 接口将客户 VRF 绑定到 MPLS，并用 [bgpd(8)](https://man.openbsdhandbook.com/bgpd.8/) 信令通告 VPN 路由。

**PE1（示例客户 VRF rdomain 10）：**

```sh
# ifconfig em2 rdomain 10 inet 10.10.10.1/24
  # VRF 中面向 CE 的接口（客户 LAN）
# ifconfig mpe0 create
  # 创建运营商边缘接口
# ifconfig mpe0 rdomain 10 mplslabel 2000 inet 192.198.0.1/32 up
  # 将 mpe0 附加到 VRF，分配每 VPN 标签，按 bgpd.conf(5) 示例添加 /32 地址

# rcctl enable bgpd
# rcctl start bgpd
```

**PE1** 上 `/etc/bgpd.conf` 的最小摘录，用于定义 VPN 并向 **PE2** 启用 VPN AFI/SAFI：

```
## /etc/bgpd.conf — mpe0 上的 L3 VPN，与远端 PE 建立 IBGP

AS 65000
router-id 192.0.2.1

vpn "cust10" on mpe0 {
    rd 65000:10
    import-target rt 65000:10
    export-target rt 65000:10
    network 10.10.10.0/24     # 来自 VRF rdomain 10，经 mpe0
}

neighbor 192.0.2.2 {
    remote-as 65000
    descr "PE2 core session"
    announce IPv4 vpn
}
```

在 **PE2** 上镜像该设置：创建 VRF（例如 rdomain 20），附加 `mpe0`（标签 3000），在 `vpn` 块中通告其客户网络，并向 PE1 启用 `announce IPv4 vpn`。`vpn` 块使用 `mpe` 接口的标签与 /32 地址，且可绑定到与 `bgpd` 运行所在 rdomain 不同的 rdomain。

**验证（任一 PE）：**

```sh
$ bgpctl show summary
  # 与远端 PE 的会话应为 Established
$ bgpctl show rib vpn
  # VPN RIB 应列出带 RT/RD 的客户前缀，且 mpe0 为出接口
$ route -T 10 -n show
  # 客户 VRF（rdomain 10）显示经 mpe0 的远端 VPN 路由
```

### 4) 用 mpw(4) 伪线与 bridge(4) 实现 L2 VPLS

对于二层多点服务，创建到远端 PE 的伪线并与 CE 端口桥接。`ldpd` 可信令 VPLS 并将伪线绑定到 `mpw(4)`。

**PE1：**

```sh
# ifconfig mpw1 create up
  # 创建伪线接口（参数将由 ldpd 设置）
# ifconfig bridge0 create
# ifconfig bridge0 add em2
  # 将面向 CE 的端口加入网桥
# ifconfig bridge0 add mpw1
  # 将伪线加入网桥
```

**PE1** 上 `/etc/ldpd.conf` 的 VPLS 块：

```
## /etc/ldpd.conf — 含一条伪线的 VPLS 实例（PE1）

router-id 192.0.2.1

l2vpn "cust-vpls" type vpls {
    bridge bridge0
    interface em2
    pseudowire mpw1 {
        pw-id 100
        neighbor-id 192.0.2.2    # PE2 的 LSR-ID
    }
}
```

在 **PE2** 上，创建 `mpw1`，加入本地 `bridge0`，使用相同的 `pw-id`，并将 `neighbor-id` 设为 `192.0.2.1`。如有需要，调整 `mpw(4)` 上的 MTU 及控制字/流感知开关（`pwecw`、`-pwefat`）。

**验证（任一 PE）：**

```sh
$ ldpctl show neighbors
  # VPLS 目标邻居应显示为 up
$ ifconfig mpw1
  # 本地/远端标签（若已设置）与运行状态
$ ifconfig bridge0
  # 成员列表（CE 端口 + mpw1）及受保护状态（若已配置）
```

> **注：** 通过 GRE 底层传输时，[gre(4)](https://man.openbsdhandbook.com/gre.4/) 可封装 MPLS；须相应规划 MTU。

## 验证

- **核心 LDP 健康：** `ldpctl show neighbors`、`ldpctl show lib`。
- **接口状态：** `ifconfig <if> | grep -i mpls` 确认 MPLS 启用与 MTU。
- **L3VPN 控制平面：** `bgpctl show summary`（Established）、`bgpctl show rib vpn`（路由、标签、mpe）。
- **VRF 数据平面：** `route -T <rdomain> -n show` 及在 CE 子网间 traceroute，确认经 MPLS 核心转发。

## 故障排查

- **无 LDP 邻居。** 确保链路两端均启用 MPLS、底层 IP 可达、接口已列入 `ldpd.conf`。检查发现/传输优先级与 GTSM（多跳时）。
- **直连路由缺少标签。** 检查 `ldpctl show lib`。对需要 EXP/ECN 处理的特定路径，考虑使用 `explicit-null yes`。
- **VRF 中无 L3VPN 路由。** 确认 `mpe` 处于正确的 rdomain、带有 /32 地址与 `mplslabel`，且邻居协商了 VPN AFI/SAFI（`announce IPv4 vpn`）。
- **伪线 down。** 核对 `pw-id` 匹配、`neighbor-id`（LSR-ID）正确、MTU 一致。按需在 `mpw(4)` 上调整控制字（`pwecw`）或流感知传输（`-pwefat`）。
- **MTU 黑洞。** MPLS 每个标签增加 4 字节（常见两个标签）。降低载荷 MTU 或尽可能提升底层 MTU；用大包 ping 重新测试。

## 参见

- [网络](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- 相关：[**参考架构**](reference-architectures.md)
- 相关：[**高可用与状态复制**](high-availability.md)
