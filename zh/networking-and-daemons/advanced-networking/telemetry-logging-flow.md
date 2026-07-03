# 遥测、日志与流导出

## 概述

本章提供 OpenBSD 防火墙与路由器的运维可观测性模式：通过 [pflog(4)](https://man.openbsdhandbook.com/pflog.4/) 与 [pflogd(8)](https://man.openbsdhandbook.com/pflogd.8/) 进行包过滤日志；用 [pflow(4)](https://man.openbsdhandbook.com/pflow.4/) 向外部收集器导出流（NetFlow v5/v9，以及在你的发行版支持时使用 IPFIX）；用 [snmpd(8)](https://man.openbsdhandbook.com/snmpd.8/) 进行系统与服务遥测；用 [syslogd(8)](https://man.openbsdhandbook.com/syslogd.8/) 配合 [syslog.conf(5)](https://man.openbsdhandbook.com/syslog.conf.5/) 进行集中日志转发。验证依赖 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)、[tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/) 与 [systat(1)](https://man.openbsdhandbook.com/systat.1/)。这些模式用于为事件响应、容量规划与安全监控产生可操作的信号。

## 设计考量

- **时钟与标识。** 关联分析要求时间准确。运行 [ntpd(8)](https://man.openbsdhandbook.com/ntpd.8/)，并确保遥测的主机名唯一、IP 来源稳定。
- **范围与隐私。** 仅记录与导出你能存储并保护的内容。除非有明确理由，避免完整数据包载荷。聚合可见性优先采用采样或流级遥测。
- **职责分离。** 将防火墙/路由器视为生产者。重度分析、长期存储与仪表盘在外部系统上进行。
- **网络路径。** 将收集器置于专用管理网络或隔离 VPN 上。保护基于 UDP 的馈送（例如 NetFlow）免受丢失与伪造。
- **版本注意事项。** [pflow(4)](https://man.openbsdhandbook.com/pflow.4/) 支持 NetFlow v5/v9。IPFIX 的可用性与选项取决于发行版。启用特定模板前须在 OpenBSD 7.8 上确认能力。
- **轮转与留存。** 为 `/var` 合理分配容量，并在 newsyslog.conf(5) 中设置合适的轮转。将日志转发出主机；不要仅依赖本地磁盘。

## 配置

### 1) 包过滤日志（pflog）

在 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/) 中启用定向日志。规则上的 `log` 关键字向 `pflog0` 输出元数据，[pflogd(8)](https://man.openbsdhandbook.com/pflogd.8/) 将其作为轮转 pcap 存储于 `/var/log/pflog`。

```pf
## /etc/pf.conf — 带规则日志的摘录

set skip on lo

wan = "em0"
lan = "em1"
lan_net = "10.10.10.0/24"

block all

# 记录被阻断的 WAN 主动流量
block log on $wan

# 记录到防火墙的 SSH 管理访问尝试
pass  log on $wan proto tcp to ($wan) port 22 keep state

# 允许并记录从 LAN 到任意地址的 HTTPS（演示用；生产环境须收紧）
pass  log on $lan proto tcp from $lan_net to any port 443 keep state
```

应用策略并确保 pflogd 已启用：

```sh
# pfctl -f /etc/pf.conf
  # 原子重载 PF 规则
# rcctl enable pflogd
  # 开机时启动 pflog 捕获
# rcctl start pflogd
  # 立即启动；写入 /var/log/pflog
# tcpdump -n -e -ttt -i pflog0
  # PF 日志流的实时视图（Ctrl-C 停止）
```

> 用 `pfctl -vvsr` 查看哪些规则带 `log`，并检查每条规则的计数器。

### 2) 用 pflow(4) 导出流（NetFlow v5/v9，可选 IPFIX）

[pflow(4)](https://man.openbsdhandbook.com/pflow.4/) 通过 UDP 向远端收集器导出 PF 状态流量摘要。配置 `pflow0` 接口，指定稳定的**源地址**、收集器**目的**（IP:端口）与**协议**版本。

假定：

- 导出源：`192.0.2.10`（环回或管理地址）
- 收集器：`192.0.2.50:2055`（NetFlow）
- 协议：v9

```sh
# ifconfig pflow0 create
  # 创建导出器接口
# ifconfig pflow0 flowsrc 192.0.2.10 flowdst 192.0.2.50 2055 flowproto 9
  # 设置源、目的与 NetFlow 版本
# ifconfig pflow0 up
  # 激活导出器
# ifconfig pflow0
  # 验证参数
```

在 PF 中放行导出路径（按需替换 `em0`/地址）：

```sh
## /etc/pf.conf — 允许 pflow 导出到收集器

set skip on lo
wan = "em0"

pass out on $wan proto udp from 192.0.2.10 to 192.0.2.50 port 2055 keep state
```

重载并通过对收集器的抓包确认：

```sh
# pfctl -f /etc/pf.conf
# tcpdump -ni $wan host 192.0.2.50 and udp port 2055
  # 观察流导出包；同时在收集器侧验证
```

> 若收集器期望 IPFIX，在 OpenBSD 7.8 上检查 [pflow(4)](https://man.openbsdhandbook.com/pflow.4/) 的 IPFIX 支持与模板选项，然后相应切换 `flowproto`。

### 3) 用 snmpd(8) 进行系统遥测

[snmpd(8)](https://man.openbsdhandbook.com/snmpd.8/) 暴露标准 MIB（接口、系统、传感器）。将其绑定到管理接口并限制访问。具体模型（community 还是 SNMPv3）取决于你的安全策略；工具支持时推荐 SNMPv3。

最小示例：在 LAN 上监听，并允许来自 `10.10.10.0/24` 的只读查询。

```
## /etc/snmpd.conf — 最小只读服务

listen on 10.10.10.1

system contact "netops@example.com"
system location "DC1-R1"

read-only community "monitoring" source 10.10.10.0/24
```

```sh
# rcctl enable snmpd
# rcctl start snmpd
# pkill -0 snmpd && echo "snmpd running"
  # 基本存活检查
```

> SNMPv3 用户配置与 trap 接收器参见 [snmpd.conf(5)](https://man.openbsdhandbook.com/snmpd.conf.5/)。确保 PF 允许来自 NMS 的 UDP 161，以及任何 trap 出站（UDP 162）。

### 4) 用 syslogd(8) 进行集中日志转发

用 [syslog.conf(5)](https://man.openbsdhandbook.com/syslog.conf.5/) 将本地日志转发到远端收集器。UDP 转发在主机前加单个 `@`。

```
## /etc/syslog.conf — 将所有日志转发到收集器并保留本地文件

*.*     @192.0.2.60
# 在下方保留既有的本地文件规则（按需精简）
auth.info       /var/log/authlog
daemon.info     /var/log/daemon
kern.info       /var/log/kernlog
pf.info         /var/log/pflogdump
```

```sh
# rcctl reload syslogd
  # 重新读取配置；syslogd 发送到 192.0.2.60:514/udp
# logger -t test "syslog forwarding test $(date)"
  # 发送测试消息；验证其到达收集器
```

> 若收集器要求 TCP 或 TLS，在其前端放置日志中继，或查阅收集器的接入选项。保持 `/etc/newsyslog.conf` 的本地留存配置合理。

## 验证

- **PF 规则计数器与日志**，使用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/) 与实时 pflog：

```sh
$ pfctl -vvsr | egrep 'log\b'
  # 确认哪些规则记录日志，并观察计数器递增
$ tcpdump -n -e -ttt -i pflog0
  # 测试期间的实时数据包日志
$ tcpdump -nr /var/log/pflog | head
  # 检查轮转后的捕获文件
```

- **链路上的 pflow 导出**：

```sh
# tcpdump -ni em0 host 192.0.2.50 and udp port 2055
  # 确认 NetFlow/流导出包离开路由器
```

- **SNMP 可达性**（从 NMS 或测试主机）：

```sh
$ snmpwalk -v2c -c monitoring 10.10.10.1 sysUpTime.0
  # 基本存活；如适用则替换为 v3 命令
```

- **syslog 转发路径**：

```sh
$ logger -t probe "forward-test $(date)"
  # 发出消息并确认其到达 192.0.2.60
$ tail -n 5 /var/log/messages
  # 本地副本仍按 syslog.conf 可用
```

- **仪表盘**：部署期间使用 [systat(1)](https://man.openbsdhandbook.com/systat.1/) 的 `pf` 与 `ifstat` 模式，将排队、规则命中与接口速率与导出的遥测关联。

## 故障排查

- **无 pflog 记录。** 确保规则含 `log`、PF 已加载、[pflogd(8)](https://man.openbsdhandbook.com/pflogd.8/) 正在运行。用 `tcpdump -i pflog0` 绕过文件验证流。
- **收集器看不到 pflow。** 核对 `pflow0` 上的 `flowsrc/flowdst/flowproto`、PF 允许到收集器的 UDP、中间 ACL 放行。确认时间同步；部分收集器会丢弃偏差过大的时间戳。
- **pflow 丢失率高。** 流导出基于 UDP，尽力而为。减小采样窗口（如支持）、专用管理路径，或部署更近的收集器。确保突发时 CPU 未饱和。
- **SNMP 超时。** 确认 `listen on` 地址与 `source` 限制匹配 NMS。核对 PF 允许来自 NMS 的 UDP 161，且 community 或 v3 凭据正确。
- **Syslog 未转发。** 检查 `/etc/syslog.conf` 语法，重载 `syslogd`，并确认到收集器 UDP 514 的可达性。若收集器要求 TCP/TLS，使用其支持的接入方法。
- **/var 磁盘压力。** 提高 newsyslog.conf(5) 中的轮转频率，并将日志/流出主机。考虑排除嘈杂的 facility 或提高阈值。

## 参见

- [网络](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- 相关：[**QoS 与流量整形**](qos-traffic-shaping.md)
- 相关：[**大规模网络服务**](network-services-at-scale.md)
- 相关：[**故障排查手册**](troubleshooting-playbooks.md)
