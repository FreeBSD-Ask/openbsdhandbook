# PF 表

## 简介

表用于保存一组 IPv4 和/或 IPv6 地址。表的查询速度极快，且比[列表](lists-and-macros.md#lists)消耗更少的内存和处理器时间。因此，表非常适合保存大量地址——保存 50,000 个地址的表，查询时间仅比保存 50 个地址的表略长。表可用于以下用途：

- 规则中的源和/或目的地址。
- 转换和重定向地址，分别对应 `nat-to` 和 `rdr-to` 规则选项。
- `route-to`、`reply-to` 和 `dup-to` 规则选项中的目的地址。

表可在 **/etc/pf.conf** 中创建，也可使用 `pfctl` 创建。

## 配置

在 `pf.conf` 中，使用 `table` 指令创建表。每个表可指定以下属性：

- `const` - 表创建后，其内容不可更改。未指定此属性时，可随时使用 `pfctl` 在表中添加或删除地址，即使在 securelevel 为 2 或更高时也可操作。
- `persist` - 即使没有规则引用该表，内核也会将其保留在内存中。未指定此属性时，当引用该表的最后一条规则被清除后，内核会自动删除该表。

示例：

```pf
table <goodguys> { 192.0.2.0/24 }
table <rfc1918>  const { 192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8 }
table <spammers> persist
block in on fxp0 from { <rfc1918>, <spammers> } to any
pass  in on fxp0 from <goodguys> to any
```

地址也可使用取反（即"非"）修饰符指定，例如：

```pf
table <goodguys> { 192.0.2.0/24, !192.0.2.5 }
```

`goodguys` 表现在会匹配 **192.0.2.0/24** 网络中的所有地址，但 **192.0.2.5** 除外。

注意，表名始终包含在尖括号 `< >` 中。

表也可从包含 IP 地址和网络列表的文本文件中填充：

```pf
table <spammers> persist file "/etc/spammers"
block in on fxp0 from <spammers> to any
```

**/etc/spammers** 文件包含 IP 地址和/或 [CIDR](https://web.archive.org/web/20150213012421/http://public.swbell.net/dedicated/cidr.html) 网络块的列表，每行一条。

## 使用 `pfctl` 操作

可随时使用 `pfctl` 操作表。例如，要向上一步创建的表添加条目：

```sh
# **pfctl -t spammers -T add 203.0.113.0/24**
```

若表不存在，此命令也会创建该表。要列出表中的地址：

```sh
# **pfctl -t spammers -T show**
```

`-v` 参数也可与 `-T show` 配合使用，以显示每个表条目的统计信息。要从表中删除地址：

```sh
# **pfctl -t spammers -T delete 203.0.113.0/24**
```

有关使用 `pfctl` 操作表的更多信息，请参阅 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/) 手册页。

## 指定地址

除用 IP 地址指定外，主机也可用主机名指定。主机名解析为 IP 地址时，所有解析得到的 IPv4 和 IPv6 地址都会放入表中。IP 地址也可通过指定有效的接口名、接口组或 `self` 关键字来加入表。随后，表中将包含分配给该接口或组的所有 IP 地址，或分配给本机的所有 IP 地址（包括环回地址）。

指定地址时有一个限制：**0.0.0.0/0** 和 **0/0** 在表中不可用。替代办法是硬编码该地址，或使用[宏](lists-and-macros.md#macros)。

## 地址匹配

对表的地址查询会返回匹配范围最窄的条目。由此可创建如下表：

```pf
table <goodguys> { 172.16.0.0/16, !172.16.1.0/24, 172.16.1.100 }

block in on dc0
pass  in on dc0 from <goodguys>
```

任何从 `dc0` 进入的数据包，其源地址都会与 `<goodguys>` 表进行匹配：

- **172.16.50.5** - 最窄匹配为 **172.16.0.0/16**；数据包匹配该表，将被放行
- **172.16.1.25** - 最窄匹配为 `!172.16.1.0/24`；数据包匹配了表中的某条目，但该条目被取反（使用了 `!` 修饰符）；数据包不匹配该表，将被阻断
- **172.16.1.100** - 精确匹配 **172.16.1.100**；数据包匹配该表，将被放行
- **10.1.4.55** - 不匹配该表，将被阻断
