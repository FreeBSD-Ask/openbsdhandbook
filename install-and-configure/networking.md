# 网络

## 概述

OpenBSD 提供灵活而安全的框架来配置与管理网络接口。本章讲解如何配置有线与无线网络、IPv4 与 IPv6 地址、DNS 解析、路由、网桥与 trunk，并涵盖诊断工具与安全实践。所有配置都通过 OpenBSD 原生的文件与工具完成，例如 `ifconfig(8)`、`hostname.if(5)`、`dhclient(8)` 与 `pf(4)`。

## 网络接口基础

每个网络接口按驱动名加单元号命名，例如首个 Intel 千兆以太网接口为 `em0`，Intel 无线设备为 `iwn0`。显示所有接口（包括未激活的接口）：

```sh
# ifconfig -a
```

查看指定接口的状态：

```sh
# ifconfig em0
```

根据检测到的硬件列出可用网络接口，可查看系统启动信息：

```sh
# dmesg | grep -E '^[a-z]+[0-9]+.*(Ethernet|wireless|network)'
```

部分无线设备的固件未包含在基本系统中。用 `fw_update(8)` 安装所需固件：

```
# fw_update
```

## 配置有线接口

网络接口配置通过 **/etc/hostname.if** 文件持久化，其中 `if` 对应接口名（例如 `em0`）。

使用 DHCP：

```sh
# echo dhcp > /etc/hostname.em0
```

配置静态 IP 地址：

```sh
# cat /etc/hostname.em0
inet 192.168.1.100 255.255.255.0 192.168.1.1
```

网关地址必须在所配置的子网内可达。

手动启用接口：

```sh
# ifconfig em0 up
```

应用配置改动：

```
# sh /etc/netstart em0
```

这些命令需要 root 权限。

### 配置接口别名

**接口别名**允许为一个网络接口分配多个 IP 地址，在许多场景下有用，包括：

- 让多个服务绑定到不同 IP
- 运行虚拟主机
- 出于组织或策略目的分离流量

别名可与主地址处于同一子网，也可处于完全不同的子网。别名处于**同一子网**时，子网掩码必须设为 `255.255.255.255`，以免出现意外的路由行为。别名属于**不同子网**时，必须指定该子网的正确子网掩码，且多数情况下需要显式添加路由才能让地址可达。

接口别名在接口的启动文件 **/etc/hostname.if** 中配置，其中 *if* 是接口名（例如 `em0`、`re0` 或 `iwn0`）。这些配置在启动时自动应用，或手动用 `sh /etc/netstart` 应用。

#### 示例：同一子网内的多个别名

下例中，系统主 IPv4 地址为 **10.0.0.2/24**，另有两个同子网别名：

文件：**/etc/hostname.re0**

```
inet 10.0.0.2 255.255.255.0
inet alias 10.0.0.3 255.255.255.255
inet alias 10.0.0.4 255.255.255.255
```

立即生效：

```sh
# sh /etc/netstart re0
```

#### 示例：不同子网的别名

若要从不同网段添加别名，子网掩码必须与该网段匹配。例如，在 **192.168.100.0/24** 子网中添加别名：

```
inet 10.0.0.2 255.255.255.0
inet alias 192.168.100.10 255.255.255.0
```

此时 OpenBSD 不会自动为新子网添加路由。需要时，必须在 **/etc/mygate** 中定义静态路由，或用 `route(8)` 设置。

#### 查看别名

默认情况下，`ifconfig(8)` 只显示每个接口的主地址。要查看所有已配置地址（包括别名），使用 `-A` 选项：

```sh
# ifconfig -A
```

此命令会显示每个接口的所有 IPv4 与 IPv6 地址，包括别名。

## 无线网络

连接开放网络：

```sh
# echo "nwid mynetwork" > /etc/hostname.iwn0
# sh /etc/netstart iwn0
```

连接 WPA2 加密网络：

```sh
# echo 'nwid mynetwork wpakey "mypassword"' > /etc/hostname.iwn0
```

`wpakey` 含特殊字符时需加引号。

连接多个接入点或配置其他参数：

```
join mynetwork wpakey mypassword
dhcp
```

扫描可用网络：

```sh
# ifconfig iwn0 up
# ifconfig iwn0 scan
```

WPA3 支持仅限 `iwm(4)` 与 `athn(4)` 等驱动在兼容硬件上使用。详见相关驱动手册页。

## 主机名与 DNS 解析

在 **/etc/myname** 中设置系统主机名：

