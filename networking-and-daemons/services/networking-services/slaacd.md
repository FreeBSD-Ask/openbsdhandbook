# slaacd

## 概述

OpenBSD 原生支持 **无状态地址自动配置（SLAAC）**，详见 [RFC 4862](https://datatracker.ietf.org/doc/html/rfc4862)。SLAAC 允许 IPv6 主机根据本地路由器发送的路由通告（RA）自动配置地址。此机制无需集中式 DHCPv6 基础设施，是 OpenBSD 客户端 IPv6 配置的默认方式。

`slaacd(8)` 守护进程负责接收这些通告，并据此配置 IPv6 地址、路由与解析器设置。

在典型的 OpenBSD 网络中，`slaacd(8)` 与 [`rad(8)`](rad.md) 配合使用，后者在服务器或路由器一侧提供路由通告。

## OpenBSD 中的 DHCP 与自动配置组件

下表汇总 OpenBSD 上用于 DHCP 与自动配置的主要组件，涵盖 IPv4 与 IPv6 支持：

| 组件 | 角色 | IP 版本 | 是否在基本系统中 | 用途 |
| --- | --- | --- | --- | --- |
| `dhclient(8)` | DHCP 客户端 | IPv4 | ✔ | 从 DHCP 服务器获取 IPv4 地址与 DNS |
| `dhcpd(8)` | DHCP 服务器 | IPv4 | ✔ | 向客户端分配 IPv4 地址与配置 |
| `slaacd(8)` | SLAAC 客户端守护进程 | IPv6 | ✔ | 接收 RA 消息并配置接口 |
| [`rad(8)`](rad.md) | 路由通告守护进程 | IPv6 | ✔ | 广播 IPv6 前缀与路由信息 |
| `dhcp6s` | DHCPv6 服务器（来自 ports） | IPv6 | ✘ | 提供有状态 IPv6 配置（很少需要） |

对大多数 IPv6 网络而言，`slaacd(8)` 与 [`rad(8)`](rad.md) 已足够，无需第三方软件。

## 使用 slaacd(8) 配置 SLAAC

`slaacd(8)` 守护进程监听路由通告，并配置：

- IPv6 地址（使用通告的前缀）
- 默认网关
- DNS 解析器设置（若通过 RDNSS 通告）

### 在接口上启用 SLAAC

要在接口上启用 SLAAC，创建或编辑对应的 **/etc/hostname.if** 文件，内容为：

```
inet6 autoconf
```

例如，在 `em0` 上启用 SLAAC：

```sh
# echo "inet6 autoconf" > /etc/hostname.em0
# sh /etc/netstart em0
```

这样 `slaacd(8)` 会自动启动，并应用收到的 IPv6 配置。

### 运行时管理 slaacd

`slaacd` 虽会为已配置接口自动运行，但也可通过 `rcctl(8)` 显式管理：

```sh
# rcctl enable slaacd
# rcctl start slaacd
# rcctl check slaacd
# rcctl restart slaacd
```

查看日志：

```sh
# tail -f /var/log/messages
```

## 查看已分配的 IPv6 地址

配置成功后，可用以下命令查看 IPv6 地址：

```sh
$ ifconfig em0
```

示例输出：

```
inet6 fe80::1%em0 prefixlen 64 scopeid 0x1
inet6 2001:db8:abcd:1234::123 prefixlen 64 autoconf
```

`autoconf` 关键字表明该地址经由 SLAAC 分配。

## 通过 RDNSS 配置 DNS

若路由器使用 **递归 DNS 服务器通告（RDNSS）** 通告 DNS 服务器地址，`slaacd(8)` 会将其写入 **/etc/resolv.conf**（前提是该文件可写）。

要允许此行为，确保 **/etc/resolv.conf** 未受系统不可变标志保护：

```sh
# chflags noschg /etc/resolv.conf
```

否则 SLAAC 仍会完成，但 DNS 解析可能无法正常工作。

## 通告 IPv6 前缀

SLAAC 要成功，网络上必须存在路由通告。这些通告通常由 `rad(8)` 提供，须运行在默认网关或路由器上。

完整配置细节请参阅 [rad(8)](rad.md) 一章。

## 在接口上禁用 SLAAC

要禁用 SLAAC，从接口配置文件中移除 `inet6 autoconf` 指令，并重启接口：

```sh
# rm /etc/hostname.em0
# echo "inet6 2001:db8::123 255.255.255.0" > /etc/hostname.em0
# sh /etc/netstart em0
```

或者将文件留空，或省略 `autoconf` 关键字，以阻止自动 IPv6 配置。

## 排障

### 确认接口地址

```sh
$ ifconfig em0
```

检查带 `autoconf` 标记的 `inet6` 地址。

### 监控路由通告

```sh
# tcpdump -i em0 icmp6
```

此命令将显示收到的 RA 消息。

### 验证 DNS 配置

```sh
$ cat /etc/resolv.conf
$ host openbsd.org
```

若 **resolv.conf** 未反映预期设置，确认 RDNSS 是否正在通告，以及该文件是否受保护。
