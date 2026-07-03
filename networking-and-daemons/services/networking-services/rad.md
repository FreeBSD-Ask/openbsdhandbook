# rad

## 概述

OpenBSD 上的 IPv6 地址配置通常使用 **无状态地址自动配置（SLAAC）**，而非 DHCPv6。`rad(8)` 守护进程提供 **路由通告（RA）**，[`slaacd(8)`](slaacd.md) 等客户端据此自行配置 IPv6 地址、路由以及可选的 DNS 解析器信息，无需中央 DHCPv6 服务器。

旧版 OpenBSD 使用 `rtadvd(8)` 承担此角色。OpenBSD 6.4 起 `rtadvd(8)` 为 `rad(8)` 取代，因此新部署的 OpenBSD 系统应使用 `rad(8)` 与 **/etc/rad.conf**。

IPv4 通常使用 DHCP 进行地址配置，而 OpenBSD 基本系统有意未提供 DHCPv6 服务器功能。轻量级 DHCPv6 服务器可通过 ports（`dhcp6s`）获取，但实际中很少需要。

## OpenBSD 中的 DHCP 与自动配置组件

下表汇总 OpenBSD 上可用的主要 DHCP 与 IPv6 自动配置组件，涵盖 IPv4 与 IPv6。

| 组件 | 角色 | IP 版本 | 是否在基本系统中 | 用途 |
| --- | --- | --- | --- | --- |
| `dhclient(8)` | DHCP 客户端 | IPv4 | ✔ | 从 DHCP 服务器获取 IPv4 地址与 DNS |
| `dhcpd(8)` | DHCP 服务器 | IPv4 | ✔ | 向客户端分配 IPv4 地址与配置 |
| [`slaacd(8)`](slaacd.md) | SLAAC 客户端守护进程 | IPv6 | ✔ | 接收 RA 消息并配置接口 |
| `rad(8)` | 路由通告守护进程 | IPv6 | ✔ | 广播 IPv6 前缀、路由与 DNS 信息 |
| `dhcp6s` | DHCPv6 服务器（来自 ports） | IPv6 | ✘ | 需要时提供有状态 IPv6 配置 |

对大多数 OpenBSD 上的 IPv6 环境而言，`rad(8)` 与 [`slaacd(8)`](slaacd.md) 已足以提供稳健的自动配置，无需额外软件。

## 配置

`rad(8)` 的配置文件为 **/etc/rad.conf**。最小配置即在所选接口上通告：

```
interface em0
```

要在 `em0` 接口上通告 IPv6 前缀，按如下方式启用并启动守护进程：

```sh
# rcctl enable rad
# rcctl start rad
```

这样 `rad(8)` 就会在 `em0` 上发送路由通告，由客户端机器（通常运行 `slaacd(8)`）处理。

### 定制通告行为

更详尽的 **/etc/rad.conf** 可设置通告的前缀与 DNS 信息：

```
interface em0 {
    prefix 2001:db8:1::/64
    dns {
        nameserver 2001:db8:1::1
        search example.org
    }
}
```

这样会通告 LAN 前缀，并向支持的客户端提供递归 DNS 服务器（RDNSS）信息。

### 示例：使用静态 IPv6 前缀的 LAN 路由器

给定一个静态分配的 IPv6 前缀（例如 **2001:db8:1::/64**），路由器须：

1. 为其 LAN 接口分配地址：

   ```sh
   # ifconfig em0 inet6 2001:db8:1::1 prefixlen 64 alias
   ```
2. 创建 **/etc/rad.conf**，包含一个 `interface em0` 块。
3. 确保已启用数据包转发：

   ```sh
   # sysctl net.inet6.ip6.forwarding=1
   ```
4. 在 **/etc/sysctl.conf** 中持久化转发设置：

   ```conf
   net.inet6.ip6.forwarding=1
   ```
5. 启用并启动 `rad(8)`：

   ```sh
   # rcctl enable rad
   # rcctl start rad
   ```

## 关于 DHCPv6 与混合环境的说明

OpenBSD 基本系统不含 DHCPv6 服务器。`dhcp6s` 守护进程可通过 ports 集合获取，但通常仅在需要 **有状态** IPv6 配置（如集中 IP 管理或租约跟踪）时使用。

多数 OpenBSD 系统会这样运作：

- `rad(8)` 通告前缀、路由与可选的 DNS 解析器信息
- 客户端运行 [`slaacd(8)`](slaacd.md) 自动配置

如此即可提供完整的 IPv6 连通性，无需 DHCPv6，也符合常见的 IPv6 部署实践。

## 调试与验证

检查 `rad(8)` 是否已启用并运行：

```sh
# rcctl check rad
```

验证路由通告是否正在发送：

```sh
# tcpdump -n -i em0 icmp6
```

留意 `Router Advertisement` ICMPv6 消息。

验证客户端接口是否接受了 RA 并配置了地址：

```sh
# ifconfig em0
```

预期输出应包含一个 `inet6` 自动配置地址，通常以通告的前缀开头。

## 其他注意事项

确保系统防火墙（`pf(4)`）放行 IPv6 流量，包括 ICMPv6 路由通告：

**/etc/pf.conf** 片段示例：

```pf
pass inet6 proto ipv6-icmp from :: to ff02::1
```

该规则放行 SLAAC 所需的出站多播通告。
