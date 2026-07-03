# QoS 与流量整形

## 摘要

本章介绍如何使用 PF 队列对流量进行分类与整形，以保护延迟敏感的流并管理批量传输。队列配置与分配定义在 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
中，运行时由 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
管理。实时查看使用 [systat(1)](https://man.openbsdhandbook.com/systat.1/)
，定向抓包使用 [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
。这些模式可为交互式协议提供可预测的延迟，同时保持链路利用率。

## 设计考量

- **整形出口而非入口。** 整形控制的是本端发送的流量。无法直接减缓已从互联网抵达的入站数据包；优先采用出口整形与 ACK 优先来影响远端发送方。
- **链路速率的准确性。** 为 WAN 接口指定切合实际的带宽（运营商速率减去开销）。若根队列允许借用至物理线速，默认队列可能饱和链路。
- **简单类别为佳。** 起步先用三类：*交互/语音*、*默认*与*批量*。仅在测量显示存在争用时才扩展。
- **队列分配策略。** 在规则上用 `set queue`（常通过 `match`）将流量分配到队列。考虑采用双队列形式，以优先处理 TCP ACK 与 `lowdelay` 流量。
- **优先级与队列。** `set prio` 选择接口优先级队列（0–7），并映射到 [vlan(4)](https://man.openbsdhandbook.com/vlan.4/)
  的 VLAN PCP。其本身不做整形。仅在需要 L2 标记时，与队列配合谨慎使用。
- **先验证。** 启用计数器，用可复现的流测试，并观察负载下的队列利用率。保留上一份 `pf.conf` 以便回滚。

## 配置

下例对单条上行链路上的出站流量进行整形。优先保障 SSH 交互与 VoIP，使 Web 浏览保持响应，并降级批量传输。请将接口名、地址与速率调整为与你的环境一致。

### 假设

- `em0` —— WAN（合同速率 100 Mb/s）
- `em1` —— LAN（`10.10.10.0/24`）

### 队列与分类（单文件策略）

将下列内容放入 `/etc/pf.conf`。*QUEUEING* 与 `set queue` 语法定义见 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
。

```sh
## /etc/pf.conf — QoS with PF queues (example)

set skip on lo

# Interfaces
wan     = "em0"
lan     = "em1"
lan_net = "10.10.10.0/24"

# -------------------------
# Queue tree on the WAN
# -------------------------
# Root queue at or slightly below line rate; avoid over-commit until measured.
queue rootq on $wan bandwidth 95M max 100M

# Realistic classes. Borrowing is implicit unless max bounds them.
queue voice       parent rootq bandwidth 2M    min 1M
queue web         parent rootq bandwidth 40M
  queue web_bulk  parent web   bandwidth 35M
  queue web_prio  parent web   bandwidth 5M    min 2M
queue ssh         parent rootq bandwidth 5M
  queue ssh_bulk  parent ssh   bandwidth 3M
  queue ssh_prio  parent ssh   bandwidth 2M    min 1M
queue default     parent rootq bandwidth 48M   default
queue bulk        parent rootq bandwidth 10M   max 20M

# -------------------------
# Base policy (tighten to your model)
# -------------------------
block all

# Permit LAN to anywhere; classification follows in match rules below
pass in on $lan from $lan_net to any keep state

# Permit outbound from the firewall itself
pass out on $wan proto { tcp udp icmp } from ($wan) to any keep state

# -------------------------
# Classification (sticky via match rules)
# -------------------------

# VoIP (SIP + RTP range example)
match out on $wan proto udp from $lan_net to any port { 5060, 10000:20000 } \
    set queue voice

# SSH: favor interactivity by assigning TCP ACKs and lowdelay to the second queue
match out on $wan proto tcp from $lan_net to any port 22 \
    set queue (ssh_bulk, ssh_prio)

# HTTPS: keep browsing responsive by prioritizing ACKs/lowdelay
match out on $wan proto tcp from $lan_net to any port 443 \
    set queue (web_bulk, web_prio)

# Bulk examples: backup, torrents, rsync (adjust as needed)
match out on $wan proto { tcp udp } from $lan_net to any port { 873, 6881:6999 } \
    set queue bulk

# Default: everything else lands in the default queue
# (No explicit rule needed because 'default' is marked on the queue tree)
```

加载并确认：

```sh
# pfctl -f /etc/pf.conf
  # 原子化重载
# pfctl -vvsq
  # 检查队列树、速率与计数器
```

### 可选：使用 `set prio` 标记 VLAN PCP

若上游支持 802.1Q PCP，可在保留队列控制的同时，为语音分配更高的接口优先级。此举会标记 L2 帧，可能有助于企业级链路上的上游调度。

```pf
## Excerpt — add to classification if using VLAN PCP
match out on $wan proto udp from $lan_net to any port { 5060, 10000:20000 } \
    set prio 6 set queue voice
```

### 可选：对已发布服务的回包整形

当向互联网发布 LAN 服务时，对服务器发出的**出站**回包进行分类，使返回流量遵循 QoS 策略。

```pf
## Excerpt — shape egress from a LAN web server
web_svc = "10.10.10.50"

# If you also use rdr on the WAN elsewhere, keep that configuration unchanged.
# Prioritize ACKs for HTTPS responses leaving via the WAN.
match out on $wan proto tcp from $web_svc to any port 443 \
    set queue (web_bulk, web_prio)
```

## 验证

- **队列用量与限制**，使用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
  ：

```sh
$ pfctl -vvsq
  # 验证队列树、min/max/burst、当前/峰值速率与包计数器
$ pfctl -vvsr | egrep 'set queue|set prio'
  # 确认规则到队列的映射已按预期加载
```

- **实时视图**，使用 [systat(1)](https://man.openbsdhandbook.com/systat.1/)
  ：

```sh
$ systat pf
  # 负载下观察状态与规则计数器；通过 pfctl 关注队列计数器
```

- **链路层核查**，使用 [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
  ：

```sh
# tcpdump -ni em0 port 22 or port 443 or udp and portrange 10000-20000
  # 确认测试期间存在相应流，并与队列计数器对照
```

- **负载生成。** 使用受控传输（例如用 `scp`/`rsync` 模拟批量、用 SSH 会话模拟交互），演示在批量流量活跃时键入仍保持响应。

## 故障排查

- **对延迟或吞吐量无影响。** 确保在实际承载流量的**出口**接口上创建了队列树，且 `set queue` 规则的方向匹配正确（通常为 `match out on $wan`）。用 `pfctl -vvsr` 与 `pfctl -vvsq` 验证。
- **"queue does not exist."** `set queue` 引用了出接口上不存在的叶子队列名。更正队列名，或确保该队列是合适父队列下的**叶子**。
- **默认队列饱和。** 若延迟敏感流量仍受影响，降低 `default` 的 `bandwidth` 或 `max`，或提高优先队列的 `min`。用可重复测试验证。
- **意外借用。** 若子队列增长超出预期，用 `max` 限制，或降低父队列带宽。
- **ACK 饥饿。** 未使用双队列形式时，TCP ACK 可能排在大包之后。对需要响应式 ACK 的类别使用 `set queue (bulk, prio)`。
- **二层标记被忽略。** `set prio` 映射到 VLAN PCP，但上游设备可能忽略。视作提示而非强制。
- **整形了错误的流量。** 用 `tcpdump` 验证选择器。常见错误是把客户端出站流量匹配到 `in on $wan`，而非 `out on $wan`。

## 另请参见

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- 相关：[**大规模 IPv6**](ipv6-at-scale.md)
- 相关：[**遥测、日志与流导出**](telemetry-logging-flow.md)
