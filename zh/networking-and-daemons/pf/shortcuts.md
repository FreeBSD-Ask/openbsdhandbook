# PF 简化技巧

## 简介

PF 提供多种简化规则集的方法。较好的例子包括使用[宏](lists-and-macros.md#macros)和[列表](lists-and-macros.md#lists)。此外，规则集语言（即语法）也提供了一些简化规则集的捷径。一般而言，规则集越简单，越容易理解和维护。

## 使用宏

宏很有用，因为它避免了将地址、端口号、接口名称等硬编码进规则集。服务器 IP 地址变了？没问题，只需更新宏即可；无需再去改动那些你花时间精力精心调整的过滤规则。

PF 规则集中的一种常见约定是为每个网络接口定义一个宏。若网卡需要更换为使用不同驱动程序的型号，只需更新宏，过滤规则即可照常工作。另一个好处是在多台机器上安装同一规则集时：某些机器可能装有不同网卡，用宏定义网络接口后，规则集只需极少的修改即可部署。用宏定义规则集中易于变化的信息（如端口号、IP 地址、接口名称）是推荐做法。

```conf
# 为每个网络接口定义宏
IntIF = "dc0"
ExtIF = "fxp0"
DmzIF = "fxp1"
```

另一种常见约定是用宏定义 IP 地址和网络块。当 IP 地址变更时，这能大幅减少规则集的维护工作。

```conf
# 定义我们的网络
IntNet = "192.168.0.0/24"
ExtAdd = "192.0.2.4"
DmzNet = "10.0.0.0/24"
```

若内部网络扩展或重新编号为不同的 IP 段，更新宏即可：

```
IntNet = "{ 192.168.0.0/24, 192.168.1.0/24 }"
```

规则集重新加载后，一切照常工作。

## 使用列表

来看一组适合放入规则集的规则，用于处理不应在 Internet 上出现的 [RFC 1918](https://tools.ietf.org/html/rfc1918) 地址——这类地址一旦出现在 Internet 上，通常意味着麻烦：

```pf
block in  quick on egress inet from 127.0.0.0/8 to any
block in  quick on egress inet from 192.168.0.0/16 to any
block in  quick on egress inet from 172.16.0.0/12 to any
block in  quick on egress inet from 10.0.0.0/8 to any
block out quick on egress inet from any to 127.0.0.0/8
block out quick on egress inet from any to 192.168.0.0/16
block out quick on egress inet from any to 172.16.0.0/12
block out quick on egress inet from any to 10.0.0.0/8
```

再看下面的简化版本：

```pf
block in quick  on egress inet from { 127.0.0.0/8, 192.168.0.0/16, \
   172.16.0.0/12, 10.0.0.0/8 } to any
block out quick on egress inet from any to { 127.0.0.0/8, \
   192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8 }
```

规则集从八行精简为两行。若将宏与列表结合使用，效果更佳：

```pf
NoRouteIPs = "{ 127.0.0.0/8, 192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8 }"

block in  quick on egress from $NoRouteIPs to any
block out quick on egress from any to $NoRouteIPs
```

注意，宏和列表简化的是 `pf.conf` 文件本身，但这些行实际上会由 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/) 展开为多条规则。因此，上述示例实际展开为如下规则：

```pf
block in  quick on egress inet from 127.0.0.0/8 to any
block in  quick on egress inet from 192.168.0.0/16 to any
block in  quick on egress inet from 172.16.0.0/12 to any
block in  quick on egress inet from 10.0.0.0/8 to any
block out quick on egress inet from any to 10.0.0.0/8
block out quick on egress inet from any to 172.16.0.0/12
block out quick on egress inet from any to 192.168.0.0/16
block out quick on egress inet from any to 127.0.0.0/8
```

可见，PF 的展开纯粹是为了方便 `pf.conf` 文件的编写者和维护者，并非真正简化了 pf 所处理的规则。

宏不仅能定义地址和端口，还能用于 PF 规则文件中的任何位置：

```pf
pre  = "pass in quick on ep0 inet proto tcp from "
post = "to any port { 80, 6667 }"

$pre 198.51.100.80 $post
$pre 203.0.113.79  $post
$pre 203.0.113.178 $post
```

展开后为：

```pf
pass in quick on ep0 inet proto tcp from 198.51.100.80 to any port = 80
pass in quick on ep0 inet proto tcp from 198.51.100.80 to any port = 6667
pass in quick on ep0 inet proto tcp from 203.0.113.79  to any port = 80
pass in quick on ep0 inet proto tcp from 203.0.113.79  to any port = 6667
pass in quick on ep0 inet proto tcp from 203.0.113.178 to any port = 80
pass in quick on ep0 inet proto tcp from 203.0.113.178 to any port = 6667
```

## PF 语法

PF 的语法相当灵活，这也使得规则集具备很大的灵活性。PF 能推断某些关键字，意味着这些关键字无需在规则中显式写出；关键字的顺序也比较宽松，不必死记严格的语法。

### 关键字的省略

要定义"默认拒绝"策略，需要两条规则：

```pf
block in  all
block out all
```

这可以精简为：

```
block
```

未指定方向时，PF 会认定该规则对双向数据包均适用。

类似地，`from any to any` 和 `all` 子句也可以省略，例如：

```pf
block in on rl0 all
pass  in quick log on rl0 proto tcp from any to any port 22
```

可以简化为：

```pf
block in on rl0
pass  in quick log on rl0 proto tcp to port 22
```

第一条规则阻断 `rl0` 上来自任何地方、去往任何地方的所有入站数据包，第二条规则放行 `rl0` 上去往端口 22 的 TCP 流量。

### `return` 的简化

若规则集用于阻断数据包并以 TCP RST 或 ICMP Unreachable 响应，可能如下：

```pf
block in all
block return-rst  in proto tcp all
block return-icmp in proto udp all
block out all
block return-rst  out proto tcp all
block return-icmp out proto udp all
```

这可以简化为：

```
block return
```

PF 看到 `return` 关键字时，会根据所阻断数据包的协议，自动发送适当的响应，或不作任何响应。

### 关键字顺序

大多数情况下，关键字的指定顺序很灵活。例如，如下规则：

```pf
pass in log quick on rl0 proto tcp to port 22 flags S/SA queue ssh label ssh
```

也可以写成：

```pf
pass in quick log on rl0 proto tcp to port 22 queue ssh label ssh flags S/SA
```

其他类似的变体同样有效。
