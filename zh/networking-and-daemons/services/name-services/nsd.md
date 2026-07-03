# NSD

## 概述

`nsd(8)` 是由 NLnet Labs 开发的高性能权威 DNS 服务器，内置于 OpenBSD 基本系统。该服务器专用于安全、高效地提供 DNS 区域服务，不支持递归查询。`unbound(8)` 负责为客户端执行 DNS 解析，而 `nsd(8)` 只应答其显式配置所服务的域的查询。

本章介绍如何在 OpenBSD 上配置与管理 `nsd(8)`，以便向公众或内部网络提供 DNS 区域服务。

## DNS 服务对比

要理解 `nsd(8)` 在 OpenBSD DNS 生态中的定位，可参考下表：

| 特性 | `unbound(8)` | `nsd(8)` | `named(8)`（BIND） |
| --- | --- | --- | --- |
| 用途 | 解析器 | 权威服务器 | 两者皆可 |
| 是否包含于 OpenBSD 基本系统 | 是 | 是 | 否 |
| 递归解析 | 是 | 否 | 是 |
| 权威服务器 | 否 | 是 | 是 |
| DNSSEC 校验 | 是 | 否（仅提供 DNSSEC 数据） | 是 |
| DNS-over-TLS 支持 | 是 | 否 | 是 |
| 复杂度 | 低 | 低 | 高 |
| 使用场景 | 客户端系统 | 托管区域 | 混合环境 |

## 安装

`nsd(8)` 包含于 OpenBSD 7.8 基本系统，无需额外安装软件包。

开机启用 `nsd`：

```sh
# rcctl enable nsd
```

启动守护进程：

```sh
# rcctl start nsd
```

主配置目录为 **/var/nsd**。配置文件存放于：

```sh
/var/nsd/etc/nsd.conf
```

区域文件通常存放于：

```sh
/var/nsd/zones/
```

## 配置

### 基础配置文件

下面是域 `example.com` 的最小配置示例：

```
server:
    hide-version: yes
    ip-address: 192.0.2.1

zone:
    name: "example.com"
    zonefile: "example.com.zone"
```

`server:` 段定义守护进程级选项，`zone:` 段定义每个待服务的域。

在 **/var/nsd/etc/nsd.conf** 创建该文件后，测试配置：

```sh
# nsd-checkconf /var/nsd/etc/nsd.conf
```

### 区域文件格式

区域文件采用标准 BIND 风格语法：

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

将该文件保存为 **/var/nsd/zones/example.com.zone**。

### 权限

确保属主与权限正确：

```sh
# chown -R _nsd:_nsd /var/nsd
# chmod -R 755 /var/nsd
```

## 区域编译与重载

`nsd(8)` 提供区域服务前，须先将区域编译为二进制数据库：

```sh
# nsd-control rebuild
```

该命令读取配置并编译区域数据。随后重载：

```sh
# nsd-control reload
```

查看运行状态：

```sh
# nsd-control status
```

## NSD 与 DNSSEC

`nsd(8)` 不校验 DNSSEC，但可提供已签名的区域数据。签名须在外部完成（例如使用 `ldns-signzone`）。

对区域文件签名的示例：

```sh
# ldns-signzone example.com.zone Kexample.com.+008+12345.key Kexample.com.+008+12345.private
```

更新 **nsd.conf** 引用已签名的区域文件，然后照常重建与重载。

## 控制 NSD

使用 `nsd-control(8)` 进行运行时控制：

```sh
# nsd-control status
# nsd-control reload
# nsd-control addzone example.com /var/nsd/etc/nsd.conf
# nsd-control delzone example.com
```

`nsd-control` 工具需要控制密钥与证书，使用以下命令创建：

```sh
# nsd-control-setup
```

该命令生成 **/var/nsd/etc/nsd_control.key** 与 `.pem` 文件。

## 日志与调试

日志通过 `syslog(3)` 发送至 `daemon` 设施。查看日志：

```sh
# tail -f /var/log/daemon
```

调试时提高详细级别：

```
server:
    verbosity: 2
```

然后重启服务：

```sh
# rcctl restart nsd
```

## 使用示例

提供域 `example.com` 服务：

1. 将区域文件放在 **/var/nsd/zones/example.com.zone**
2. 在 **/var/nsd/etc/nsd.conf** 中添加配置
3. 执行：

```sh
# nsd-checkconf
# nsd-control rebuild
# rcctl restart nsd
```

验证：

```sh
$ drill @192.0.2.1 www.example.com
```

## 关键文件与目录汇总

| 文件/目录 | 说明 |
| --- | --- |
| **/var/nsd/etc/nsd.conf** | NSD 配置文件 |
| **/var/nsd/zones/** | 区域数据文件 |
| **/var/nsd/db/nsd.db** | 编译后的二进制区域数据库 |
| **/var/nsd/run/nsd.pid** | `nsd(8)` 的 PID 文件 |
| `/var/nsd/etc/nsd_control.*` | `nsd-control` 的 TLS 密钥与证书 |
| **/var/log/daemon** | 经 syslog 输出的日志 |
