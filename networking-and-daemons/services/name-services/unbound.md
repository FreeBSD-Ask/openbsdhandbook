# Unbound

## 概述

`unbound(8)` 是一款支持校验、递归与缓存的 DNS 解析器，内置于 OpenBSD 基本系统。该服务默认为本地名称解析启用，适用于工作站与服务器环境。Unbound 注重简洁、安全与性能，支持 DNSSEC 校验、本地区域以及 DNS-over-TLS 等特性。

本章介绍如何在 OpenBSD 上配置与管理 `unbound(8)`，并涵盖 DNS 隐私与本地网络集成等高级选项。本章还提供与其他 DNS 服务的对比，以厘清解析器与权威服务器之间的差异。

## DNS 服务对比

下表列出 OpenBSD 上常用的三种 DNS 服务 `unbound(8)`、`nsd(8)` 与 BIND（`named(8)`）之间的关键差异。

| 特性 | `unbound(8)` | `nsd(8)` | `named(8)`（BIND） |
| --- | --- | --- | --- |
| 用途 | 解析器 | 权威服务器 | 两者皆可 |
| 是否包含于 OpenBSD 基本系统 | 是 | 是 | 否 |
| 递归解析 | 是 | 否 | 是 |
| 权威服务器 | 否 | 是 | 是 |
| DNSSEC 校验 | 是 | 否 | 是 |
| DNS-over-TLS 支持 | 是 | 否 | 是 |
| 复杂度 | 低 | 低 | 高 |
| 使用场景 | 客户端系统 | 托管区域 | 混合环境 |

需要安全、私密的 DNS 解析时使用 `unbound(8)`。需要对外提供 DNS 区域服务时使用 `nsd(8)`。仅当需要兼具两种角色，或需要 `unbound` 与 `nsd` 不具备的高级 DNS 功能时，才使用 BIND（`named`）。

## 启用 Unbound

OpenBSD 已预装 Unbound，并默认为本地解析启用。

查看其状态：

```sh
# rcctl check unbound
```

显式启用：

```sh
# rcctl enable unbound
# rcctl start unbound
```

Unbound 的配置文件位于：

```sh
/etc/unbound/unbound.conf
```

## 默认配置

默认配置通过 localhost 提供 DNS 解析，并启用缓存与 DNSSEC 校验。

检查 **/etc/resolv.conf**：

```sh
nameserver 127.0.0.1
lookup file bind
```

这样系统解析器就会使用运行于回环接口上的 Unbound。

## 配置

默认的 **/var/unbound/etc/unbound.conf** 由系统自动生成。自定义配置可放在 **/etc/unbound/unbound.conf**；若禁用基本配置，也可放在 **/var/unbound/etc/unbound.conf**。

要改用静态配置：

1. 禁用自动配置生成：

```sh
# rcctl disable resolvd
```

2. 创建配置文件：

```
server:
    verbosity: 1
    interface: 127.0.0.1
    access-control: 127.0.0.0/8 allow
    cache-max-ttl: 86400
    cache-min-ttl: 3600
    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    auto-trust-anchor-file: "/var/unbound/db/root.key"
    prefetch: yes

forward-zone:
    name: "."
    forward-tls-upstream: yes
    forward-addr: 1.1.1.1@853       # Cloudflare 的 DNS over TLS
    forward-addr: 9.9.9.9@853       # Quad9 的 DNS over TLS
```

然后重启 Unbound：

```sh
# rcctl restart unbound
```

## DNS over TLS

要启用 DNS-over-TLS 安全查询，使用 `forward-zone` 块并指定使用 TLS 端口（`@853`）的上游服务器。Unbound 须在编译时加入 TLS 支持（OpenBSD 基本系统版本已包含）。

校验解析与 DNSSEC 校验：

```sh
$ drill openbsd.org
$ drill -D openbsd.org
```

`-D` 显示 DNSSEC 状态。

## 本地区域与主机覆盖

可使用 `local-data` 或 `local-zone` 添加静态主机映射：

```
local-zone: "example.test." static
local-data: "host1.example.test. IN A 192.168.1.10"
local-data: "host2.example.test. IN AAAA fd00::1"
```

这样无需依赖外部 DNS，即可在内部覆盖名称解析。

## 使用 DNSSEC

DNSSEC 默认启用。信任锚通过以下文件管理：

```sh
/var/unbound/db/root.key
```

刷新信任锚：

```sh
# unbound-anchor -a /var/unbound/db/root.key
```

DNSSEC 校验失败时将返回 SERVFAIL 响应，以防欺骗。

## 与 dhclient 集成

默认情况下，`dhclient(8)` 与 `resolvd(8)` 配合工作，后者可能覆盖 **/etc/resolv.conf**。

为避免冲突：

1. 禁用 `resolvd`：

```sh
# rcctl disable resolvd
```

2. 手动设置 **resolv.conf**：

```sh
# echo 'nameserver 127.0.0.1' > /etc/resolv.conf
```

3. 阻止 `dhclient` 覆盖该文件：

```sh
# chflags schg /etc/resolv.conf
```

要恢复原状：

```sh
# chflags noschg /etc/resolv.conf
```

## 监控与调试

查看基本状态：

```sh
# unbound-control status
```

查询统计信息：

```sh
# unbound-control stats
```

清空缓存：

```sh
# unbound-control flush
```

跟踪解析过程：

```sh
# unbound-control lookup openbsd.org
```

日志写入 **/var/log/messages**。可在配置中提高详细级别以便调试：

```
server:
    verbosity: 3
```

然后重启：

```sh
# rcctl restart unbound
```

## 使用示例

测试本地解析：

```sh
$ doas drill openbsd.org
    # 使用本地解析器解析 openbsd.org
$ doas unbound-control status
    # 显示当前状态，包括运行时间、版本与统计信息
```

## 关键文件与目录汇总

| 文件/目录 | 说明 |
| --- | --- |
| **/etc/unbound/unbound.conf** | 可选的本地配置 |
| **/var/unbound/etc/unbound.conf** | 当前生效的配置（默认） |
| **/var/unbound/db/root.key** | DNSSEC 根信任锚 |
| **/etc/resolv.conf** | 系统解析器配置 |
| **/var/log/messages** | unbound 日志的默认位置 |
