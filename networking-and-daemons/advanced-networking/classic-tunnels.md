# 经典轻量隧道

## 摘要

本章介绍基本系统中用于务实叠加与迁移的轻量隧道机制：[gif(4)](https://man.openbsdhandbook.com/gif.4/)
（通用 IP-in-IP）、[gre(4)](https://man.openbsdhandbook.com/gre.4/)
（通用路由封装）与 [etherip(4)](https://man.openbsdhandbook.com/etherip.4/)
（基于 IP 的二层以太网）。这些接口与非 OpenBSD 设备互操作性良好，适用于不需要加密、必须承载非 IP 协议（GRE）或必须延伸 L2 域（EtherIP）的场景。开机时配置使用 [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
，运行时管理使用 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
。过滤与放行规则存放于 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
，由 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
应用。

这些机制可用于临时迁移、站点间快速叠加、实验室环境，或在不引入完整 VPN 控制平面复杂性的前提下提供简单封装。

## 设计考量

- **封装选择。**
  - **gif：** 仅支持 IP-in-IP（不含 L2）；头部最小；适合在已知端点间路由 IP 子网。
  - **gre：** 负载灵活（通常为 IP）；路由器与设备广泛支持。
  - **etherip：** 跨 IP 路径桥接以太网帧；谨慎使用，避免 L2 环路。
- **安全性。** 这些机制**不提供加密或完整性**。仅在可信或受控的底层网络上使用，或与 [pf(4)](https://man.openbsdhandbook.com/pf.4/)
  过滤结合使用。需要加密隧道时，请优先选用上一章介绍的 IKEv2 或 [wg(4)](https://man.openbsdhandbook.com/wg.4/)
  。
- **转发与路由。** 通过隧道路由子网时启用 IPv4/IPv6 转发。为远端前缀添加显式路由。
- **MTU 与分片。** 封装会降低有效 MTU。初始设置较保守的 MTU，测量路径 MTU 后再精调。
- **对称性与编址。** 隧道为点对点。两端须按正确的本地/远端顺序配置。
- **过滤。** 在底层网络上放行相应 IP 协议：IP 协议 4（gif）、47（GRE）、97（EtherIP）。
- **使用范围。** 优先使用路由型叠加（gif/gre），而非 L2 延伸。仅在必须延伸广播域时才将 [bridge(4)](https://man.openbsdhandbook.com/bridge.4/)
  与 EtherIP 结合使用。

## 配置

假设两个站点：

- **站点 A** WAN：`198.51.100.10`
- **站点 B** WAN：`203.0.113.10`

LAN：

- **A LAN：** `10.0.0.0/24`
- **B LAN：** `10.20.0.0/24`

### 通用系统设置（两个站点）

启用 L3 转发，使被路由前缀能穿越隧道。通过 [sysctl.conf(5)](https://man.openbsdhandbook.com/sysctl.conf.5/)
持久化。

```sh
# printf '%s\n' \
    'net.inet.ip.forwarding=1' \
    'net.inet6.ip6.forwarding=1' >> /etc/sysctl.conf
  # 持久化 IPv4/IPv6 转发

# sysctl net.inet.ip.forwarding=1 net.inet6.ip6.forwarding=1
  # 立即应用
```

在 PF 中于 WAN 上放行封装协议。按需调整接口与策略。

```pf
## /etc/pf.conf — 轻量隧道的底层放行规则

set skip on lo
wan = "em0"   # 请根据实际 WAN 接口调整

block all
pass out on $wan keep state

# 封装协议（为更严格策略，可限定已知对端）
pass in on $wan proto { 4, 47, 97 } from 203.0.113.10 to ($wan) keep state
pass out on $wan proto { 4, 47, 97 } to 203.0.113.10 keep state
```

用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
重载。

```sh
# pfctl -f /etc/pf.conf
```

### A. gif(4)：IP-in-IP 路由型叠加

在两个站点创建 `gif0`，分配一对点对点地址，并通过隧道路由远端 LAN。

#### 站点 A

```
# /etc/hostname.gif0
tunnel 198.51.100.10 203.0.113.10
inet 10.254.0.1 255.255.255.252 10.254.0.2
mtu 1400
up
```

```sh
# sh /etc/netstart gif0
  # 激活隧道
# route add -net 10.20.0.0/24 10.254.0.2
  # 经隧道路由 B LAN
```

#### 站点 B

```
# /etc/hostname.gif0
tunnel 203.0.113.10 198.51.100.10
inet 10.254.0.2 255.255.255.252 10.254.0.1
mtu 1400
up
```

```sh
# sh /etc/netstart gif0
# route add -net 10.0.0.0/24 10.254.0.1
```

### B. gre(4)：通用路由封装

当对端明确要求 GRE（例如路由器或云网关）时通常需要使用 GRE。配置方式与 gif 类似。

#### 站点 A

```
# /etc/hostname.gre0
tunnel 198.51.100.10 203.0.113.10
inet 10.254.1.1 255.255.255.252 10.254.1.2
mtu 1400
up
```

```sh
# sh /etc/netstart gre0
# route add -net 10.20.0.0/24 10.254.1.2
```

#### 站点 B

```
# /etc/hostname.gre0
tunnel 203.0.113.10 198.51.100.10
inet 10.254.1.2 255.255.255.252 10.254.1.1
mtu 1400
up
```

```sh
# sh /etc/netstart gre0
# route add -net 10.0.0.0/24 10.254.1.1
```

> **注意：** 若对端要求 GRE 密钥或其他属性，请按 [gre(4)](https://man.openbsdhandbook.com/gre.4/)
> 使用该实现所支持的 `ifconfig gre0 ...` 选项进行配置。

### C. etherip(4)：基于 IP 的二层延伸

EtherIP 在站点间传输以太网帧。仅在路由型叠加不可行时使用（例如必须原样承载非 IP 广播或 VLAN）。避免环路，并控制广播域范围。

#### 站点 A

```
# /etc/hostname.etherip0
tunnel 198.51.100.10 203.0.113.10
up
```

创建桥接，将本地 LAN 接口（`em1`）加入隧道。桥接参数由 [bridge(4)](https://man.openbsdhandbook.com/bridge.4/)
管理。

```
# /etc/hostname.bridge0
add em1
add etherip0
up
```

激活：

```sh
# sh /etc/netstart etherip0 bridge0
```

#### 站点 B

镜像配置，反转隧道端点。

```
# /etc/hostname.etherip0
tunnel 203.0.113.10 198.51.100.10
up
```
```
# /etc/hostname.bridge0
add em1
add etherip0
up
```

```sh
# sh /etc/netstart etherip0 bridge0
```

> **注意：** L2 延伸会跨广播域与故障域。确保桥接网段之间仅有一条活动路径。如需引入冗余，须按 [bridge(4)](https://man.openbsdhandbook.com/bridge.4/)
> 设计并测试防环路机制后再投入生产。

## 验证

- **接口状态**，使用 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
  ：

```sh
$ ifconfig gif0
  # 预期 "UP"，且隧道 src/dst 与 P2P 地址正确
$ ifconfig gre0
$ ifconfig etherip0
$ ifconfig bridge0
  # 对于 EtherIP，确保两个成员（em1、etherip0）均在桥接中
```

- **路由与可达性**，使用 [route(8)](https://man.openbsdhandbook.com/route.8/)
  与 `ping(8)`：

```sh
$ netstat -rn | egrep '10\.254\.|10\.0\.0\.|10\.20\.0\.'
  # 确认隧道 /30 与远端 LAN 路由
$ ping -n -c 3 10.20.0.1
  # 从站点 A LAN 主机到站点 B LAN 主机（gif/gre）
```

- **链路层确认**，使用 [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
  ：

```sh
# tcpdump -ni em0 proto 4
  # WAN 上的 gif（IP-in-IP）包
# tcpdump -ni em0 proto 47
  # WAN 上的 GRE 包
# tcpdump -ni em0 proto 97
  # WAN 上的 EtherIP 帧
```

- **PF 规则与计数器**，使用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
  ：

```sh
$ pfctl -vvsr | egrep 'proto (4|47|97)'
  # 验证放行规则已加载，且测试时计数器递增
```

## 故障排查

- **WAN 上无隧道流量。** 检查 PF 是否按正确协议（4、47 或 97）放行至/来自对端地址的流量。在 WAN 上用 `tcpdump` 确认封装包。
- **封装已建立但无路由。** 确认 `net.inet.ip.forwarding=1`，且指向远端 LAN 的静态路由指向隧道邻居。用 `netstat -rn` 确认。
- **可达性非对称或时断时续。** 验证两端使用匹配的本地/远端 `tunnel` 端点，且 /30 编址正确。非对称常见于参数顺序颠倒。
- **分片或卡顿。** 降低隧道 MTU（例如设为 1400）后重测。稳定后测量 PMTU，谨慎上调。
- **EtherIP 上出现广播风暴。** EtherIP 延伸了广播域。确认仅有一条 L2 路径，避免在无防环路机制下桥接额外的 WAN 链路。
- **与非 OpenBSD 对端的互操作性。** 对于 GRE，检查对端的具体要求（密钥、keepalive 或 DF 处理），相应调整 `ifconfig gre0 ...` 选项。

## 另请参见

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- 相关：[**VPN 与加密隧道**](vpn-and-crypto-tunneling.md)
- 相关：[**大规模 IPv6**](ipv6-at-scale.md)
