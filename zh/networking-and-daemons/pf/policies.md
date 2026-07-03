# PF 策略

## 简介

数据包标记是一种为数据包打上内部标识符的方法，该标识符日后可用于过滤和转换规则的匹配条件。借助标记，可以在接口之间建立"信任"关系，也能判断数据包是否已经过转换规则处理。此外，还能摆脱基于规则的过滤，转而采用基于策略的过滤。

## 为数据包分配标记

要为数据包添加标记，使用 `tag` 关键字：

```pf
pass in on $int_if all tag INTERNAL_NET
```

任何匹配上述规则的数据包都会添加 `INTERNAL_NET` 标记。

也可以使用[宏](lists-and-macros.md#macros)来分配标记。例如：

```pf
name = "INTERNAL_NET"
pass in on $int_if all tag $name
```

还有一组预定义宏可供使用。

- `$if` - 接口
- `$srcaddr` - 源 IP 地址
- `$dstaddr` - 目的 IP 地址
- `$srcport` - 源端口
- `$dstport` - 目的端口
- `$proto` - 协议
- `$nr` - 规则编号

这些宏在规则集加载时展开，而非在运行时展开。

标记遵循以下规则：

- 标记具有"粘性"。一旦某条匹配规则为数据包打上标记，该标记永不删除。不过，可以用其他标记替换。
- 由于标记具有"粘性"，即使最后一条匹配规则未使用 `tag` 关键字，数据包仍可能带有标记。
- 任一时刻，一个数据包最多只能分配一个标记。
- 标记是*内部*标识符，不会通过网络发送出去。
- 标记名称最长为 63 个字符。

以如下规则集为例。

```pf
pass in on $int_if tag INT_NET
pass in quick on $int_if proto tcp to port 80 tag INT_NET_HTTP
pass in quick on $int_if from 192.168.1.5
```

- 从 `$int_if` 进入的数据包会由规则 #1 分配 `INT_NET` 标记。
- 从 `$int_if` 进入、目的端口为 80 的 TCP 数据包，首先由规则 #1 分配 `INT_NET` 标记，随后该标记由规则 #2 替换为 `INT_NET_HTTP`。
- 从 `$int_if` 进入、源地址为 **192.168.1.5** 的数据包，标记方式有两种。若该数据包的目的端口为 TCP 80，则匹配规则 #2，标记为 `INT_NET_HTTP`。否则，数据包匹配规则 #3，但会标记为 `INT_NET`。由于数据包匹配了规则 #1，`INT_NET` 标记即被应用，除非后续某条匹配规则指定了新标记，否则该标记不会删除（此即标记的"粘性"）。

## 检查已应用的标记

要检查此前应用的标记，使用 `tagged` 关键字：

```pf
pass out on egress tagged INT_NET
```

外部接口上的出站数据包必须带有 `INT_NET` 标记才能匹配上述规则。也可以使用 `!` 运算符进行反向匹配：

```pf
pass out on egress ! tagged WIFI_NET
```

## 策略过滤

策略过滤采用与编写过滤规则集不同的方法。先定义策略，规定哪些类型的流量允许通过、哪些类型予以阻断。随后，根据源/目的 IP 地址/端口、协议等传统条件，将数据包归入相应策略。例如，考察以下防火墙策略：

- 从内部 LAN 到 Internet 的流量允许通过（`LAN_INET`），且必须经过转换（`LAN_INET_NAT`）。
- 从内部 LAN 到 DMZ 的流量允许通过（`LAN_DMZ`）。
- 从 Internet 到 DMZ 中服务器的流量允许通过（`INET_DMZ`）。
- 从 Internet 重定向到 [spamd(8)](https://man.openbsdhandbook.com/spamd.8/) 的流量允许通过（`SPAMD`）。
- 所有其他流量一律阻断。

注意，该策略涵盖了流经防火墙的*所有*流量。括号中的条目表示该策略项所使用的标记。

接下来需要编写规则，将数据包归入相应策略。

```pf
block all
pass out on egress inet tag LAN_INET_NAT tagged LAN_INET nat-to ($ext_if)
pass in  on $int_if from $int_net tag LAN_INET
pass in  on $int_if from $int_net to $dmz_net tag LAN_DMZ
pass in  on egress proto tcp to $www_server port 80 tag INET_DMZ
pass in  on egress proto tcp from <spamd> to port smtp tag SPAMD rdr-to 127.0.0.1 port 8025
```

至此，定义策略的规则设置完毕。

```pf
pass in  quick on egress tagged SPAMD
pass out quick on egress tagged LAN_INET_NAT
pass out quick on $dmz_if tagged LAN_DMZ
pass out quick on $dmz_if tagged INET_DMZ
```

整个规则集设置完成后，后续修改只需调整分类规则。例如，若在 DMZ 中新增 POP3/SMTP 服务器，则需要为 POP3 和 SMTP 流量添加分类规则，如下所示：

```ini
mail_server = "192.168.0.10"
[...]
pass in on egress proto tcp to $mail_server port { smtp, pop3 } tag INET_DMZ
```

此后，邮件流量将作为 `INET_DMZ` 策略项的一部分予以放行。

完整规则集：

```sh
int_if      = "dc0"
dmz_if      = "dc1"
int_net     = "10.0.0.0/24"
dmz_net     = "192.168.0.0/24"
www_server  = "192.168.0.5"
mail_server = "192.168.0.10"

table <spamd> persist file "/etc/spammers"
# 分类——根据已定义的防火墙策略对数据包分类
block all
pass out on egress inet tag LAN_INET_NAT tagged LAN_INET nat-to (egress)
pass in on $int_if from $int_net tag LAN_INET
pass in on $int_if from $int_net to $dmz_net tag LAN_DMZ
pass in on egress proto tcp to $www_server port 80 tag INET_DMZ
pass in on egress proto tcp from <spamd> to port smtp tag SPAMD rdr-to 127.0.0.1 port 8025

# 策略执行——根据已定义的防火墙策略放行/阻断
pass in  quick on egress tagged SPAMD
pass out quick on egress tagged LAN_INET_NAT
pass out quick on $dmz_if tagged LAN_DMZ
pass out quick on $dmz_if tagged INET_DMZ
```

## 标记以太网帧

若执行标记/过滤的机器同时充当网桥，则可在以太网层执行标记。通过创建使用 `tag` 关键字的网桥过滤规则，可让 PF 基于源或目的 MAC 地址过滤流量。Bridge(4) 规则通过 `ifconfig` 命令创建。示例：

```sh
# ifconfig bridge0 rule pass in on fxp0 src 0:de:ad:be:ef:0 tag USER1
```

然后在 `pf.conf` 中：

```pf
pass in on fxp0 tagged USER1
```
