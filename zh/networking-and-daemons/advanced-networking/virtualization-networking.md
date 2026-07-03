# 虚拟化与主机网络

## 概述

本章给出 OpenBSD 虚拟化的标准网络模式，涉及 [vmd(8)](https://man.openbsdhandbook.com/vmd.8/) 与 [vmctl(8)](https://man.openbsdhandbook.com/vmctl.8/)：用 [bridge(4)](https://man.openbsdhandbook.com/bridge.4/) 实现二层桥接、用 [vether(4)](https://man.openbsdhandbook.com/vether.4/) 实现路由分段、用 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/) 锚点实现按 VM 分段。虚拟网卡使用 [tap(4)](https://man.openbsdhandbook.com/tap.4/)。Hypervisor 配置位于 [vm.conf(5)](https://man.openbsdhandbook.com/vm.conf.5/)。这些模式用于将客户机接入既有 VLAN、提供带 NAT 的隔离路由网络，以及按 VM 实施最小权限网络策略。

## 设计考量

- **接入模型。**
  - **桥接：** VM 出现在既有二层网段上。DHCP 或广播发现须到达客户机时使用。
  - **路由（vether）：** VM 位于虚拟三层接口之后；由主机路由或 NAT。用于隔离与严格控制。
- **故障域。** 避免不必要地扩展大二层层域。规模部署时优先采用 [vether(4)](https://man.openbsdhandbook.com/vether.4/) 的路由分段。
- **VLAN。** 桥接到 trunk 时，将承载 VLAN 标签的**父**接口加入网桥，使标签透传；若只需单个 VLAN，则改将其 [vlan(4)](https://man.openbsdhandbook.com/vlan.4/) 接口加入网桥。
- **MAC 计数。** 上游交换机可能限制学习到的 MAC 数。通过一个物理端口桥接大量 VM 可能超出限制。与交换策略协调，或改用路由接入。
- **安全姿态。** 将每台 VM 视为不可信。用 PF 锚点为每台 VM 应用显式策略，并对同主机 VM 间的流量默认拒绝。
- **运维。** 将 VM 镜像与网络定义纳入版本控制。初次部署时启用详细日志。采用一致的命名方案（例如 `switch "lan"`、`switch "dmz"`）。

## 配置

主机具有 `em0`（WAN/上行）与 `em1`（LAN 或 trunk）。示例展示：

1. 将 VM 桥接到既有 LAN（或 trunk）。
2. 用 `vether0` 构建带可选 NAT 的隔离路由网络。
3. 用 PF 锚点实施按 VM 策略。

### 1) Hypervisor 基础设置

为客户机接入创建网桥、定义 vmd switch，并启用 hypervisor。

```sh
# cat > /etc/hostname.bridge0 <<'EOF'
add em1
up
EOF
  # 网桥将 em1 接向 LAN/trunk；VM 将通过 tap(4) 加入

# sh /etc/netstart bridge0
  # 激活网桥

# rcctl enable vmd
  # 开机时启动 vmd(8)
# rcctl start vmd
  # 立即启动

# install -d -o root -g wheel /var/vm
  # 确保 VM 磁盘有存放位置
```

在 [vm.conf(5)](https://man.openbsdhandbook.com/vm.conf.5/) 中定义 vmd switch 与一台简单 VM。switch 将 VM 绑定到 `bridge0`。

```
## /etc/vm.conf — 示例 switch 与客户机
switch "lan" {
    interface bridge0
}

vm "web01" {
    memory 1024M
    disk "/var/vm/web01.img"
    interface { switch "lan" }
}
```

创建磁盘并启动 VM（此处省略安装介质；按你的工作流调整）。

```sh
# vmctl create /var/vm/web01.img -s 20G
  # 创建稀疏磁盘
# vmctl start web01
  # 启动 VM；vmd(8) 将一个 tap(4) 附加到 bridge0
# vmctl status
  # 列出 VM 与已分配的 tap 接口
```

### 2) 既有 LAN（或 VLAN trunk）上的桥接客户机

`bridge0` 承载 `em1`、vmd switch 指向 `bridge0` 时，客户机出现在物理 LAN 上（若 `em1` 为 trunk，则出现在所有穿越 `em1` 的 VLAN 上）。若须向客户机暴露**单个** VLAN，则改将该 VLAN 接口加入网桥：

```
## /etc/hostname.vlan10 — 示例单 VLAN 网桥成员
vlandev em1
vlan 10
up
```

```sh
# ifconfig bridge0 delete em1
  # 仅需 VLAN 10 时从网桥移除父接口
# ifconfig bridge0 add vlan10
  # 仅加入 VLAN 子接口；客户机只见 VLAN 10
# ifconfig bridge0
  # 验证成员
```

由既有基础设施提供 DHCP 或静态地址。若要在本地为桥接客户机提供 DHCP，按你的策略在 `bridge0` 或 `em1` 上配置 [dhcpd(8)](https://man.openbsdhandbook.com/dhcpd.8/)（参见服务章节）。

### 3) 用 vether(4) 与可选 NAT 实现路由客户机

在 `vether0` 之后为客户机创建隔离的三层网段。主机在 `vether0` 与上行链路间路由。

```
## /etc/hostname.vether0 — 路由客户机网段
inet 10.50.0.1 255.255.255.0
up
```

为路由网段创建专用网桥，并动态将客户机 tap 加入（客户机启动时 vmd 会添加 tap）。

```
## /etc/hostname.bridge1 — 路由网段的网桥
add vether0
up
```

将第二个 vmd switch 指向 `bridge1`，并将 VM 附加其上。

```
## /etc/vm.conf — 摘录，添加路由 switch 与 VM
switch "isolated" {
    interface bridge1
}

vm "db01" {
    memory 2048M
    disk "/var/vm/db01.img"
    interface { switch "isolated" }
}
```

启用转发，并按需用 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/) 通过 NAT 到上行链路。运行时用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/) 管理 PF。

```pf
## /etc/pf.conf — 带 NAT 的最小路由网段

set skip on lo

wan      = "em0"
isol_net = "10.50.0.0/24"

# 将隔离客户机 NAT 到上行链路
match out on $wan from $isol_net to any nat-to ($wan)

block all
pass in on vether0 from $isol_net to any keep state
pass out on $wan proto { tcp udp icmp } from ($wan) to any keep state
```

```sh
# printf '%s\n' 'net.inet.ip.forwarding=1' >> /etc/sysctl.conf
  # 持久化 IPv4 转发
# sysctl net.inet.ip.forwarding=1
  # 立即应用
# pfctl -f /etc/pf.conf
  # 加载 PF 策略
```

在主机上用绑定到 `vether0` 的 [dhcpd(8)](https://man.openbsdhandbook.com/dhcpd.8/) 为路由客户机提供地址（参见服务章节），或在 `10.50.0.0/24` 中配置静态地址。

### 4) 用 PF 锚点实现微分段（按 VM 策略）

用 PF 锚点隔离客户机，并使按 VM 的规则保持小而可审计。

主策略：

```pf
## /etc/pf.conf — 锚点框架（摘录）

set skip on lo

# 从专用目录加载所有按 VM 的策略
anchor "vm/*"
load anchor "vm/*" from "/etc/pf.vm/*.conf"

# 默认姿态
block all
pass out on egress proto { tcp udp icmp } keep state
```

按 VM 的策略文件：

```pf
## /etc/pf.vm/web01.conf — 允许从任意地址到 web01 的 HTTP/HTTPS，仅允许管理子网的 SSH
# 将规则附加到 VM 的 tap 接口（替换为实际 tapN）
intf = "tap3"

# 入站公共服务
pass in on $intf proto tcp to ($intf) port { 80, 443 } keep state

# 仅管理 LAN 的运维操作
pass in on $intf proto tcp from 10.10.0.0/24 to ($intf) port 22 keep state
```
```sh
## /etc/pf.vm/db01.conf — 仅允许 web01 访问 MySQL；其余入站全部拒绝
intf = "tap4"

# 仅 Web 层可访问数据库
pass in on $intf proto tcp from 10.50.0.20 to ($intf) port 3306 keep state
```

加载并验证：

```sh
# pfctl -f /etc/pf.conf
  # 重载主策略与锚点
# pfctl -a 'vm/*' -sr
  # 显示 vm/* 锚点内的规则
```

> 维护 VM 名称到 tap 接口的映射（来自 `vmctl status`）并在锚点文件中体现，或在 VM 部署时用模板生成锚点。

## 验证

- **Hypervisor 状态**，使用 [vmctl(8)](https://man.openbsdhandbook.com/vmctl.8/)：

```sh
$ vmctl status
  # VM 名称、ID 与 tap 接口
```

- **桥接与成员**，使用 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)：

```sh
$ ifconfig bridge0
  # 期望 em1 与一个或多个 tap(4) 成员（桥接客户机）
$ ifconfig bridge1
  # 期望 vether0 与客户机 tap（路由网段）
```

- **路由与 NAT**：

```sh
$ netstat -rn
  # 验证经 vether0 的 10.50.0.0/24 直连路由
$ pfctl -vvsr | egrep 'match out .* nat-to|\banchor vm/'
  # 确认 NAT 规则与锚点加载
```

- **在线检查**，使用 [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)：

```sh
# tcpdump -ni vether0
  # 路由网段内的客户机流量
# tcpdump -ni em1 vlan 10
  # 桥接单 VLAN 接口时的带标签帧
```

## 故障排查

- **桥接网络上客户机无法获取 DHCP。** 确认 `bridge0` 中接口正确（trunk 用 `em1`，单 VLAN 用对应 `vlan(4)`）。确保上游 DHCP 服务器可达该网段，且 PF 未阻断 BOOTP。
- **路由客户机无连通性。** 核对 `net.inet.ip.forwarding=1`、`vether0` 地址正确、PF 已加载 NAT `match` 规则。在 `$wan` 与 `vether0` 上用 `tcpdump` 追踪数据包。
- **锚点未生效。** 确认主策略中存在 `anchor` 与 `load anchor` 行，且按 VM 的文件可编译。用 `pfctl -a 'vm/*' -sr` 检查。
- **VM 接入错误网络。** 检查 `vm.conf` 中的 `switch` 段与 VM 定义中的 `interface { switch "…" }` 块。更改接入后须重载 vmd 或重启 VM。
- **上游交换机 MAC 数超限。** 减少该端口的桥接客户机，或将工作负载迁移到路由 `vether0` 模式。
- **VLAN 标签未保留。** 传递多个 VLAN 时，桥接**父**物理接口（例如 `em1`）而非单个 `vlan(4)` 子接口。仅暴露一个 VLAN 时，只桥接该 `vlan(4)` 接口。

## 参见

- [网络](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- 相关：[**高可用与状态复制**](high-availability.md)
- 相关：[**大规模网络服务**](network-services-at-scale.md)
