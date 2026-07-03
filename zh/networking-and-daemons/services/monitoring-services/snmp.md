# SNMP

## 概述

**SNMP（简单网络管理协议，Simple Network Management Protocol）** 用于监控与管理网络上的设备。它允许外部系统查询接口状态、运行时长、系统描述与资源使用情况等信息。

OpenBSD 基本系统内置 SNMP 守护进程 `snmpd(8)`，提供只读的 SNMPv1 与 SNMPv2c 服务，适用于安全的本地与远程监控。

另一可选实现 **Net-SNMP** 通过软件包提供，支持 SNMPv3、写入操作以及广泛的 MIB 扩展。多数场景下 `snmpd(8)` 已足够，且更易于加固。

## SNMP 守护进程对比

| 特性 | [snmpd](snmp.md)（基本系统） | Net-SNMP（`pkg_add net-snmp`） |
| --- | --- | --- |
| SNMP 版本 | v1、v2c（只读） | v1、v2c、v3（读写、认证） |
| 配置 | **/etc/snmpd.conf** | **/etc/snmp/snmpd.conf** 或命令行 |
| 守护进程 | `snmpd(8)` | Net-SNMP 提供的 `snmpd` |
| 集成 | 原生 OpenBSD 特性 | 更通用，风格接近 Linux |
| SNMPv3 支持 | 否 | 是 |
| 访问控制 | 仅限团体名与来源地址 | 完整的视图/用户控制 |
| 默认安全性 | 默认安全，仅限 localhost | 除非加固，否则暴露范围广 |

除非需要 SNMPv3、set/walk 访问或跨平台 MIB 扩展，否则应使用 `snmpd(8)`。

## 启用 snmpd(8)

原生 SNMP 守护进程 `snmpd(8)` 属于基本系统，通过 **/etc/snmpd.conf** 进行配置。

启用并启动服务：

```sh
# rcctl enable snmpd
# rcctl start snmpd
```

默认配置仅绑定 `localhost`，并允许使用团体名 `public` 进行查询。

本地测试：

```sh
$ snmpctl walk community public
```

## 基本配置

编辑 **/etc/snmpd.conf**，定义团体访问与允许的来源地址。

以下示例配置向私有子网公开基本系统信息：

```
listen on 192.0.2.1

community "public" source 192.0.2.0/24

system description "OpenBSD Router"
system location "Data Center 1"
system contact "noc@example.org"

sensor temperature
sensor fan
sensor voltage

include pf
include interfaces
```

此配置：

- 监听接口 IP 192.0.2.1
- 接受来自 `192.0.2.0/24` 子网的 SNMP 查询
- 为监控软件添加 `system.*` 数值
- 暴露 `pf` 状态表与接口统计
- 启用传感器读数（若 `sysctl` 支持）

编辑后重新加载配置：

```sh
# rcctl reload snmpd
```

## SNMP 查询示例

使用 `snmpctl(8)` 在本地测试：

```sh
$ snmpctl walk community public
$ snmpctl get community public oid system.description
```

从远程系统查询（例如使用 Net-SNMP 工具）：

```sh
$ snmpwalk -v2c -c public 192.0.2.1
$ snmpget -v2c -c public 192.0.2.1 sysUpTime.0
```

获取接口统计：

```sh
$ snmpwalk -v2c -c public 192.0.2.1 interfaces
```

若配置了 `pf` 包含：

```sh
$ snmpwalk -v2c -c public 192.0.2.1 pf
```

## 安全注意事项

SNMPv1 与 SNMPv2c 使用**明文团体名**。为缩小暴露面：

- 在 `snmpd.conf` 中**限制 `source` 来源地址**
- 使用 `listen on` 绑定到指定接口
- 通过防火墙规则限制入站 UDP/161

`pf.conf` 规则示例：

```pf
block in proto udp to port 161
pass in on em0 proto udp from 192.0.2.0/24 to (em0) port 161
```

避免将 SNMP 暴露到互联网。带认证的 SNMPv3 仅由 Net-SNMP 支持。

## 日志

所有访问事件与错误均通过 `syslog` 记录：

```sh
# tail -f /var/log/daemon
```

日志级别固定，无法通过 `rcctl` 启用详细或调试输出。

## 系统集成

`sensorsd(8)` 与 `snmpd` 集成，用于暴露系统温度、电压与风扇数据。

启用 `sensorsd`：

```sh
# rcctl enable sensorsd
# rcctl start sensorsd
```

若在 `snmpd.conf` 中指定了 `sensor`，这些指标会自动暴露。

对于接口与防火墙统计，指令 `include interfaces` 与 `include pf` 分别从 `ifconfig` 与 `pfctl` 拉取数据。

## Net-SNMP（可选）

若需要 SNMPv3、可写访问或 MIB 支持：

```sh
# pkg_add net-snmp
```

Net-SNMP 配置较为复杂，使用其自有语法。配置 SNMPv3：

```sh
createUser readonly MD5 "password" DES
rouser readonly
```

完整配置请参阅 **/usr/local/share/examples/net-snmp/snmpd.conf**。
