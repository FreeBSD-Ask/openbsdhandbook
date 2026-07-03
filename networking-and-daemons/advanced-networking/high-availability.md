# 高可用与状态复制

## 摘要

本章介绍如何构建双节点 OpenBSD 防火墙或网关集群，使用 [carp(4)](https://man.openbsdhandbook.com/carp.4/)
虚拟 IP、[pfsync(4)](https://man.openbsdhandbook.com/pfsync.4/)
状态复制，以及可选的、由 [relayd(8)](https://man.openbsdhandbook.com/relayd.8/)
提供的服务故障切换。文中给出内网与外网虚拟 IP（VIP）、专用状态同步网络及安全故障切换行为的标准配置。当单一网关或防火墙的故障会影响生产流量时，应采用此模式。包过滤配置在 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
中，运行时由 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
管理。

## 设计考量

- **拓扑。** 每个节点使用三块物理接口：**WAN**、**LAN** 与 **SYNC**。将 [pfsync(4)](https://man.openbsdhandbook.com/pfsync.4/)
  放在不承载其他流量的专用 L2 网段上。
- **编址。** 为每个节点在 WAN 与 LAN 上分配固定单播地址，并按网段创建具备唯一 VHID 的 CARP VIP。示例如下：
  - WAN：节点 A `203.0.113.2/29`，节点 B `203.0.113.3/29`，VIP `203.0.113.1/29`，VHID 10
  - LAN：节点 A `10.10.10.2/24`，节点 B `10.10.10.3/24`，VIP `10.10.10.1/24`，VHID 20
- **优先级与抢占。** `advskew` 较小者胜出。启用 `net.inet.carp.preempt=1`，使优先节点在修复后重新接管 MASTER。
- **隔离与过滤。** 在成员接口上允许 `proto carp`，仅在 SYNC 接口上允许 `proto pfsync`。在 SYNC 接口上跳过过滤。
- **故障域。** SYNC 链路故障必须触发 CARP 降级，使健康对端成为 MASTER。若可能出现非对称故障，需监控上游可达性。
- **配置对齐。** 两个节点上的 PF 与服务配置须保持一致。状态会复制，规则不会。
- **运维护栏。** 在维护窗口内测试故障切换。发布期间以较高日志级别记录 CARP 事件。

## 配置

示例假定接口名为 `em0`（WAN）、`em1`（LAN）、`em2`（SYNC）。请根据硬件调整。接口文件详见 [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
。接口配置与 CARP 参数由 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
管理。系统控制项由 [sysctl(8)](https://man.openbsdhandbook.com/sysctl.8/)
设置，并通过 [sysctl.conf(5)](https://man.openbsdhandbook.com/sysctl.conf.5/)
持久化。服务由 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
管理。

### 系统设置（两个节点）

```sh
# printf '%s\n' \
    'net.inet.carp.preempt=1' \
    'net.inet.carp.log=2' \
    >> /etc/sysctl.conf
  # 启用 CARP 抢占并提高日志详细级别；重启后持久生效

# sysctl net.inet.carp.preempt=1 net.inet.carp.log=2
  # 立即在运行时应用
```

### 节点 A（优先 MASTER）

```sh
# cat > /etc/hostname.em0 <<'EOF'
inet 203.0.113.2 255.255.255.248
EOF
  # em0 上的 WAN 单播地址

# cat > /etc/hostname.em1 <<'EOF'
inet 10.10.10.2 255.255.255.0
EOF
  # em1 上的 LAN 单播地址

# cat > /etc/hostname.em2 <<'EOF'
inet 10.255.255.2 255.255.255.252
EOF
  # em2 上的 SYNC 网络（专用 /30）

# cat > /etc/hostname.carp0 <<'EOF'
inet 203.0.113.1 255.255.255.248 vhid 10 carpdev em0 pass sharedsecret advskew 0
EOF
  # WAN VIP：VHID 10，advskew 最低以优先成为 MASTER

# cat > /etc/hostname.carp1 <<'EOF'
inet 10.10.10.1 255.255.255.0 vhid 20 carpdev em1 pass sharedsecret advskew 0
EOF
  # LAN VIP：VHID 20

# cat > /etc/hostname.pfsync0 <<'EOF'
up syncdev em2
EOF
  # pfsync 走 em2；默认在 SYNC 网段上多播

# sh /etc/netstart em0 em1 em2 carp0 carp1 pfsync0
  # 无需重启即可立即启用接口
```

### 节点 B（BACKUP）

```sh
# cat > /etc/hostname.em0 <<'EOF'
inet 203.0.113.3 255.255.255.248
EOF

# cat > /etc/hostname.em1 <<'EOF'
inet 10.10.10.3 255.255.255.0
EOF

# cat > /etc/hostname.em2 <<'EOF'
inet 10.255.255.3 255.255.255.252
EOF

# cat > /etc/hostname.carp0 <<'EOF'
inet 203.0.113.1 255.255.255.248 vhid 10 carpdev em0 pass sharedsecret advskew 100
EOF
  # advskew 较高，使此节点在 A 健康时保持 BACKUP

# cat > /etc/hostname.carp1 <<'EOF'
inet 10.10.10.1 255.255.255.0 vhid 20 carpdev em1 pass sharedsecret advskew 100
EOF

# cat > /etc/hostname.pfsync0 <<'EOF'
up syncdev em2
EOF

# sh /etc/netstart em0 em1 em2 carp0 carp1 pfsync0
```

### PF 策略（两个节点）

下列最小策略启用 pfsync 与 CARP 流量，在 SYNC 接口上跳过过滤，LAN 上放行全部流量，WAN 上默认阻断。请按自身安全模型调整。语法定义见 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
。加载与查询使用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
。

```sh
## /etc/pf.conf（最小 HA 骨架）

set skip on { lo, pfsync0 }

# 接口
wan = "em0"
lan = "em1"
sync = "em2"

# 放行 HA 控制平面
pass on $wan proto carp
pass on $lan proto carp
pass on $sync proto pfsync

# 示例策略（生产环境须收紧）
block all
pass on $lan
pass out on $wan proto { tcp udp icmp } from ($wan) to any keep state
```

```sh
# pfctl -f /etc/pf.conf
  # 原子化重载规则
# pfctl -sr
  # 验证规则包含 proto carp 与 proto pfsync
```

### 可选：使用 relayd 为服务提供 L4 VIP（两个节点）

使用 [relayd.conf(5)](https://man.openbsdhandbook.com/relayd.conf.5/)
在 LAN 侧前置一个服务 VIP，使其随 CARP 故障切换。守护进程由 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
管理，运行时使用 `relayctl(8)`。

```pf
## /etc/relayd.conf（带 TCP 健康检查的简单中继）
table <web_backends> { 10.10.10.11, 10.10.10.12 }

relay "www" {
    listen on 10.10.10.1 port 80
    forward to <web_backends> check tcp port 80
}
```

```sh
# rcctl enable relayd
  # 开机启动
# rcctl start relayd
  # 立即启动
```

### 可选：在 SYNC 中断或上游丢失时进行事件驱动降级

除内核在特定故障时自动降级外，运维人员常在 SYNC 中断或上游不可达时手动降级 CARP。最简单的运维控制方法是调整受影响节点上所有 CARP VIP 的 `advskew`。下例强制节点转为 BACKUP，或恢复为优先 MASTER：

```sh
# ifconfig carp0 advskew 240
  # 大幅降级此节点的 WAN VIP

# ifconfig carp1 advskew 240
  # 大幅降级此节点的 LAN VIP

# ifconfig carp0 advskew 0
  # 恢复 WAN VIP 的优先优先级

# ifconfig carp1 advskew 0
  # 恢复 LAN VIP 的优先优先级
```

可借助 `ifstated(8)` 等策略工具实现该决策的自动化（此处不展开，条件语法请参阅其手册页），在链路或可达性故障时触发降级。

## 验证

- **接口角色。** 确认每个 CARP VIP 的 MASTER/BACKUP 状态：

```sh
$ ifconfig carp0
  # 显示 "MASTER" 或 "BACKUP" 及 VHID/advskew
$ ifconfig carp1
$ ifconfig pfsync0
  # 确认 syncdev em2 与 up 状态
```

- **状态复制。** 通过集群发起一条连接，然后用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
  验证两个节点上是否存在状态：

```sh
$ pfctl -vvss | head
  # 显示已复制状态，及其创建者、年龄与计数器
```

- **链路层同步。** 使用 [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
  在 SYNC 接口上观察 pfsync 流量：

```sh
# tcpdump -n -e -ttt -i pfsync0
  # 预期可见稳态更新；新流与故障切换期间会出现突发
```

- **CARP 健康。** 使用 [sysctl(8)](https://man.openbsdhandbook.com/sysctl.8/)
  查看 CARP sysctl：

```sh
$ sysctl net.inet.carp
  # 显示 allow、preempt、日志级别与当前 demotion 值
```

- **运行时面板。** 使用 [systat(1)](https://man.openbsdhandbook.com/systat.1/)
  的 `pf` 视图查看实时计数器，并检查 `/var/log/messages` 中的 CARP 角色变更。

## 故障排查

- **MASTER/MASTER 状态。** 两个节点在同一 VLAN 上均自称 MASTER。检查 VHID 冲突、`pass` 不一致或广播域被隔离。查看 `ifconfig carpX` 并修正 VHID 或密码。
- **状态未复制。** 确认 `pfsync0` 已 up 且 `syncdev` 正确，PF 已启用，且 `proto pfsync` 仅在 SYNC 接口上放行。用 `tcpdump -i pfsync0` 验证。
- **角色抖动。** 上游不稳定或非对称路径可能引发抢占循环。临时提高不稳定节点的 `advskew`，排查物理链路，并考虑在修复前禁用抢占。
- **故障切换生效，但连接中断。** 确保两个节点规则集一致，NAT 设置匹配。确认状态已复制（`pfctl -vvss`），并检查 LAN 主机在角色切换时是否收到 CARP 发出的无偿 ARP。
- **故障切换后 relayd 不再服务。** 确保 `relayd` 在两个节点上运行，监听 CARP VIP，且健康检查与服务匹配。查看 `/var/log/relayd` 与 `relayctl show relays`。
- **降级值卡在高值。** 查看 `sysctl net.inet.carp.demotion`。找到并消除根因（例如 SYNC 中断）。将 `advskew` 恢复至正常值并清除故障即可重置。

## 另请参见

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- 相关：[**多 WAN 与策略路由**](multi-wan-policy-routing.md)
- 相关：[**故障排查手册**](troubleshooting-playbooks.md)
