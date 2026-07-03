# 大规模网络服务

## 概述

本章介绍如何在 OpenBSD 上构建可靠、可扩展的网络服务：用 [nsd(8)](https://man.openbsdhandbook.com/nsd.8/) 配合 [nsd.conf(5)](https://man.openbsdhandbook.com/nsd.conf.5/) 提供权威 DNS；用 [unbound(8)](https://man.openbsdhandbook.com/unbound.8/) 配合 [unbound.conf(5)](https://man.openbsdhandbook.com/unbound.conf.5/) 提供带验证的递归 DNS；用 [dhcpd(8)](https://man.openbsdhandbook.com/dhcpd.8/) 配合 [dhcpd.conf(5)](https://man.openbsdhandbook.com/dhcpd.conf.5/) 进行 IPv4 地址分配；用 [ntpd(8)](https://man.openbsdhandbook.com/ntpd.8/) 配合 [ntpd.conf(5)](https://man.openbsdhandbook.com/ntpd.conf.5/) 提供时间服务。服务生命周期管理使用 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)。包过滤放行规则位于 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)，由 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/) 应用。这些模式适用于将 DNS、DHCP 和 NTP 作为一等冗余服务运行的分支机构、园区网与 PoP。

## 设计考量

- **角色分离。** 将权威 DNS 与递归 DNS 绑定到不同 IP 地址。常见做法是 NSD 监听公网/WAN 地址（或专用服务 VIP），Unbound 监听 LAN 地址。
- **冗余。** 每个关键服务至少使用两台服务器。结合高可用章节的 CARP 与 pfsync，实现服务 VIP 故障切换。
- **安全姿态。** 仅允许来自互联网的 UDP/TCP 53 流量**到达权威服务器**。将递归限制在内网前缀。在 Unbound 中启用 DNSSEC 验证。
- **运维边界。** NSD 不支持动态更新；区域管理须显式进行并纳入版本控制。Unbound 不是权威服务器；二者同主机运行时，用 stub 区域优先采用本地权威应答。
- **IPv6。** 在权威区域中提供 AAAA 记录，确保递归服务同时应答 A 与 AAAA。IPv6 地址分配优先采用 RA（参见 IPv6 章节）；DHCPv6 不在本章范围内。
- **时间服务。** DNSSEC 验证要求时间正确。确保解析器上运行 `ntpd(8)`，且客户端有本地 NTP 源。

## 配置

示例假定：

- 接口：`em0`（WAN）、`em1`（LAN `10.10.10.0/24`）。
- 服务 IP：`203.0.113.53`（WAN 上的权威 DNS）、`10.10.10.1`（LAN 上的递归 DNS 与 NTP）。
- 权威区域：`example.com.`，由 NSD 提供。
- LAN 客户端的本地递归由 Unbound 提供。

### 1) 用 nsd(8) 提供权威 DNS

将 NSD 绑定到公网地址，从本地区域文件提供 `example.com.`。启用控制套接字以便安全重载。

```
## /etc/nsd.conf — WAN 地址上的 NSD 权威服务

server:
    ip-address: 203.0.113.53
    hide-version: yes
    do-ip4: yes
    do-ip6: no
    server-count: 1
    zonesdir: "/var/nsd/zones"
    # nsd-control(8) 的控制套接字
    control-enable: yes

zone:
    name: "example.com"
    zonefile: "example.com.zone"
```

创建区域文件（最小示例；按需扩展）。生产环境中 SOA 与 NS 记录必须正确。

```
## /var/nsd/zones/example.com.zone — 最小区域

$ORIGIN example.com.
$TTL 300
@   IN SOA ns1.example.com. hostmaster.example.com. (
        2025010101 ; 序列号 (YYYYMMDDNN)
        3600       ; 刷新
        600        ; 重试
        1209600    ; 过期
        300        ; 最小 TTL
)
    IN  NS  ns1.example.com.
ns1 IN  A   203.0.113.53
www IN  A   203.0.113.80
api IN  A   203.0.113.81
```

首次设置控制密钥并启动服务：

```sh
# nsd-control-setup
  # 在 /var/nsd/etc 下生成控制密钥/证书；一次性操作
# rcctl enable nsd
  # 开机启动
# rcctl start nsd
  # 立即启动
# nsd-control reload
  # 校验并加载区域
```

### 2) 用 unbound(8) 提供带验证的递归 DNS

Unbound 将监听 LAN，仅服务内网客户端，并验证 DNSSEC。NSD 与 Unbound 同主机时，通过 **stub** 将本地区域转发到 NSD，让 NSD 监听 WAN、Unbound 监听 LAN，以避免 53 端口冲突。

```
## /var/unbound/etc/unbound.conf — 带验证的 LAN 递归

server:
    interface: 127.0.0.1
    interface: 10.10.10.1
    access-control: 10.10.10.0/24 allow
    access-control: 127.0.0.0/8 allow

    # 加固与隐私
    hide-identity: yes
    hide-version: yes
    qname-minimisation: yes
    harden-referral-path: yes
    harden-glue: yes
    prefetch: yes

    # DNSSEC
    auto-trust-anchor-file: "/var/unbound/db/root.key"

    # 根引导；Unbound 可在需要时自动获取根 NS
    root-hints: "/var/unbound/db/root.hints"

# 优先采用本地权威应答（由 NSD 提供的可选内部区域）
stub-zone:
    name: "corp.example."
    stub-addr: 127.0.0.1@5353
```

若需要 Unbound 优先采用本地区域、同时 NSD 保留 WAN 上的 53 端口，可让 NSD 额外绑定 `127.0.0.1 port 5353` 并在该处提供 `corp.example.`（见下例）。否则省略 `stub-zone`。

```
## 摘录 — 内部区域的额外 NSD 监听器（环回）

server:
    # ... 既有的 server 块 ...
    ip-address: 127.0.0.1@5353

zone:
    name: "corp.example"
    zonefile: "corp.example.zone"
```

初始化 Unbound 控制并启动解析器：

```sh
# unbound-control-setup
  # 在 /var/unbound/etc 下生成控制密钥/证书；一次性操作
# rcctl enable unbound
# rcctl start unbound
# unbound-control status
  # 确认验证器已就绪并加载
```

### 3) 用 dhcpd(8) 提供 IPv4 DHCP

提供 `10.10.10.0/24`，通告本地解析器与默认网关，并给出静态映射示例。

```
## /etc/dhcpd.conf — LAN 上的 IPv4 DHCP

option domain-name "corp.example";
option domain-name-servers 10.10.10.1;
option routers 10.10.10.1;
option subnet-mask 255.255.255.0;
default-lease-time 3600;
max-lease-time 7200;
authoritative;

subnet 10.10.10.0 netmask 255.255.255.0 {
    range 10.10.10.100 10.10.10.200;

    # 静态保留示例
    host printer-01 {
        hardware ethernet 00:11:22:33:44:55;
        fixed-address 10.10.10.50;
    }
}
```

启用守护进程并将其绑定到 LAN 接口：

```sh
# rcctl set dhcpd flags em1
  # 指定服务的接口（必填）
# rcctl enable dhcpd
# rcctl start dhcpd
# tail -n 20 /var/db/dhcpd.leases
  # 查看活动租约
```

> IPv6 地址分配优先采用 `rad(8)` 的路由通告，参见 IPv6 章节。

### 4) 用 ntpd(8) 提供时间服务

在 LAN 上提供内部 NTP 源，并保持服务器自身时间正确。

```
## /etc/ntpd.conf — 服务 LAN 并从上游池同步

listen on 10.10.10.1
servers pool.ntp.org
```

```sh
# rcctl enable ntpd
# rcctl start ntpd
# ntpctl -s status
  # 报告同步状态
```

### 5) DNS、DHCP 与 NTP 的 PF 放行规则

允许外部查询到达 WAN 上的 NSD、LAN 上的递归服务、LAN 上的 DHCP，以及客户端的 NTP。

```pf
## /etc/pf.conf — 服务放行规则（合并到你的策略中）

set skip on lo

wan = "em0"
lan = "em1"

# 基础策略
block all

# WAN 上的权威 DNS（NSD）
pass in on $wan proto { udp tcp } to 203.0.113.53 port 53 keep state

# LAN 上的递归 DNS（Unbound）
pass in on $lan proto { udp tcp } to 10.10.10.1 port 53 keep state

# LAN 上的 DHCP（服务器从 67 端口应答到客户端 68 端口）
pass in on $lan proto udp from port 68 to port 67 keep state
pass out on $lan proto udp from port 67 to port 68 keep state

# LAN 上的 NTP
pass in on $lan proto udp to 10.10.10.1 port 123 keep state

# 解析器与 NSD 的出站按需放行
pass out on $wan proto { udp tcp } to any port { 53, 123 } keep state
```

编辑后重载：

```sh
# pfctl -f /etc/pf.conf
  # 原子加载规则
```

## 验证

- **NSD 提供的权威区域**（从远端主机或防火墙本身），使用 `drill(1)`：

```sh
$ drill @203.0.113.53 example.com SOA
  # 期望返回所配置的 SOA
$ drill @203.0.113.53 www.example.com A
  # 期望返回 203.0.113.80
```

- **带 DNSSEC 验证的递归解析**（从 LAN 客户端）：

```sh
$ drill @10.10.10.1 cloudflare.com A
  # 常规递归可用
$ drill -S @10.10.10.1 dnssec-failed.org
  # 应返回验证失败（bogus），证明 DNSSEC 已生效
```

- **Unbound 与 NSD 控制**：

```sh
# unbound-control list_stubs
  # 确认已加载的 stub 区域
# nsd-control stats
  # 权威服务的基本 qps/计数器
```

- **DHCP 租约与客户端配置**：

```sh
$ grep dhcp /var/log/messages | tail -n 20
  # 服务器活动
$ cat /var/db/dhcpd.leases | tail -n 10
  # 租约数据库更新
```

- **NTP 状态**：

```sh
$ ntpctl -s peers
  # 上游对等体与偏移
$ ntpctl -s status
  # 总体同步状态
```

## 故障排查

- **53 端口冲突。** NSD 与 Unbound 不能同时绑定同一 IP:端口。将 NSD 绑定到 WAN 地址、Unbound 绑定到 LAN 地址，或将其中一个移至 `127.0.0.1@5353`，并用 Unbound 的 `stub-zone` 处理内部域。
- **开放解析器暴露。** 若外部主机可递归，用 `access-control` 将 Unbound 限制在内网前缀，并确认 PF 拒绝 WAN 访问递归监听器。
- **DNSSEC 失败。** 验证要求系统时间正确且信任锚为最新。确保 `ntpd(8)` 已同步，且 Unbound 拥有有效的 `auto-trust-anchor-file`。
- **权威区域未加载。** 检查 `nsd-control reload` 输出与 syslog 中的语法错误。校验 SOA/NS 正确性与序列号递增。
- **DHCP 未服务。** 确认 `rcctl set dhcpd flags <lan-ifaces>`，且 PF 允许 LAN 上的 UDP 67/68。检查 `/var/db/dhcpd.leases` 与日志，排查地址池耗尽或 MAC 不匹配。
- **客户端忽略 NTP。** 确保 LAN 客户端指向 `10.10.10.1` 作为 NTP 源，且 PF 允许从 LAN 到服务器的 UDP 123。

## 参见

- [网络](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- 相关：[**高可用与状态复制**](high-availability.md)
- 相关：[**遥测、日志与流导出**](telemetry-logging-flow.md)
- 相关：[**大规模 IPv6**](ipv6-at-scale.md)
