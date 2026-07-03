# BIND

## 概述

BIND（Berkeley Internet Name Domain）是一款功能完备的 DNS 服务器实现，由 ISC 维护。`unbound(8)` 与 `nsd(8)` 各自只承担一种 DNS 职能（解析器或权威服务器），而 BIND 的 `named(8)` 可同时承担两种角色。BIND 适用于需要灵活 DNS 视图、动态更新或复杂区域配置的高级场景。然而，由于复杂度高且历史上存在安全问题，BIND 未纳入 OpenBSD 基本系统，须从软件包安装。

本章介绍如何在 OpenBSD 7.8 上安装、配置与运行 BIND，涵盖权威与递归两种模式。

## DNS 服务对比

要厘清 BIND 在 OpenBSD 可用 DNS 软件中的定位，可参考下表：

| 特性 | `unbound(8)` | `nsd(8)` | `named`（BIND） |
| --- | --- | --- | --- |
| 用途 | 解析器 | 权威服务器 | 两者皆可 |
| 是否包含于 OpenBSD 基本系统 | 是 | 是 | 否 |
| 递归解析 | 是 | 否 | 是 |
| 权威服务器 | 否 | 是 | 是 |
| DNSSEC 校验 | 是 | 否（仅提供） | 是 |
| DNS-over-TLS 支持 | 是 | 否 | 是 |
| 复杂度 | 低 | 低 | 高 |
| 使用场景 | 客户端 | DNS 托管 | 复杂/混合 DNS 部署 |

## 安装

BIND 不属于 OpenBSD 基本系统。使用软件包系统安装：

```sh
# pkg_add isc-bind
```

该命令将 `named(8)` 及其配置工具安装到 **/etc/named/** 与 **/var/named/** 下。

开机启用 BIND：

```sh
# rcctl enable isc_named
```

立即启动服务：

```sh
# rcctl start isc_named
```

## 配置

主配置文件为：

```sh
/etc/named/named.conf
```

默认情况下，BIND 以 **/var/named** 作为工作目录，区域文件与运行时数据存放于此。

### 递归解析器示例

将 BIND 配置为带缓存的递归解析器：

```
options {
    directory "/var/named";
    allow-query { any; };
    recursion yes;
    allow-recursion { any; };
    listen-on { 127.0.0.1; };
    forwarders {
        8.8.8.8;
        1.1.1.1;
    };
};
```

保存配置后：

```sh
# named-checkconf /etc/named/named.conf
# rcctl restart isc_named
```

使用 `drill` 或 `dig` 验证：

```sh
$ drill @127.0.0.1 openbsd.org
```

### 权威服务器示例

将 BIND 配置为提供 `example.com` 区域服务：

```
zone "example.com" {
    type master;
    file "example.com.zone";
};
```

创建区域文件：

```
$ORIGIN example.com.
$TTL 3600
@   IN  SOA ns1.example.com. hostmaster.example.com. (
        2025080401 ; 序列号
        3600       ; 刷新
        1800       ; 重试
        604800     ; 过期
        86400 )    ; 最小 TTL
    IN  NS  ns1.example.com.
    IN  NS  ns2.example.com.
ns1 IN  A   192.0.2.1
ns2 IN  A   192.0.2.2
www IN  A   192.0.2.100
```

将该文件保存为 **/var/named/example.com.zone**。

检查语法并重载：

```sh
# named-checkzone example.com /var/named/example.com.zone
# rcctl restart isc_named
```

### 日志

日志须显式启用：

```
logging {
    channel default_log {
        file "/var/named/named.log";
        severity info;
        print-time yes;
    };
    category default { default_log; };
};
```

确保目录与文件可写：

```sh
# touch /var/named/named.log
# chown _named:_named /var/named/named.log
```

## DNSSEC 支持

BIND 支持 DNSSEC 签名与校验。递归解析器通过以下方式启用校验：

```
options {
    dnssec-validation auto;
};
```

要提供已签名区域，使用 `dnssec-keygen` 与 `dnssec-signzone`：

```sh
# dnssec-keygen -a ECDSAP256SHA256 -n ZONE example.com
# dnssec-signzone -o example.com example.com.zone Kexample.com.*
```

更新区域文件并照常重载。

## 运行时控制

BIND 使用 `rndc(8)` 进行控制：

```sh
# rndc reload
# rndc flush
# rndc status
```

要启用 `rndc`，须生成密钥并添加控制配置：

```
include "/etc/named/rndc.key";

controls {
    inet 127.0.0.1 allow { localhost; } keys { "rndc-key"; };
};
```

执行：

```sh
# rndc-confgen -a
```

该命令创建 **/etc/named/rndc.key**。

## 使用示例

### 作为本地解析器

本地使用 BIND：

1. 启用并配置 `recursion yes`
2. 设置 **/etc/resolv.conf**：

```
nameserver 127.0.0.1
```

### 作为权威服务器

提供区域服务：

1. 在 `named.conf` 中创建 `zone` 段
2. 在 **/var/named** 中创建区域文件
3. 测试并重载配置

使用 `drill` 查询：

```sh
$ drill @192.0.2.1 www.example.com
```

## 文件与目录

| 路径 | 用途 |
| --- | --- |
| **/etc/named/named.conf** | BIND 配置 |
| **/var/named/** | 区域与运行时文件 |
| **/etc/named/rndc.key** | `rndc` 控制密钥 |
| **/var/named/named.log** | 可选的日志输出 |

## 安全注意事项

BIND 体量大、复杂度高，历史上比最小化方案存在更多安全漏洞。应在 chroot 或 jailed 环境中运行（OpenBSD 通过 `chroot(2)` 自动完成）。功能允许时，优先选用 `unbound(8)` 或 `nsd(8)`。
