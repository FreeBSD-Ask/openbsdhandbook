# PF 负载均衡

## 简介

地址池是两个或更多地址的集合，由一组用户共享使用。可在 [filter](filter.md) 选项 `nat-to`、`rdr-to`、`route-to`、`reply-to` 与 `dup-to` 中将其指定为目标地址。

地址池有四种使用方式：

- `bitmask` — 将池地址的网络部分嫁接到被修改地址（`nat-to` 规则的源地址、`rdr-to` 规则的目的地址）之上。例：若地址池为 **192.0.2.1/24**、被修改地址为 **10.0.0.50**，则结果地址为 **192.0.2.50**。若地址池为 **192.0.2.1/25**、被修改地址为 **10.0.0.130**，则结果地址为 **192.0.2.2**。
- `random` — 从池中随机选取一个地址。
- `source-hash` — 用源地址的哈希值决定使用池中的哪个地址。此方法确保给定源地址始终映射到同一池地址。喂给哈希算法的密钥可选择性地以十六进制或字符串形式在 `source-hash` 关键字后指定。默认情况下，pfctl 每次加载规则集时都会生成一个随机密钥。
- `round-robin` — 在地址池中按顺序循环。这是默认方法，也是地址池用[表](tables.md)指定时唯一允许的方法。

除 `round-robin` 方法外，地址池必须以 [CIDR](https://web.archive.org/web/20150213012421/http://public.swbell.net/dedicated/cidr.html)（无类别域间路由）网络块形式表示。`round-robin` 方法可使用[列表](lists-and-macros.md#lists)或[表](tables.md)接受多个单独地址。

`sticky-address` 选项可与 `random` 和 `round-robin` 池类型联用，确保特定源地址始终映射到同一重定向地址。

## NAT 地址池

地址池可作为 [`nat-to`](nat.md) 规则中的转换地址。连接会依据所选方法，将源地址转换为池中的某个地址。这在 PF 为超大型网络执行 NAT 时很有用。由于每个转换地址的 NAT 连接数有限，增加额外的转换地址可让 NAT 网关扩展以服务更多用户。

此例中，使用两个地址的池来转换出站数据包。对每条出站连接，PF 会以轮询方式在地址间轮换。

```pf
match out on egress inet nat-to { 192.0.2.5, 192.0.2.10 }
```

此方法的一个缺点是：来自同一内部地址的连续连接不总会被转换到同一转换地址。这可能引发干扰，例如在浏览按 IP 地址跟踪用户登录的网站时。替代方案是使用 `source-hash` 方法，让每个内部地址始终转换到同一转换地址。为此，地址池必须是 [CIDR](https://web.archive.org/web/20150213012421/http://public.swbell.net/dedicated/cidr.html) 网络块。

```pf
match out on egress inet nat-to 192.0.2.4/31 source-hash
```

此规则使用地址池 **192.0.2.4/31**（**192.0.2.4** - **192.0.2.5**）作为出站数据包的转换地址。由于 `source-hash` 关键字，每个内部地址始终转换到同一转换地址。

## 负载均衡入站连接

地址池也可用于负载均衡入站连接。例如，入站 Web 服务器连接可分散到一组 Web 服务器上：

```pf
web_servers = "{ 10.0.0.10, 10.0.0.11, 10.0.0.13 }"

match in on egress proto tcp to port 80 rdr-to $web_servers \
    round-robin sticky-address
```

连续连接会以轮询方式重定向到各 Web 服务器，且来自同一源的连接会发送到同一 Web 服务器。只要存在引用此连接的状态，此"粘性连接"就会持续存在。状态超时后，粘性连接也随之结束。该主机的后续连接会被重定向到轮询中的下一台 Web 服务器。

## 负载均衡出站流量

当未运行适当的多路径路由协议（如 [BGP4](http://www.rfc-editor.org/rfc/rfc1771.txt)）时，可将地址池与 `route-to` 过滤选项结合，对两条或更多互联网连接做负载均衡。`route-to` 配合 `round-robin` 地址池，可将出站连接均匀分布到多条出站路径上。

为此还需要一项额外信息：每条互联网连接上相邻路由器的 IP 地址。此地址喂给 `route-to` 选项，以控制出站数据包的去向。

下例在两条互联网连接间负载均衡出站流量：

```pf
lan_net = "192.168.0.0/24"
int_if  = "dc0"
ext_if1 = "fxp0"
ext_if2 = "fxp1"
ext_gw1 = "198.51.100.100"
ext_gw2 = "203.0.113.200"

pass in on $int_if from $lan_net route-to \
   { ($ext_if1 $ext_gw1), ($ext_if2 $ext_gw2) } round-robin
```

`route-to` 选项用于从*内部*接口*进入*的流量，指定流量将在哪些出站网络接口间均衡，以及各自的网关。注意，`route-to` 选项必须出现在需要均衡流量的*每条*过滤规则上（不能与 `match` 规则联用）。

为确保源地址属于 `$ext_if1` 的数据包始终路由到 `$ext_gw1`（`$ext_if2` 与 `$ext_gw2` 同理），规则集中应包含如下两行：

```pf
pass out on $ext_if1 from $ext_if2 route-to ($ext_if2 $ext_gw2)
pass out on $ext_if2 from $ext_if1 route-to ($ext_if1 $ext_gw1)
```

最后，每个出站接口也可使用 NAT：

```pf
match out on $ext_if1 from $lan_net nat-to ($ext_if1)
match out on $ext_if2 from $lan_net nat-to ($ext_if2)
```

负载均衡出站流量的完整示例可能如下：

```pf
lan_net = "192.168.0.0/24"
int_if  = "dc0"
ext_if1 = "fxp0"
ext_if2 = "fxp1"
ext_gw1 = "198.51.100.100"
ext_gw2 = "203.0.113.200"

#  在每个互联网接口上对出站连接做 NAT
match out on $ext_if1 from $lan_net nat-to ($ext_if1)
match out on $ext_if2 from $lan_net nat-to ($ext_if2)

#  默认拒绝
block in
block out

#  在内部接口上放行所有出站数据包
pass out on $int_if to $lan_net
#  快速放行目标是网关自身的任何数据包
pass in quick on $int_if from $lan_net to $int_if
#  对来自内部网络的出站流量做负载均衡。
pass in on $int_if from $lan_net \
    route-to { ($ext_if1 $ext_gw1), ($ext_if2 $ext_gw2) } \
    round-robin
#  让 https 流量保持在单条连接上；某些 Web 应用，
#  尤其是"安全"应用，不允许会话中途更换连接
pass in on $int_if proto tcp from $lan_net to port https \
    route-to ($ext_if1 $ext_gw1)

#  外部接口的通用 "pass out" 规则
pass out on $ext_if1
pass out on $ext_if2

#  将 $ext_if1 上任意 IP 的数据包路由到 $ext_gw1，
#  $ext_if2 与 $ext_gw2 同理
pass out on $ext_if1 from $ext_if2 route-to ($ext_if2 $ext_gw2)
pass out on $ext_if2 from $ext_if1 route-to ($ext_if1 $ext_gw1)
```
