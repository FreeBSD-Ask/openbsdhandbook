# 多 WAN 与策略路由

## 摘要

本章介绍如何在具有两条或多条上游链路的 OpenBSD 边缘上运行。涵盖使用 `route-to` 的出站策略、用 `reply-to` 实现的对称回流、按接口的 NAT，以及在多家运营商上向互联网发布入站服务。配置存放于 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
，由 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
管理。接口属性由 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
管理。需要自动故障切换时，可结合 [ifstated(8)](https://man.openbsdhandbook.com/ifstated.8/)
进行链路与可达性检查。当需要跨运营商分流或引导流量，或需在不使用 BGP 的前提下为入站 NAT 提供确定性回流路径时，应采用此模式。

## 设计考量

- **拓扑。** 每条 WAN 使用独立的物理接口。例如：`em0` 接 ISP-A，`em3` 接 ISP-B。保留清晰的 **LAN** 接口用于策略分类。
- **编址与网关。** 每条 WAN 有自己的网关。例如：ISP-A `198.51.100.1`，ISP-B `203.0.113.5`。
- **非对称性。** 状态过滤与 NAT 会把流绑定到创建该状态的出口。更改出口需要新状态。故障切换时需清理受影响的状态。
- **NAT 对齐。** 多 WAN 必须按接口做 NAT。若打算跨多个出口引导流量，避免使用单一 `egress` NAT。
- **策略粒度。** 按源网络、应用端口或目的地分类。简单为佳。先从按子网策略起步。
- **入站服务。** 没有 BGP 或外部 DNS 调度时，入站存在性按运营商各自独立。使用 `reply-to` 强制响应从接收连接的接口原路返回。
- **监控与自动化。** 先检测特定运营商故障，再变更策略。使用 [ifstated(8)](https://man.openbsdhandbook.com/ifstated.8/)
  触发干净的策略重载与可选的状态重置。
- **MTU。** 混合接入技术（例如 PPPoE 与以太网）可能引入分片。若观察到 PMTUD 问题，可考虑带 MSS 钳制的保守 scrub。

## 配置

下例实现两条上行链路，具备确定性的出站与入站行为。请将接口名与地址调整为与你的环境一致。

### 接口假设

- `em0` —— ISP-A，地址 `198.51.100.10/29`，网关 `198.51.100.1`
- `em3` —— ISP-B，地址 `203.0.113.10/29`，网关 `203.0.113.5`
- `em1` —— LAN，`10.10.10.0/24` 用户与服务器
- 可选的入站发布服务器：`10.10.10.50`（HTTPS）

### 带每 WAN NAT、route-to 与 reply-to 的 PF 策略

将下列内容放入 `/etc/pf.conf`。加载与查询使用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
。语法定义见 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
。

```sh
## /etc/pf.conf — Multi-WAN and policy-based routing (example)

set skip on lo

# Interfaces and networks
lan      = "em1"
wan_a    = "em0"
wan_b    = "em3"
lan_net  = "10.10.10.0/24"

# Upstream gateways
gw_a = "198.51.100.1"
gw_b = "203.0.113.5"

# Optional convenience macros
web_svc = "10.10.10.50"
tcp_services = "{ 22, 80, 443 }"

# Conservative scrub; adjust as required
scrub in all max-mss 1452

# Per-interface NAT for each uplink
match out on $wan_a inet from $lan_net to any nat-to ($wan_a)
match out on $wan_b inet from $lan_net to any nat-to ($wan_b)

# Base policy
block all

# Inbound control-plane to the firewall itself (per-WAN reply-to for symmetry)
pass in on $wan_a proto { tcp, udp, icmp } to ($wan_a) \
    reply-to ($wan_a $gw_a) keep state
pass in on $wan_b proto { tcp, udp, icmp } to ($wan_b) \
    reply-to ($wan_b $gw_b) keep state

# Outbound policy from LAN:
# - Default via ISP-A
# - Specific subnet or application class via ISP-B
# Order matters: specific policies first, then the default.

# Example: send a source subnet (or specific hosts) via ISP-B
# Replace 10.10.20.0/24 with a real subset if applicable.
# pass in on $lan from 10.10.20.0/24 to any \
#     route-to ($wan_b $gw_b) keep state

# Example: send bulk/backup traffic via ISP-B (by destination ports)
pass in on $lan proto { tcp, udp } from $lan_net to any port { 6881:6999, 873 } \
    route-to ($wan_b $gw_b) keep state

# Default: everything else goes via ISP-A
pass in on $lan from $lan_net to any \
    route-to ($wan_a $gw_a) keep state

# Inbound publishing of a HTTPS service on both providers
# rdr occurs before filtering; provide a pass rule that includes reply-to
# so return traffic follows the same ISP it arrived on.

# ISP-A publication
rdr on $wan_a proto tcp from any to ($wan_a) port 443 -> $web_svc
pass in on $wan_a proto tcp from any to $web_svc port 443 \
    reply-to ($wan_a $gw_a) keep state

# ISP-B publication
rdr on $wan_b proto tcp from any to ($wan_b) port 443 -> $web_svc
pass in on $wan_b proto tcp from any to $web_svc port 443 \
    reply-to ($wan_b $gw_b) keep state

# Allow LAN to firewall services as needed (SSH/HTTPS example)
pass in on $lan proto tcp from $lan_net to ($lan) port $tcp_services keep state

# Outbound from the firewall itself (administration, package fetches)
pass out on $wan_a inet to any keep state
pass out on $wan_b inet to any keep state
```

#### 说明

- `route-to` 在数据包**进入** PF 处生效。对于源自 LAN 的流量，用 `pass in on $lan ... route-to (...)` 匹配。
- WAN 侧 pass 规则上的 `reply-to` 将入站会话的回流流量固定到接收该连接的接口与网关。
- NAT 规则按上行链路编写，使转换跟随策略选定的出口。

### 运维开关与状态清理

在计划内故障切换或测试期间，调整策略并仅重置必须迁移的流，会比较有用。

```sh
# pfctl -f /etc/pf.conf
  # 任何变更后原子化重载

# ifconfig em0 down
  # 模拟 ISP-A 故障（测试后用：ifconfig em0 up 恢复接口）

# pfctl -k 0.0.0.0/0 -k 0.0.0.0/0
  # 清空所有 IPv4 状态，以便选择新路径
# pfctl -K "10.10.10.0/24"
  # 或仅针对 LAN 前缀选择性清除状态

# tcpdump -ni em3 icmp or port 443
  # 测试期间观察流量迁移到 ISP-B
```

若使用 [ifstated(8)](https://man.openbsdhandbook.com/ifstated.8/)
自动检测上行健康，需确保动作序列包含对预备策略的干净 `pfctl -f`，以及对受影响前缀或应用的定向状态重置。

## 验证

使用 PF 计数器、实时抓包与系统路由验证选择与对称性是否符合预期。

- **规则计数器与状态**，使用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
  ：

```sh
$ pfctl -vvsr
  # 验证特定 "route-to" 规则是否匹配且计数器递增
$ pfctl -vvss | egrep 'em0|em3'
  # 检查状态；状态详情中可见出口接口与网关选择
```

- **路径验证**，使用 [traceroute(8)](https://man.openbsdhandbook.com/traceroute.8/)
  与 [ping(8)](https://man.openbsdhandbook.com/ping.8/)
  ：

```sh
$ traceroute -n 1.1.1.1
  # 从 LAN 主机经防火墙；默认应走 ISP-A
$ traceroute -n 9.9.9.9
  # 对于被策略匹配到 ISP-B 的目的地，预期走 ISP-B 路径
$ ping -n -c 3 8.8.8.8
  # 基本可达性；切换策略时重复测试
```

- **接口级观察**，使用 [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
  ：

```sh
# tcpdump -ni em0 host 8.8.8.8 or port 443
  # 确认选中 ISP-A 出口的流
# tcpdump -ni em3 host 9.9.9.9 or port 443
  # 确认选中 ISP-B 出口的流
```

- **系统路由**，使用 [route(8)](https://man.openbsdhandbook.com/route.8/)
  ：

```sh
$ route -n show -inet
  # 系统默认路由对 PF "route-to" 决策不具有权威性，
  # 但对防火墙自身发起的流量有影响
```

## 故障排查

- **非对称回流路径。** 入站服务的 WAN pass 规则缺失或错误的 `reply-to`，会导致响应从错误的 ISP 发出。在每条 WAN 上加 `reply-to`。
- **NAT 不匹配。** 策略切换出口时若 LAN 客户端失去连接，确保每条上行链路都有对应的按接口 `match out on $wan_* nat-to ($wan_*)` 规则。
- **流粘在旧出口。** 现有状态固定在原路径上。用 `pfctl -K <subnet>` 仅清除受影响状态，或定向重启应用。
- **路径变更后性能问题。** 不同运营商间 MTU 或 MSS 差异可能破坏 PMTUD。测试期间保持保守的 `scrub ... max-mss`，事后再精调。
- **策略未匹配。** 规则顺序很关键。把特定选择器放在默认 LAN 路由规则之前。用 `pfctl -vvsr` 确认。
- **防火墙自身流量使用了错误的 ISP。** `route-to` 不影响防火墙自身发起的流量。用 [route(8)](https://man.openbsdhandbook.com/route.8/)
  调整系统默认路由，或为应用指定特定源地址。

## 另请参见

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- 相关：[**高可用与状态复制**](high-availability.md)
- 相关：[**故障排查手册**](troubleshooting-playbooks.md)
