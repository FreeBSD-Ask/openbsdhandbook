# 大规模 IPv6

## 摘要

本章介绍在 OpenBSD 上大规模运维 IPv6：接口编址、路由通告（RA）、邻居发现（ND）、上游集成与生产环境护栏。使用基本系统中的路由通告守护进程 [rad(8)](https://man.openbsdhandbook.com/rad.8/)
，配置存放于 [rad.conf(5)](https://man.openbsdhandbook.com/rad.conf.5/)
；客户端自动配置使用 [slaacd(8)](https://man.openbsdhandbook.com/slaacd.8/)
；ND 检查使用 [ndp(8)](https://man.openbsdhandbook.com/ndp.8/)
；接口管理使用 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
；包过滤在 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
中配置，由 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
应用。这些模式适用于园区网段、数据中心 Pod 与 ISP 边缘——即运维多个 /64 或更大聚合的场景。

## 设计考量

- **前缀规划。** 每个 L2 网段分配一个 /64。对于园区与 Pod，从运营商处至少获取 /56（256 × /64）或 /48（65,536 × /64）。采用层次化编址，使其与故障域和路由边界对应。
- **转发模型。** 在 VLAN 间路由，避免跨故障域延伸 L2。IPv6 首跳冗余可复用 HA 章节中的 CARP 模式。
- **路由通告。** RA 用于传递链路上前缀、默认网关、DNS（RDNSS）与搜索域（DNSSL）。RA 计时器须保持一致，每个 L2 上仅通告一个 /64。
- **ND 与 ICMPv6。** 不要阻断 ICMPv6。邻居发现、路径 MTU 发现与错误处理都依赖它。
- **前缀委派（PD）。** 上游可能动态委派 /56 或 /60。需规划如何将委派空间细分并通告给下游 L2。请参阅 OpenBSD 7.8 发行说明与 [slaacd(8)](https://man.openbsdhandbook.com/slaacd.8/)
  了解当前 PD 客户端能力。
- **NPTv6。** 优先采用正规路由。仅在受限的重新编址场景下才考虑网络前缀转换（NPTv6），并充分测试。当前支持与注意事项请见 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
  。
- **大规模运维。** 统一 RA 默认值，集中日志，并按网段监测 ND 表与错误计数器。利用变更窗口测试 DAD、RA 接收与 PMTU。

## 配置

示例假设：

- 路由器接口 `em1`（LAN-A VLAN 10）、`em2`（LAN-B VLAN 20）、`em0`（WAN）。
- 已分配聚合 `2001:db8:10::/56`，其中 `2001:db8:10:0010::/64` 为 LAN-A，`2001:db8:10:0020::/64` 为 LAN-B。

### 1) 系统转发与接口编址（路由器）

启用 IPv4/IPv6 转发，并为路由器的 LAN 接口分配 IPv6 地址。

```sh
# printf '%s\n' \
    'net.inet.ip.forwarding=1' \
    'net.inet6.ip6.forwarding=1' \
    >> /etc/sysctl.conf
  # 持久化 L3 转发

# sysctl net.inet.ip.forwarding=1 net.inet6.ip6.forwarding=1
  # 立即应用

# cat > /etc/hostname.em1 <<'EOF'
inet6 2001:db8:10:10::1 64
up
EOF
  # LAN-A：路由器地址与前缀长度

# cat > /etc/hostname.em2 <<'EOF'
inet6 2001:db8:10:20::1 64
up
EOF
  # LAN-B：路由器地址与前缀长度

# sh /etc/netstart em1 em2
  # 立即启用接口
```

### 2) 使用 rad(8) 进行路由通告

每个 L2 上通告一个 /64。包含 RDNSS 与 DNSSL，使主机通过 RA 学习 DNS 设置。

```
## /etc/rad.conf — 在两个接口上通告 /64 与 DNS

# LAN-A（VLAN 10）
interface em1
    prefix 2001:db8:10:10::/64
    rdnss 2001:db8:10:10::53
    dnssl example.internal

# LAN-B（VLAN 20）
interface em2
    prefix 2001:db8:10:20::/64
    rdnss 2001:db8:10:20::53
    dnssl example.internal
```

```sh
# rcctl enable rad
  # 开机启动
# rcctl start rad
  # 立即启动；开始发送 RA
```

> 初始发布期间每个网段只保留一台通告路由器。若部署带 CARP 的冗余路由器，确保两台都运行 `rad(8)`，并使虚拟默认网关地址出现在 DNS 与文档中。

### 3) 客户端自动配置（主机）

OpenBSD 主机使用 [slaacd(8)](https://man.openbsdhandbook.com/slaacd.8/)
根据 RA 生成地址。请确保接口请求自动配置。

```
## /etc/hostname.em1（主机端）
inet6 autoconf
up
```

```sh
# rcctl enable slaacd
# rcctl start slaacd
  # 主机将从 RA 配置全局地址与默认路由
```

### 4) PF 对 IPv6 控制平面与数据的放行

放行必需的 ICMPv6，允许 LAN 流量，并应用保守的 scrub 以稳定 PMTU。请按安全模型调整。

```pf
## /etc/pf.conf — 最小化 IPv6 放行规则

set skip on lo

lan = "{ em1, em2 }"
wan = "em0"

block all

# ICMPv6 对 ND 与 PMTU 必不可少；在所有接口上放行
pass inet6 proto icmp6 keep state

# 放行 LAN IPv6；按策略进一步收敛
pass in on $lan inet6 from { 2001:db8:10:10::/64, 2001:db8:10:20::/64 } to any keep state

# 出站至 WAN
pass out on $wan inet6 from any to any keep state

# 保守的 scrub，避免部署期间出现 PMTU 黑洞
scrub in on $wan inet6
```

```sh
# pfctl -f /etc/pf.conf
# pfctl -sr | grep icmp6
  # 验证 ICMPv6 放行规则已生效
```

### 5) 上游集成与前缀委派（概述）

当 ISP 委派前缀（例如 /56）时，按确定性方式细分（例如 VLAN-ID ↔ `2001:db8:10:NN::/64` 中的 0xNN），并通过 `rad(8)` 按各 L2 通告。视版本与运营商要求而定，前缀委派可能动态下发。请参阅 OpenBSD 7.8 文档了解 [slaacd(8)](https://man.openbsdhandbook.com/slaacd.8/)
，并规划运维流程（租约监控、变更后重新通告）。无法使用 PD 时，从自有聚合中静态分配。

### 6) 关于 NPTv6 的说明

当重新编号压力大且无法采用路由时，NPTv6 可在保留主机位的前提下，将一个全局 /64 转换为另一个。优先使用原生路由；将 NPTv6 保留给边缘场景与实验室验证。语法与约束请参阅 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
（针对 OpenBSD 7.8 版本），并在生产前于隔离网段中测试。

## 验证

- **地址与路由**：确认每个 L2 的 /64 与默认路由。

```sh
$ ifconfig em1 em2 | grep inet6
  # 验证路由器与主机的全局地址
$ route -n show -inet6
  # 主机上确认经由链路上路由器的默认路由；路由器上确认已连接的 /64
```

- **邻居发现**：使用 [ndp(8)](https://man.openbsdhandbook.com/ndp.8/)
  检查邻居缓存与路由器条目。

```sh
$ ndp -an
  # 邻居与路由器缓存；查找过期或不完整条目
```

- **链路上的 RA 与 ND**：使用 [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
  在 LAN 上抓取 ICMPv6。

```sh
# tcpdump -ni em1 icmp6
  # 预期可见路由通告、邻居请求/通告
```

- **通过 RDNSS 获取的 DNS**：验证主机是否通过 RA 学到 DNS 服务器。

```sh
$ getent hosts www.example.internal
  # 使用学到的 RDNSS 进行名称解析
```

## 故障排查

- **主机获取不到 IPv6 地址。** 确认 `rad(8)` 已运行并在正确接口上通告，路由器上 `net.inet6.ip6.forwarding=1` 已启用，且 PF 未阻断 ICMPv6。用 `tcpdump -ni emX icmp6` 观察 RA。
- **重复地址检测（DAD）失败。** 若地址停留在 `tentative` 或被标记为 `duplicate`，检查是否有意外的 RA 发送方或配置错误的任播地址。查看 `ndp -an` 中冲突的 MAC。
- **可达性时断时续或卡顿。** `Packet Too Big` 报文可能被阻断。确保端到端放行 ICMPv6，并用大包 ping（`ping -s`）验证 PMTU。
- **路由器选择异常。** 确保每个 L2 上仅有预期的路由器在通告。迁移时先在退役路由器上禁用 RA，再撤回前缀。
- **PD 前缀变更。** 当委派前缀变化时，及时更新接口地址与 `rad.conf(5)`。自动化下游更新，或从委派聚合采用确定性映射。
- **防火墙拒绝 IPv6。** 验证策略中包含显式 `inet6` 规则。混合 `inet`/`inet6` 环境中，常会误只放行 IPv4。

## 另请参见

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- 相关：[**QoS 与流量整形**](qos-traffic-shaping.md)
- 相关：[**故障排查手册**](troubleshooting-playbooks.md)
