# 大规模二层与三层设计

## 概述

本章给出在 OpenBSD 上扩展二层与三层设计的实用模式：用 [vlan(4)](https://man.openbsdhandbook.com/vlan.4/) 进行 VLAN 规划；用 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/) 与 [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/) 实现路由接入（SVI 风格接口）；用 [carp(4)](https://man.openbsdhandbook.com/carp.4/) 实现首跳冗余；以及任播服务部署（静态 ECMP 或基于 BGP；参见 OpenBGPD）。这些模式用于限制故障域、简化运维，并为客户端与服务提供确定性路径。

## 设计考量

- **VLAN 策略。** 每个故障域或广播范围分配一个 VLAN。名称与 ID 保持层次化（例如 `SITE-FLOOR-ROLE`）。为基础设施（路由、管理、存储）预留地址段。
- **路由接入优于桥接接入。** 接入层优先采用三层（路由器上的 SVI），而非跨楼宇或楼层扩展二层。仅在确有必要时使用 [bridge(4)](https://man.openbsdhandbook.com/bridge.4/)，桥接时按端口启用 STP。
- **首跳冗余。** 用 [carp(4)](https://man.openbsdhandbook.com/carp.4/) 为每个 VLAN 提供虚拟默认网关。用 [pfsync(4)](https://man.openbsdhandbook.com/pfsync.4/) 为防火墙复制状态。
- **VLAN 间策略。** 将 VLAN 间路由视为策略边界，并在 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/) 中实施。东西向流量默认拒绝，仅放行必要流。
- **任播部署。** 对于需要弹性的本地服务（例如递归 DNS），将相同实例部署在靠近用户处。通过分发路由器上的静态 ECMP 或大型网络中的 iBGP 引导流量（参见 [bgpd(8)](https://man.openbsdhandbook.com/bgpd.8/)）。
- **MTU 与分片。** 若通过封装承载 VLAN，须校验有效 MTU，并在部署期间按需钳制 MSS。
- **文档与一致性。** 将 `hostname.if` 文件与 PF 策略纳入版本控制，并在冗余节点间保持一致。

## 配置

示例构建一个路由接入园区网，包含两台冗余路由器（**R1** 与 **R2**）。每台路由器的上行 trunk `em1` 承载 VLAN 10（用户）与 VLAN 20（服务器）。网关由 CARP VIP 提供；VLAN 间策略由 PF 实施。最后，用静态多路径路由部署任播解析器服务。

### 1) 系统转发（两台路由器）

用 [sysctl(8)](https://man.openbsdhandbook.com/sysctl.8/) 与 [sysctl.conf(5)](https://man.openbsdhandbook.com/sysctl.conf.5/) 启用三层转发与持久化设置。

```sh
# printf '%s\n' \
    'net.inet.ip.forwarding=1' \
    'net.inet6.ip6.forwarding=1' >> /etc/sysctl.conf
  # 持久化 IPv4/IPv6 转发

# sysctl net.inet.ip.forwarding=1 net.inet6.ip6.forwarding=1
  # 立即应用
```

### 2) VLAN SVI 与 CARP 网关

假定地址分配：

- VLAN 10（用户）：`10.10.10.0/24`，网关 VIP `10.10.10.1`
- VLAN 20（服务器）：`10.20.20.0/24`，网关 VIP `10.20.20.1`

**R1** 使用 `.2`，偏好 `advskew 0`；**R2** 使用 `.3`，`advskew 100`。CARP 在 VLAN 接口上运行，作为 `carpdev`。

#### R1

```
# /etc/hostname.vlan10
vlandev em1
vlan 10
inet 10.10.10.2 255.255.255.0
up
# /etc/hostname.vlan20
vlandev em1
vlan 20
inet 10.20.20.2 255.255.255.0
up
# /etc/hostname.carp0
inet 10.10.10.1 255.255.255.0 vhid 10 carpdev vlan10 pass sharedsecret advskew 0
# /etc/hostname.carp1
inet 10.20.20.1 255.255.255.0 vhid 20 carpdev vlan20 pass sharedsecret advskew 0
```

```sh
# sh /etc/netstart vlan10 vlan20 carp0 carp1
  # 将 SVI 与 VIP 上线
```

#### R2

```
# /etc/hostname.vlan10
vlandev em1
vlan 10
inet 10.10.10.3 255.255.255.0
up
# /etc/hostname.vlan20
vlandev em1
vlan 20
inet 10.20.20.3 255.255.255.0
up
# /etc/hostname.carp0
inet 10.10.10.1 255.255.255.0 vhid 10 carpdev vlan10 pass sharedsecret advskew 100
# /etc/hostname.carp1
inet 10.20.20.1 255.255.255.0 vhid 20 carpdev vlan20 pass sharedsecret advskew 100
```

```sh
# sh /etc/netstart vlan10 vlan20 carp0 carp1
```

### 3) PF 中的 VLAN 间策略（两台路由器）

允许用户与服务器 VLAN 到达各自网关，但默认限制东西向流量。按你的策略调整。用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/) 管理。

```pf
## /etc/pf.conf — 路由接入，东西向默认拒绝

set skip on lo

users_if  = "vlan10"
servers_if= "vlan20"
users_net = "10.10.10.0/24"
servers_net="10.20.20.0/24"

block all

# ICMP 与必要控制
pass inet proto icmp keep state
pass inet6 proto icmp6 keep state

# 允许主机到达默认网关（ARP/ND 由内核处理）
pass in on $users_if from $users_net to ($users_if) keep state
pass in on $servers_if from $servers_net to ($servers_if) keep state

# 南北向（到 WAN/上行）— 示例；按环境细化
pass out on egress proto { tcp udp icmp } from { $users_net, $servers_net } to any keep state

# 东西向例外（仅放行必要流）
# 示例：用户到服务器的 HTTPS
pass in on $users_if proto tcp from $users_net to $servers_net port 443 keep state
```

```sh
# pfctl -f /etc/pf.conf
  # 原子加载策略
# pfctl -sr | egrep 'vlan10|vlan20'
  # 校验规则位置
```

### 4) 任播解析器部署（静态 ECMP）

发布一个园区本地的任播解析器 IP `10.10.255.53/32`，由两台主机提供服务：`10.20.20.11`（在 VLAN 20 中）与 `10.20.20.12`。每台解析器还在其环回接口上持有 `10.10.255.53/32`，使应答以任播地址为源。

#### 在每台解析器主机上

```sh
# ifconfig lo0 alias 10.10.255.53/32
  # 本地绑定任播地址
```

#### 在每台路由器上（R1 与 R2）：添加等价路由

用 [route(8)](https://man.openbsdhandbook.com/route.8/) 配合 `-mpath` 安装多个下一跳。流量将负载分担；撤销即移除一条路径。

```sh
# route add -mpath -host 10.10.255.53 10.20.20.11
  # 经 Server-A 的第一个下一跳
# route add -mpath -host 10.10.255.53 10.20.20.12
  # 经 Server-B 的第二个下一跳
# route -n show | grep 10.10.255.53
  # 确认两条活动路径
```

> 大型网络优先采用 iBGP 任播：每台解析器从环回接口发起 `10.10.255.53/32`，分发路由器通过 BGP 学习。参见 [bgpd.conf(5)](https://man.openbsdhandbook.com/bgpd.conf.5/) 与 OpenBGPD 章节。

### 5) 可选：边缘桥接（受限范围）

桥接不可避免时（例如小型实验室），须限制广播域并在成员端口上启用 STP。用 [bridge(4)](https://man.openbsdhandbook.com/bridge.4/) 管理。

```
## /etc/hostname.bridge0 — 最小化，并在接入端口上启用 STP
add em2
add em3
stp em2
stp em3
up
```

```sh
# sh /etc/netstart bridge0
# ifconfig bridge0
  # 验证成员与 STP 状态
```

## 验证

- **SVI 与 VIP 已上线：**

```sh
$ ifconfig vlan10 vlan20 | egrep 'inet |status'
  # 检查每个 VLAN 的路由器地址
$ ifconfig carp0 carp1
  # 期望 R1 为 MASTER、R2 为 BACKUP（或测试后反之）
```

- **VLAN 间策略：**

```sh
$ pfctl -vvsr | egrep 'vlan10|vlan20|pass in on'
  # 确认预期的放行规则，且测试时计数器递增
```

- **任播可达性与路径多样性：**

```sh
$ ping -n -c 3 10.10.255.53
  # 解析器任播地址可达
$ route -n show -host 10.10.255.53
  # 已安装两个下一跳（mpath）
$ traceroute -n 10.10.255.53
  # 观察路径；撤销一个下一跳后重复，观察故障切换
```

- **在线：**

```sh
# tcpdump -ni vlan10 arp or icmp
  # 测试时观察网关 ARP/ICMP
# tcpdump -ni vlan20 host 10.10.255.53
  # 任播流向解析器
```

## 故障排查

- **客户端无法到达网关。** 核对交换机与路由器上的 VLAN 标签一致、路由器上 `vlandev` 正确。确认 CARP 在某节点为 MASTER，且 VIP 出现在正确的 VLAN 上。
- **VLAN 间流量绕过策略。** 确保路由发生在 OpenBSD 路由器（SVI）上，而非下游设备。用 `pfctl -vvsr` 复查 PF 规则顺序。
- **CARP 抖动。** 检查 `net.inet.carp.demotion` 与链路稳定性。节点间 `advskew` 不匹配或缺失 `pass sharedsecret` 会导致角色频繁切换。
- **任播黑洞。** 若某台解析器宕机但路由仍存在，移除失效下一跳（`route delete -host 10.10.255.53 10.20.20.11`）或自动化健康检查（参见 `ifstated(8)`）。BGP 任播则依赖会话撤销。
- **STP 未阻止环路。** 确认每个网桥成员启用 STP，且网段间仅有一条桥接路径。大型网络应避免桥接，将边界移至三层。
- **非对称路径与 MTU 问题。** 校验所有 trunk 承载相同的 VLAN 集合与 MTU。测试期间可暂时采用保守的 MSS 钳制。

## 参见

- [网络](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- 相关：[**高可用与状态复制**](high-availability.md)
- 相关：[**MPLS 与标签分发**](mpls.md)
- 相关：[**参考架构**](reference-architectures.md)
