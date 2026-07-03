# 高级网络

## 摘要

本部分介绍构建与运维复杂 OpenBSD 网络所需的高级、可投产模式。重点涵盖高可用、多宿主、隧道、大规模 IPv6、服务质量、MPLS、大规模网络服务、虚拟化感知设计、L2/L3 架构、遥测、加固、参考拓扑与故障排查手册。每章提供简洁的设计指引、带注释的配置、验证步骤，以及面向运维人员的故障排查流程。

## 与“Networking”及“OpenBGPD”的关系

**Networking** 部分介绍基础接口、路由与包过滤概念，适用于多数单主机或小型部署。**高级网络** 在此基础上扩展，应对更大规模环境下的冗余、可扩展性与运维安全。

**OpenBGPD** 部分专注于外部与内部 BGP 策略、对等与路由分发。本部分在合适之处将 BGP 作为控制平面输入（例如任播服务或 MPLS VPN）加以引用，但详细的 BGP 策略与会话管理请参阅 `/openbgpd` 下专门的 **OpenBGPD** 章节。

## 目录

下列章节按复杂度递增排序。每项链接到对应章节，并附一句话摘要。

- **[高可用与状态复制](high-availability.md)**——使用 CARP、pfsync、relayd 与事件驱动故障切换构建冗余网关与防火墙。
- **[多 WAN 与策略路由](multi-wan-policy-routing.md)**——多运营商边缘的 route-to、reply-to、监控与入站注意事项。
- **[VPN 与加密隧道](vpn-and-crypto-tunneling.md)**——基于 iked 的 IKEv2、WireGuard 风格隧道、NAT 穿透与性能调优。
- **[经典轻量隧道](classic-tunnels.md)**——用于务实叠加与迁移的 gif、gre 与 etherip。
- **[大规模 IPv6](ipv6-at-scale.md)**——路由通告、ND、前缀委派、NPTv6 与大段 IPv6 运维。
- **[QoS 与流量整形](qos-traffic-shaping.md)**——使用 PF 对延迟敏感与批量流量进行分类与排队。
- **[MPLS 与标签分发](mpls.md)**——mpls(4) 数据平面、通过 ldpd 的 LDP、PE 接口（mpe）与 L2 VPLS 伪线。
- **[大规模网络服务](network-services-at-scale.md)**——基于 nsd 的权威 DNS、基于 unbound 的可验证递归、DHCP 角色与时间服务。
- **[虚拟化与主机网络](virtualization-networking.md)**——vmd、tap/vether、桥接/VLAN 与基于 PF 锚点的微分段。
- **[大规模 L2 与 L3 设计](l2-l3-design.md)**——VLAN 策略、路由接入、首跳冗余与任播服务部署。
- **[遥测、日志与流导出](telemetry-logging-flow.md)**——pflog 分析、pflow（NetFlow/IPFIX）、snmpd 与时序数据集成。
- **[加固与运维安全](hardening-operations.md)**——反欺骗、RPF、密钥管理与安全发布流程。
- **[参考架构](reference-architectures.md)**——端到端设计：冗余互联网边缘、园区/分支与 POP 模式。
- **[故障排查手册](troubleshooting-playbooks.md)**——诊断 HA 抖动、非对称路径、隧道 MTU 问题、IPv6 ND 与 QoS 验证。

## 如何使用本部分

1. 首次阅读时按顺序通读各章，建立通用模式与术语认知。
2. 实施时根据目标（例如 HA、IPv6 或 QoS）直接跳转到对应章节。
3. 修改配置前先应用 **设计考量** 中的原则。
4. 部署后立即使用 **验证** 步骤确认行为。
5. 发布与事故响应期间，将 **故障排查** 手册放在手边。

## 下一步

进入 **[高可用与状态复制](high-availability.md)**，随后按顺序继续阅读各章，完成完整浏览。也可直接跳转到与当前任务匹配的章节。