```
myhostname.example.net
```

在 **/etc/resolv.conf** 中配置 DNS 服务器：

```
search example.net
nameserver 9.9.9.9
nameserver 2620:fe::fe
```

指定 IPv6 名称服务器时，确保具备 IPv6 连通性。

本地解析主机名：

```sh
# cat /etc/hosts
127.0.0.1    localhost
192.168.1.1  gw.example.net gw
```

默认情况下，`dhclient(8)` 会覆盖 **/etc/resolv.conf**。要阻止覆盖：

```
# chflags schg /etc/resolv.conf
```

**警告：** 使用 `schg` 会阻止动态更新，不适合移动或动态环境。可改用 `dhclient.conf(5)` 自定义。

## 路由与默认网关

查看路由表：

```
# netstat -rn
# netstat -rn -f inet6
```

添加默认路由：

```
# route add default 192.168.1.1
```

删除默认路由：

```
# route delete default
```

静态配置下，将默认网关写入 **/etc/hostname.if** 以持久化。更复杂的路由：

```
!/sbin/route add -inet6 default 2001:db8::1
```

`!` 前缀表示在接口配置完成后执行该命令。

## IPv6 配置

通过 SLAAC 使用自动 IPv6 地址：

```sh
# echo 'inet6 autoconf' > /etc/hostname.em0
```

启用隐私扩展（临时地址）：

```
inet6 autoconfprivacy
```

禁用 SLAAC：

```
inet6 -autoconf
```

静态 IPv6 配置：

```
inet6 2001:db8::100 64
!/sbin/route add -inet6 default 2001:db8::1
```

测试 IPv6 连通性：

```
# ping6 -c 3 openbsd.org
# traceroute6 openbsd.org
```

通过链路本地地址发现本地主机：

```
# ping6 ff02::1%iwn0
```

## 网桥与接口聚合

创建网桥：

```sh
# cat /etc/hostname.bridge0
add em0
add em1
up
```

配置成员接口：

```sh
# cat /etc/hostname.em0
up media 1000baseT mediaopt full-duplex

# cat /etc/hostname.em1
up media 1000baseT mediaopt full-duplex
```

创建链路聚合（LACP）trunk：

```sh
# cat /etc/hostname.trunk0
trunkproto lacp trunkport em0 trunkport em1
inet 192.168.1.2 255.255.255.0 192.168.1.1
```

**注意：** 交换机必须支持 LACP 或生成树协议。查看支持的介质类型：

```sh
# ifconfig em0 media
```

## 网络诊断

测试基本连通性：

```
# ping 1.1.1.1
# ping openbsd.org
```

测试 DNS：

```
# host openbsd.org
```

查看 ARP 表：

```
# arp -a
```

追踪网络路径：

```
# traceroute openbsd.org
```

查看无线信号强度：

```sh
# ifconfig iwn0
```

查看可用无线网络：

```sh
# ifconfig iwn0 scan
```

发送 IPv6 多播 ping：

```
# ping6 ff02::1%iwn0
```

抓包：

```
# tcpdump -i em0
```

`tcpdump` 需要 root 权限。

## 安全的网络实践

保持固件最新：

```
# fw_update
```

避免连接不安全的网络。使用强 WPA2 或 WPA3 口令。

启用基本防火墙：

```sh
# cat /etc/pf.conf
set block-policy drop
block all
pass out all keep state
pass in on em0 proto tcp to port 22 keep state
```

启用并加载防火墙：

```sh
# rcctl enable pf
# pfctl -f /etc/pf.conf
# pfctl -e
```

查看被阻止的流量：

```
# tcpdump -n -e -ttt -i pflog0
```

## VLAN 配置

配置 VLAN 接口：

```sh
# cat /etc/hostname.vlan0
vnetid 10 parent em0
inet 192.168.10.100 255.255.255.0 192.168.10.1
```

确保连接到 `em0` 的交换机端口配置为 trunk，且支持 VLAN ID 10。

## 本地 DNS 缓存

用 `unbound(8)` 启用本地 DNS 缓存：

```sh
# rcctl enable unbound
# rcctl start unbound
```

将 **/etc/resolv.conf** 中的名称服务器设为 **127.0.0.1**。

## 性能调优

设置最大传输单元（MTU）：

```sh
# ifconfig em0 mtu 9000
```

禁用 TCP 分段卸载（TSO）等功能：

```sh
# ifconfig em0 -tso
```

其他可调选项参见 `ifconfig(8)`。
