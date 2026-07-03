# PF

OpenBSD 手册的 PF 部分详细介绍如何配置和管理 Packet Filter 防火墙。PF 通过过滤规则、NAT、重定向及多种高级特性控制网络流量。

本部分包含：

- [pfctl 速查表](cheat_sheet.md)
  常用 `pfctl` 命令与选项的速查表。
- [锚点](anchors.md)
  使用锚点模块化 PF 规则集，并动态加载规则。
- [过滤](filter.md)
  编写过滤规则，控制进出流量。
- [转发](forwarding.md)
  在 PF 中启用并配置数据包转发。
- [列表与宏](lists-and-macros.md)
  对地址分组并定义宏，简化规则管理。
- [负载均衡](load-balancing.md)
  将流量分散到多台服务器或多条网络路径。
- [日志](logging.md)
  捕获并分析 PF 日志数据。
- [NAT](nat.md)
  配置网络地址转换，让专有网络接入互联网。
- [选项](options.md)
  配置影响规则处理与性能的全局 PF 选项。
- [策略](policies.md)
  为数据包处理设置默认规则策略。
- [简写](shortcuts.md)
  高效编写 PF 规则的技巧与简写。
- [表](tables.md)
  使用表高效管理大批地址。
