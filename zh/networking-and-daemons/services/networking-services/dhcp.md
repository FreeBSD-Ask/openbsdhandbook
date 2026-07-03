# DHCP

## 概述

动态主机配置协议（DHCP）允许网络中的系统自动获取 IPv4 地址以及 DNS 解析器、默认网关等关联配置信息。OpenBSD 原生支持 IPv4 的 DHCP 客户端与服务器。IPv6 地址配置则使用 SLAAC（无状态地址自动配置）配合 `slaacd(8)`，而非 DHCPv6。

本章介绍如何在 OpenBSD 中配置 IPv4 的 DHCP 客户端与服务器，包括接口配置、租约管理与可选设置。IPv6 自动配置与路由通告请参阅 [SLAAC](slaacd.md) 一章。

## IP 配置工具概览

下表对比了 OpenBSD 基本系统与 ports 中可用于管理 IPv4 与 IPv6 地址配置的工具：

| 工具 | 说明 | IPv4 | IPv6 |
| --- | --- | --- | --- |
| `dhclient(8)` | DHCP 客户端 | ✔ | ✖（使用 `slaacd`） |
| `dhcpd(8)` | DHCP 服务器 | ✔ | ✖ |
| `dhclient.conf(5)` | DHCP 客户端配置文件 | ✔ | ✖ |
| `slaacd(8)` | SLAAC 客户端 | ✖ | ✔ |
| `rad(8)` | 路由通告守护进程 | ✖ | ✔ |
| `dhcp6s`（ports） | DHCPv6 服务器（经 ports） | ✖ | ✔ |
| `dhcp6c`（ports） | DHCPv6 客户端（经 ports） | ✖ | ✔ |

IPv6 支持方面，OpenBSD 优先采用 SLAAC 而非 DHCPv6。基本系统已包含 SLAAC 所需的全部工具，但不包含 DHCPv6。详细指引请参阅 [SLAAC 与 IPv6 自动配置](slaacd.md) 一章。

## 配置 DHCP 客户端

若网络接口配置中带有 `dhcp` 关键字，系统会在启动时自动调用 DHCP 客户端。

### 接口设置

要配置网络接口使用 DHCP，创建接口配置文件 **/etc/hostname.if**，内容为 `dhcp`。例如，在 `em0` 接口上启用 DHCP：

```sh
# echo "dhcp" > /etc/hostname.em0
# sh /etc/netstart em0
```

这样在启动或接口启用时，系统会执行 `dhclient(8)`。

### 租约文件管理

通过 DHCP 获取的租约存放在 **/var/db/dhclient.leases.ifname**，其中 `ifname` 为接口名。

查看当前租约信息：

```sh
$ cat /var/db/dhclient.leases.em0
```

手动释放租约：

```sh
# dhclient -r em0
```

强制 `dhclient` 请求新租约：

```sh
# dhclient em0
```

### 主机名配置

默认情况下，`dhclient` 会向 DHCP 服务器发送系统主机名。要禁用此行为，在 **/etc/dhclient.conf** 中添加：

```
send host-name "";
```

## DHCP 客户端定制

自定义 DHCP 选项可在 **/etc/dhclient.conf** 中指定。该文件支持按接口配置，并覆盖从 DHCP 服务器收到的值。

### 配置示例

覆盖 DNS 服务器与域名：

```
interface "em0" {
    supersede domain-name "example.net";
    supersede domain-name-servers 192.0.2.53, 192.0.2.54;
}
```

更改在下次对该接口运行 `dhclient` 时生效。

## 配置 DHCP 服务器

OpenBSD 内置 ISC DHCP 服务器，即 `dhcpd(8)`，为本地网络上的客户端提供动态地址分配。

### dhcpd.conf 示例

最小的 DHCP 服务器配置文件类似如下：

```
option domain-name "example.net";
option domain-name-servers 192.0.2.53;

subnet 192.0.2.0 netmask 255.255.255.0 {
    range 192.0.2.100 192.0.2.200;
    option routers 192.0.2.1;
}
```

将此配置保存为 **/etc/dhcpd.conf**。

### 启动与启用 DHCP 服务器

在指定接口（例如 `em1`）上启用并启动 DHCP 服务器：

```sh
# rcctl enable dhcpd
# rcctl set dhcpd flags em1
# rcctl start dhcpd
```

必须通过 `rcctl set dhcpd flags ...` 显式指定接口，守护进程才能运行。

## 日志与排障

DHCP 客户端与服务器的日志写入 **/var/log/messages**。实时观察 DHCP 活动：

```sh
# tail -f /var/log/messages
```

确保同一网段上没有其他 DHCP 服务器运行。包过滤规则也须放行 DHCP 流量（UDP 端口 67 与 68）。

服务器端诊断时，以前台详细模式启动 `dhcpd`：

```sh
# dhcpd -d -f em1
```
