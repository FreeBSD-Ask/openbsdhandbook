# 故障排查剧本

## 概述

本章为常见生产环境故障提供确定性的排查剧本：高可用（HA）角色抖动、多 WAN 边缘的非对称路径、隧道 MTU 黑洞、IPv6 邻居发现（ND）与路由通告（RA）故障，以及服务质量（QoS）的实用验证方法。每个剧本仅使用基本系统工具：[pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
中的包过滤策略，用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
查看；接口用 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
；实时抓包用 [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
；系统控制用 [sysctl(8)](https://man.openbsdhandbook.com/sysctl.8/)
；可达性测试用 [ping(8)](https://man.openbsdhandbook.com/ping.8/)
与 ping6(8)；路径检查用 [traceroute(8)](https://man.openbsdhandbook.com/traceroute.8/)
；IPv6 邻居工具用 [ndp(8)](https://man.openbsdhandbook.com/ndp.8/)
；以及按需使用的服务专用工具，如 IPsec 的 [ikectl(8)](https://man.openbsdhandbook.com/ikectl.8/)
、relayd 的 `relayctl(8)`。设备级特性包括 [carp(4)](https://man.openbsdhandbook.com/carp.4/)
、[pfsync(4)](https://man.openbsdhandbook.com/pfsync.4/)
、[wg(4)](https://man.openbsdhandbook.com/wg.4/)
、[gif(4)](https://man.openbsdhandbook.com/gif.4/)
与 [gre(4)](https://man.openbsdhandbook.com/gre.4/)
。

## 设计考量

- **先复现，再缩小范围。** 复现故障，在靠近故障点处抓一次包，借助计数器与状态表避免猜测。
- **以出口视角优先。** 在策略边缘，状态与 NAT 将流量固定到出口。改路由前先查状态。
- **一次只改一处。** 做一处可回滚的改动，观察结果，结论不明确就还原。
- **时钟与日志。** 确保时间准确（参见 [ntpd(8)](https://man.openbsdhandbook.com/ntpd.8/)
  ）。用 **/var/log/messages** 与 PF 规则计数器对齐观测结果。
- **安全。** 生产事故期间，优先降级节点或隔离路径，而非整体重启。保留一个故障安全 PF 锚点用于管理访问（参见加固章节）。

## 配置（运维工具箱）

在每台设备上准备一份最小化、可复用的工具集。

```sh
# rcctl enable pflogd ; rcctl start pflogd
  # 持久化并启动 PF 日志记录到 /var/log/pflog

# pfctl -sr | sed -n '1,40p'
  # 快照当前活动规则顺序，便于后续关联

# sysctl net.inet.carp
  # 基线 CARP 抢占/日志/降级值（在 HA 节点上）

# install -d -m 700 /root/ts ; date > /root/ts/incident.start
  # 创建带起始时间戳的工作暂存区
```

---

## 剧本 1 — HA 角色抖动（CARP/pfsync）

**症状。** 频繁的 MASTER↔BACKUP 翻转，短暂中断，或状态失步。

**可能原因。** SYNC 链路不稳、VHID/口令不一致、上游可达性不对称，或抢占过于激进。

### 步骤

1. **观察 CARP 状态与降级值。**

```sh
$ ifconfig carp0 carp1
  # 预期每个 VIP 对应一个 MASTER 与一个 BACKUP
$ sysctl net.inet.carp.demotion
  # 非零表示存在故障（如接口 down、pfsync 异常）
$ tail -n 100 /var/log/messages | egrep -i 'carp|pfsync'
  # 角色变更与原因的时间线
```

2. **验证 SYNC 域与状态复制。**

```sh
$ ifconfig pfsync0
  # 确认 syncdev 与 UP 状态
# tcpdump -ni pfsync0
  # 负载下观察持续更新；新流/故障转移时出现突发
$ pfctl -vvss | head
  # 两节点上的状态条目、创建者与计数器
```

3. **检查 HA 控制面放行规则与 VHID/密钥。**

```sh
$ pfctl -sr | egrep 'proto (carp|pfsync)'
  # 放行 carp 与 pfsync 的规则
$ grep -H carpdev /etc/hostname.carp*
  # 两节点的 VHID 与共享口令是否一致？
```

4. **先稳定，再排查。**

```sh
# ifconfig carp0 advskew 240 ; ifconfig carp1 advskew 240
  # 降级本节点（让对端优先做 MASTER）
# sysctl net.inet.carp.preempt=0
  # 临时禁用抢占，避免来回切换
```

5. **常见修复。** 更换或隔离 SYNC 线缆/VLAN，在 PF 中确保 `set skip on pfsync0`，对齐规则集与 NAT，然后再重新启用抢占并恢复 `advskew`。

---

## 剧本 2 — 非对称路径与入站 NAT 故障

**症状。** 部分目的可达，部分超时。入站服务能连接，但回应从错误的 WAN 出去。出站策略看似被忽略。

**可能原因。** NAT 服务的 WAN pass 规则缺少 `reply-to`，状态过期，或 `route-to` 规则顺序有误。

### 步骤

1. **确认规则匹配与状态出口。**

```sh
$ pfctl -vvsr | egrep 'route-to|reply-to'
  # 确保存在每 WAN 的固定规则且顺序正确
$ pfctl -vvss | egrep 'em0|em3|Gateway'
  # 查看状态选定的出口接口/网关
```

2. **复现时同时在两条 WAN 上抓包。**

```sh
# tcpdump -ni em0 host <client-or-server> and port 443
# tcpdump -ni em3 host <client-or-server> and port 443
  # 确认 SYN/ACK 或回应走哪个接口
```

3. **采取纠正措施。**

```sh
# pfctl -K 10.10.10.50
  # 清除已发布服务器的状态，使新状态采用修正后的策略
# pfctl -f /etc/pf.conf
  # 添加缺失的 reply-to/route-to 后重新加载
$ route -n show -inet | head
  # 注意：系统默认路由仅影响防火墙自身发起的流量
```

---

## 剧本 3 — 隧道 MTU / 黑洞检测（IPsec、WireGuard、gif/gre）

**症状。** 小包成功，大流量卡住。TLS 握手完成但大响应挂起。看不到 ICMP "Too Big"。

**可能原因。** 封装开销超过底层 MTU；PMTUD 被阻断；MSS 过大。

### 步骤

1. **以递增的有效载荷大小探测。**

```sh
$ ping -n -c 3 -s 1400 <remote-ipv4>
  # 尝试大有效载荷；逐步减小（如 1380、1360）直到成功
$ ping6 -n -c 3 -s 1400 <remote-ipv6>
  # IPv6 更敏感；调整直到成功
```

2. **检查隧道并调整 MTU/MSS。**

- **IPsec（IKEv2）：**

```sh
$ ikectl show sa ; ikectl show flows
  # 确认 SA/flow 已建立
# tcpdump -ni enc0 host <peer-inside>
  # 解密后的明文流量
```

缓解时先加保守的 scrub，再细化：

```
## pf.conf 摘录 — 临时 MSS 钳制
scrub in on egress max-mss 1452
```

- **WireGuard：**

```sh
$ ifconfig wg0
  # 查看 wgpeer 统计；注意当前 MTU
# ifconfig wg0 mtu 1380
  # 降低 MTU 以容纳开销
```

- **gif/gre：**

```sh
$ ifconfig gif0 ; ifconfig gre0
  # 检查已配置的 MTU
# ifconfig gif0 mtu 1400
  # 降低并重新测试；按路径测量值调整
```

3. **确认 ICMP 可见性。**

```sh
# tcpdump -ni egress icmp or icmp6
  # 查找 "Fragmentation needed"（v4）或 "Packet Too Big"（v6）
```

---

## 剧本 4 — IPv6 ND 与 RA 故障（主机获取不到 IPv6）

**症状。** 客户端缺少全局 IPv6 地址，仅有链路本地地址，或丢失默认路由。出现重复地址检测（DAD）失败。

**可能原因。** RA 未在正确接口上发送、ICMPv6 被阻断、多台路由器同时通告，或 ND 缓存过期。

### 步骤

1. **检查路由器侧（RA 发送方）。**

```sh
# rcctl check rad
  # RA 守护进程是否运行？
$ ifconfig em1 | grep inet6
  # 路由器在该 /64 上是否有在链地址
# tcpdump -ni em1 icmp6
  # 观察该网段上的路由通告
```

2. **在 PF 中放行必需的 ICMPv6 并重新加载。**

```pf
## pf.conf 摘录 — 确保放行 ICMPv6
pass inet6 proto icmp6 keep state
```

```sh
# pfctl -f /etc/pf.conf
```

3. **从主机侧检查（客户端）。**

```sh
$ ndp -an | head
  # 应出现路由器条目且 MAC 可达
# rcctl check slaacd
  # 自动配置客户端是否运行？
$ route -n show -inet6 | egrep '::/0|2001:'
  # 默认路由经由路由器；存在全局地址
```

4. **处理重复地址或流氓 RA。**

```sh
$ ndp -an | egrep -i 'incomplete|failed'
  # 识别异常条目
# ndp -d <IPv6-or-mac>
  # 清除有问题的缓存条目（通过 ND 重新填充）
```

若两台路由器无意中通告同一 /64，先在退出的路由器上禁用 RA，再撤回地址。

---

## 剧本 5 — QoS 验证（PF 队列）

**症状。** 交互式流量在负载下受影响；队列计数器未按预期变化。

**可能原因。** 分类方向不匹配、队列名称错误，或父队列允许无限制借用。

### 步骤

1. **确认队列树与计数器。**

```sh
$ pfctl -vvsq
  # 队列树、带宽/min/max、每队列的包数/字节数
```

2. **验证分类规则与命中数。**

```sh
$ pfctl -vvsr | egrep 'set queue|set prio'
  # 确认预期规则存在且顺序正确
# tcpdump -ni egress port 22 or port 443
  # 产生流量并与队列计数器关联
```

3. **在负载下演练并观察。**

```sh
$ scp largefile user@remote:/dev/null &
  # 制造批量负载
$ ssh user@remote
  # 在观察 'pfctl -vvsq' 的同时验证按键延迟
```

4. **调整并重新测试。**

- 降低 default/bulk 队列的 `bandwidth` 或设置 `max`。
- 采用双队列赋值，例如 `set queue (web_bulk, web_prio)`，优先处理 ACK。

---

## 验证

- **可重复测试。** 保留一段简短脚本，在改动前后执行 ping、curl 已知端点，并输出一条 syslog 标记。
- **改动前后计数器。** 每一步前后记录 `pfctl -vvsr`、`pfctl -vvsq` 与 `pfctl -s state | wc -l`。
- **抓包。** 故障期间保存带时间戳的 `tcpdump` 记录，与 PF 快照一并留存，便于后续分析。

```sh
$ date && pfctl -vvsr | wc -l && pfctl -vvsq | wc -l
  # 带时间戳的规则/队列计数快速校验
# tcpdump -ni egress -w /root/ts/egress.pcap host <peer> and port <svc>
  # 抓包以供离线审查
```

## 故障排查

- **改动未生效。** 确认已重新加载 PF（`pfctl -f`），并在路径选择必须改变时清除过期状态。
- **数据路径与抓包不符。** 确认在正确的接口上抓包（物理口 vs. VLAN vs. 隧道），且 `set skip on` 未让流量绕过 PF。
- **HA "能用"但出现丢包。** 确认两节点规则集与 NAT 完全一致；状态会复制，规则不会。
- **MTU 仍不一致。** 检查中间链路（PPPoE、VPN 或 VLAN 堆叠）。若上游过滤了 ICMP 控制报文，保留 MSS 钳制。

## 参见

- [网络](../../install-and-configure/networking.md)
- [OpenBGBD](../services/networking-services/openbgpd.md)
- 相关：[**高可用与状态复制**](high-availability.md)
- 相关：[**多 WAN 与策略路由**](multi-wan-policy-routing.md)
- 相关：[**VPN 与加密隧道**](vpn-and-crypto-tunneling.md)
- 相关：[**大规模 IPv6**](ipv6-at-scale.md)
- 相关：[**QoS 与流量整形**](qos-traffic-shaping.md)
