# 参考架构

## 概述

本章汇总由前述组件构建的端到端参考设计，给出三种标准模式：

1. **冗余互联网边缘**，使用 [carp(4)](https://man.openbsdhandbook.com/carp.4/)、[pfsync(4)](https://man.openbsdhandbook.com/pfsync.4/)、[pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/) 中的按上行 NAT 与策略路由，由 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/) 管理。
2. **园区/分支机构路由接入**，使用 VLAN SVI、[rad(8)](https://man.openbsdhandbook.com/rad.8/) 的路由通告、[unbound(8)](https://man.openbsdhandbook.com/unbound.8/) 的内部递归、[nsd(8)](https://man.openbsdhandbook.com/nsd.8/) 的权威区域，以及 [dhcpd(8)](https://man.openbsdhandbook.com/dhcpd.8/) 的 DHCP。
3. **区域 PoP**，使用 [ldpd(8)](https://man.openbsdhandbook.com/ldpd.8/) 与 [mpe(4)](https://man.openbsdhandbook.com/mpe.4/) 的 MPLS LDP 核心，或使用 [iked(8)](https://man.openbsdhandbook.com/iked.8/) 或 [wg(4)](https://man.openbsdhandbook.com/wg.4/) 的加密中心-分支。BGP 策略见 OpenBGPD 章节。

每种模式均包含最小化、可复制粘贴的配置与验证指引。

## 设计考量

- **故障域。** 将状态同步与控制流量置于专用链路。除非确有必要，避免跨路由器桥接。
- **配置一致性。** HA 对上保持 PF 与服务配置相同；仅设备特定的接口地址可不同。
- **变更管理。** 首次部署时使用原子 PF 重载与回滚定时器。保留一个故障安全锚点以维持管理访问。
- **地址规划。** 按 VLAN 分配确定性的 /24（IPv4）与 /64（IPv6），并与站点和角色结构化映射。
- **可观测性。** 启用 [pflogd(8)](https://man.openbsdhandbook.com/pflogd.8/)，并（可选）向收集器启用 [pflow(4)](https://man.openbsdhandbook.com/pflow.4/)；用 [ntpd(8)](https://man.openbsdhandbook.com/ntpd.8/) 保持时钟准确。

---

## 模式 1 — 冗余互联网边缘（双防火墙，双 ISP）

**范围。** 两台 OpenBSD 防火墙组成 HA；两条 WAN 上行（ISP-A 与 ISP-B）；用 `route-to` 与 `reply-to` 实现确定性的出站引导与对称入站。

### 拓扑与接口假定

- `em0` = ISP-A，`em3` = ISP-B，`em1` = LAN，`em2` = SYNC
- CARP VIP：WAN-A `198.51.100.2/29`（vhid 10），WAN-B `203.0.113.2/29`（vhid 20），LAN `10.10.10.1/24`（vhid 30）
- 节点 A 偏好（advskew 0），节点 B 备份（advskew 100）

### 系统与接口（节点 A；节点 B 上以 advskew 100 镜像）

```sh
# printf '%s\n' 'net.inet.carp.preempt=1' >> /etc/sysctl.conf
# sysctl net.inet.carp.preempt=1
  # 修复后偏好指定的 MASTER

# cat > /etc/hostname.em2 <<'EOF'
inet 10.255.255.2 255.255.255.252
EOF
  # 状态同步 /30

# cat > /etc/hostname.pfsync0 <<'EOF'
up syncdev em2
EOF
  # 经 SYNC 的 pfsync

# cat > /etc/hostname.em0 <<'EOF'
inet 198.51.100.3 255.255.255.248
EOF
# cat > /etc/hostname.em3 <<'EOF'
inet 203.0.113.3 255.255.255.248
EOF
# cat > /etc/hostname.em1 <<'EOF'
inet 10.10.10.2 255.255.255.0
EOF

# cat > /etc/hostname.carp0 <<'EOF'
inet 198.51.100.2 255.255.255.248 vhid 10 carpdev em0 pass sharedsecret advskew 0
EOF
# cat > /etc/hostname.carp1 <<'EOF'
inet 203.0.113.2 255.255.255.248 vhid 20 carpdev em3 pass sharedsecret advskew 0
EOF
# cat > /etc/hostname.carp2 <<'EOF'
inet 10.10.10.1 255.255.255.0 vhid 30 carpdev em1 pass sharedsecret advskew 0
EOF

# sh /etc/netstart em0 em1 em2 em3 carp0 carp1 carp2 pfsync0
  # 无需重启即可激活
```

### PF 策略（两个节点）

```pf
## /etc/pf.conf — 边缘 HA + 多 WAN

set skip on { lo, pfsync0 }

# 接口与宏
lan   = "em1"
wan_a = "em0"
wan_b = "em3"
lan_net = "10.10.10.0/24"
gw_a = "198.51.100.1"
gw_b = "203.0.113.1"
web_svc = "10.10.10.50"

# 规范化
scrub in all max-mss 1452

# 允许 HA 控制平面
pass on $lan proto carp
pass on $wan_a proto carp
pass on $wan_b proto carp
pass on em2 proto pfsync

# 按上行 NAT
match out on $wan_a inet from $lan_net to any nat-to ($wan_a)
match out on $wan_b inet from $lan_net to any nat-to ($wan_b)

# 基础策略
block all

# 入站到防火墙自身，带对称应答
pass in on $wan_a proto { tcp udp icmp } to ($wan_a) reply-to ($wan_a $gw_a) keep state
pass in on $wan_b proto { tcp udp icmp } to ($wan_b) reply-to ($wan_b $gw_b) keep state

# 来自 LAN 的出站策略：
# - 批量走 ISP-B
# - 默认走 ISP-A
pass in on $lan proto { tcp udp } from $lan_net to any port { 873, 6881:6999 } \
    route-to ($wan_b $gw_b) keep state
pass in on $lan from $lan_net to any \
    route-to ($wan_a $gw_a) keep state

# 两条 ISP 上的入站服务（HTTPS）
rdr on $wan_a proto tcp to ($wan_a) port 443 -> $web_svc
pass in on $wan_a proto tcp to $web_svc port 443 reply-to ($wan_a $gw_a) keep state
rdr on $wan_b proto tcp to ($wan_b) port 443 -> $web_svc
pass in on $wan_b proto tcp to $web_svc port 443 reply-to ($wan_b $gw_b) keep state

# LAN 到防火墙服务（SSH/HTTPS 示例）
pass in on $lan proto tcp from $lan_net to ($lan) port { 22, 443 } keep state

# 防火墙的出站
pass out on { $wan_a, $wan_b } proto { tcp udp icmp } from (self) to any keep state
```

**验证（任一节点）：**

```sh
$ ifconfig carp0 carp1 carp2 | egrep 'vhid|status'
  # 每个 VIP 的 MASTER/BACKUP
$ pfctl -vvsr | egrep 'route-to|reply-to|proto carp|pfsync'
  # 策略绑定与 HA 放行
$ pfctl -vvss | head
  # 两个节点上均存在复制状态
$ tcpdump -ni pfsync0
  # SYNC 上的流更新
```

---

## 模式 2 — 园区/分支机构路由接入与本地服务

**范围。** 两台路由器用 CARP 提供 VLAN 网关，通过 `rad(8)` 通告 IPv6，并提供内部递归（Unbound）、权威区域（NSD）、DHCP 与任播解析器部署。

### 网关与 CARP（R1/R2）

假定 trunk `em1`，VLAN 10 用户 `10.10.10.0/24`，VLAN 20 服务器 `10.20.20.0/24`，VIP `.1`，路由器单播 `.2`（R1）/ `.3`（R2）。

```
# /etc/hostname.vlan10 (R1)
vlandev em1
vlan 10
inet 10.10.10.2 255.255.255.0
up
# /etc/hostname.vlan20 (R1)
vlandev em1
vlan 20
inet 10.20.20.2 255.255.255.0
up
# /etc/hostname.carp0 (R1)
inet 10.10.10.1 255.255.255.0 vhid 10 carpdev vlan10 pass shared advskew 0
# /etc/hostname.carp1 (R1)
inet 10.20.20.1 255.255.255.0 vhid 20 carpdev vlan20 pass shared advskew 0
```

在 R2 上以 `.3` 与 `advskew 100` 镜像。

### 服务节点上的路由通告与 DNS

`rad(8)` 通告 /64；Unbound 提供递归；NSD 提供你的区域；DHCP 提供 IPv4。

```
## /etc/rad.conf
interface vlan10
    prefix 2001:db8:10:10::/64
    rdnss 2001:db8:10:10::53
    dnssl corp.example
interface vlan20
    prefix 2001:db8:10:20::/64
    rdnss 2001:db8:10:20::53
    dnssl corp.example
```
```
## /var/unbound/etc/unbound.conf (摘录)
server:
    interface: 10.10.10.53
    interface: 10.20.20.53
    access-control: 10.10.10.0/24 allow
    access-control: 10.20.20.0/24 allow
    auto-trust-anchor-file: "/var/unbound/db/root.key"
```
```
## /etc/nsd.conf (摘录)
server:
    ip-address: 10.20.20.53
    zonesdir: "/var/nsd/zones"
zone:
    name: "corp.example"
    zonefile: "corp.example.zone"
```
```
## /etc/dhcpd.conf (摘录)
subnet 10.10.10.0 netmask 255.255.255.0 {
    range 10.10.10.100 10.10.10.200;
    option routers 10.10.10.1;
    option domain-name-servers 10.10.10.53;
}
subnet 10.20.20.0 netmask 255.255.255.0 {
    range 10.20.20.100 10.20.20.200;
    option routers 10.20.20.1;
    option domain-name-servers 10.20.20.53;
}
```

### 任播解析器地址

从两台解析器发布 `10.10.255.53/32`，并在 R1/R2 上用 ECMP 路由。

```sh
# ifconfig lo0 alias 10.10.255.53/32
  # 在每台解析器上

# route add -mpath -host 10.10.255.53 10.20.20.11
# route add -mpath -host 10.10.255.53 10.20.20.12
  # 在每台路由器上；用 'route -n show' 确认
```

**验证：**

```sh
$ ifconfig vlan10 vlan20 carp0 carp1 | egrep 'status|vhid'
$ drill @10.10.10.53 openbsd.org A
$ ping -n -c 3 10.10.255.53
$ ndp -an | head
```

---

## 模式 3 — 区域 PoP

常见两种选项：

### A) 带 L3 VPN 的 MPLS LDP 核心

在核心链路上启用 MPLS，运行 `ldpd(8)`，并通过 [mpe(4)](https://man.openbsdhandbook.com/mpe.4/) 接入客户 VRF。BGP VPN 策略在 OpenBGPD 中定义。

```sh
# ifconfig em0 mpls
# ifconfig em1 mpls
# rcctl enable ldpd ; rcctl start ldpd
```

```
## /etc/ldpd.conf (核心接口)
router-id 192.0.2.1
address-family ipv4 {
    interface em0
    interface em1
}
```

用 `mpe0` 将客户 VRF（rdomain 10）绑定到 MPLS：

```sh
# ifconfig em2 rdomain 10 inet 10.10.10.1/24
# ifconfig mpe0 create
# ifconfig mpe0 rdomain 10 mplslabel 2000 inet 192.198.0.1/32 up
```

通过 OpenBGPD VPN AFI/SAFI 通告客户前缀（`bgpd.conf` 的 `vpn` 段及到其他 PE 的 iBGP 参见 OpenBGPD 章节）。

**验证：**

```sh
$ ldpctl show neighbors
$ ldpctl show lib | head
$ route -T 10 -n show
```

### B) 加密中心-分支（IKEv2）

小型 PoP 可用 `iked(8)` 将分支连接到区域中心。

```sh
## /etc/iked.conf — 分支到中心（按站点选择器）
ikev2 "spoke-hub" active esp from 10.30.0.0/24 to 10.0.0.0/16 \
    peer 198.51.100.50 \
    ikesa enc aes-256-gcm prf sha256 group 14 \
    childsa enc aes-256-gcm \
    psk "change-me"
```

```sh
# rcctl enable iked ; rcctl start iked
# ikectl show sa ; ikectl show flows
```

**PoP PF 骨架（两种选项）：**

```pf
## /etc/pf.conf — PoP 边缘（摘录）
set skip on lo
block all
pass out on egress proto { tcp udp icmp } keep state
# 允许来自 NOC 的管理
pass in on egress proto tcp from 198.51.100.0/24 to (egress) port 22 keep state
# 允许 VRF 或隧道接口上的客户 LAN（按需调整）
pass in on enc0 from 10.30.0.0/24 to any keep state
# 或对 MPLS VRF：
# pass in on mpe0 from 10.30.0.0/24 to any keep state
```

---

## 验证

- **HA 边缘：** `ifconfig carp*`、`pfctl -vvsr` 中 `route-to`/`reply-to` 的计数器、`tcpdump -ni pfsync0`。
- **园区服务：** `drill`、`unbound-control status`、`nsd-control stats`、`dhcpd` 租约、`rad(8)` 在正确 VLAN 上通告。
- **PoP：** `ldpctl show neighbors/lib`、`bgpctl show summary`（适用时）、`ikectl show sa/flows`。
- **遥测：** `tcpdump -i pflog0`、`pfctl -vvsr | grep log`、`ntpctl -s status`。

## 故障排查

- **边缘非对称路径。** 入站 NAT 的 WAN 规则缺失 `reply-to` 是最常见原因。按上行添加并清除受影响状态。
- **状态失步。** 核对 `pfsync0` 的 `syncdev` 与 PF 放行。确保两节点运行相同规则集与 NAT 模型。
- **IPv6 客户端 SLAAC 失败。** 确认 `rad(8)` 每个二层网段仅通告一个 /64、ICMPv6 已放行，且仅预期路由器发送 RA。
- **任播黑洞。** 从 ECMP 路由移除失效下一跳或自动化撤销；确保解析器确实绑定了任播地址。
- **PoP 标签缺口。** MPLS 须确认核心接口已启用 `mpls` 且出现在 `ldpd.conf` 中。加密分支须检查 UDP 500/4500 可达性与 PSK 一致。
- **服务端口冲突。** 权威（NSD）与递归（Unbound）须使用不同 IP 或端口；切勿将递归暴露到互联网。

## 参见

- [网络](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- 相关：[**高可用与状态复制**](high-availability.md)
- 相关：[**多 WAN 与策略路由**](multi-wan-policy-routing.md)
- 相关：[**大规模网络服务**](network-services-at-scale.md)
- 相关：[**MPLS 与标签分发**](mpls.md)
