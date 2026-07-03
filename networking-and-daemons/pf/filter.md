# PF 过滤规则

## 简介

包过滤是指数据包穿越网络接口时有选择地放行或阻断。PF 检查数据包时所依据的标准基于第三层（IPv4 与 IPv6）及第四层（TCP、UDP、ICMP、ICMPv6）头部。最常用的标准是源地址、目的地址、源端口、目的端口与协议。

过滤规则指定数据包须匹配的条件，以及匹配时所采取的动作（阻断或放行）。过滤规则按从上到下的顺序评估。除非数据包匹配了含 `quick` 关键字的规则，否则会针对*所有*过滤规则评估后才采取最终动作。最后匹配的规则为"赢家"，决定对数据包采取何种动作。过滤规则集起始处隐含一条 `pass all`，意味着若数据包未匹配任何过滤规则，最终动作将是放行。

## 规则语法

过滤规则的通用语法（*已大幅简化*）如下：

```ini
action [direction] [log] [quick] [on interface] [af] [proto protocol]
       [from src_addr [port src_port]] [to dst_addr [port dst_port]]
       [flags tcp_flags] [state]
```

***action***

匹配数据包所采取的动作，`pass` 或 `block`。`pass` 动作会将数据包交回内核进一步处理，`block` 动作则依据 [block-policy](options.md#set-block-policy-option) 选项的设置作出反应。默认反应可通过指定 `block drop` 或 `block return` 覆盖。

***direction***

数据包在接口上的流向，`in` 或 `out`。

***log***

指定数据包应通过 pflogd 记录。若规则创建状态，则仅记录建立状态的那个数据包。要无条件记录所有数据包，使用 `log (all)`。

***quick***

若数据包匹配含 `quick` 的规则，该规则即被视为最后匹配的规则，并执行指定的*动作*。

***interface***

数据包穿越的网络接口名称或组。接口可通过 ifconfig 命令加入任意组。内核也会自动创建若干组：

- `egress` 组，包含持有默认路由的接口。
- 克隆接口的接口族组。例如 `ppp` 或 `carp`。

这会让规则分别匹配穿越任意 `ppp` 或 `carp` 接口的数据包。

***af***

数据包的地址族，`inet` 表示 IPv4，`inet6` 表示 IPv6。PF 通常能根据源地址和/或目的地址推断此参数。

***protocol***

数据包的第四层协议：

- `tcp`
- `udp`
- `icmp`
- `icmp6`
- **/etc/protocols** 中的有效协议名
- 0 到 255 之间的协议号
- 使用[列表](lists-and-macros.md#lists)指定的一组协议

***src\_addr***, ***dst\_addr***

IP 头部中的源地址/目的地址。地址可指定为：

- 单个 IPv4 或 IPv6 地址。
- [CIDR](https://web.archive.org/web/20150213012421/http://public.swbell.net/dedicated/cidr.html) 网络块。
- 完整限定域名，加载规则集时通过 DNS 解析。所有解析得到的 IP 地址都会替换进规则。
- 网络接口或组的名称。接口上的所有 IP 地址都会替换进规则。
- 网络接口名后跟 `/*netmask*`（如 `/24`）。接口上的每个 IP 地址与网络掩码组合成 CIDR 网络块，替换进规则。
- 网络接口或组名放在括号 `( )` 中。这告诉 PF：当命名接口上的 IP 地址变化时更新规则。这对通过 DHCP 或拨号获取 IP 地址的接口很有用，无需每次地址变化都重载规则集。
- 网络接口名后跟以下任一修饰符：

  - `:network` — 替换为 CIDR 网络块（如 **192.168.0.0/24**）
  - `:broadcast` — 替换为网络广播地址（如 **192.168.0.255**）
  - `:peer` — 替换为点对点链路上对端的 IP 地址

    此外，`:0` 修饰符可附加在接口名或上述任一修饰符之后，表示 PF 不应将别名 IP 地址纳入替换。接口放在括号中时也可使用这些修饰符。例：`fxp0:network:0`
- [表](tables.md)。
- 关键字 `urpf-failed` 可用于源地址，表示应执行 [uRPF 检查](#urpf)。
- 上述任一项前加 `!`（"非"）修饰符取反。
- 使用[列表](lists-and-macros.md#lists)指定的一组地址。
- 关键字 `any`，表示所有地址。
- 关键字 `all`，是 `from any to any` 的简写。

***src\_port***, ***dst\_port***

第四层包头中的源端口/目的端口。端口可指定为：

- 1 到 65535 之间的数字
- [**/etc/services**](https://man.openbsdhandbook.com/services.5/) 中的有效服务名
- 使用[列表](lists-and-macros.md#lists)指定的一组端口
- 范围：
  - `!=`（不等于）
  - `<`（小于）
  - `>`（大于）
  - `<=`（小于或等于）
  - `>=`（大于或等于）
  - `><`（范围）
  - `<>`（反向范围）

    最后两个是二元运算符（接受两个参数），且不包含参数本身。
  - `:`（闭区间）

    闭区间运算符也是二元运算符，且包含参数本身。

***tcp\_flags***

使用 `proto tcp` 时，指定 TCP 头部中必须置位的标志。标志以 `flags *check*/*mask*` 形式指定。例如：`flags S/SA` —— 指示 PF 仅检查 S 与 A（SYN 与 ACK）标志，且仅当 SYN 标志"置位"时匹配（默认应用于所有 TCP 规则）。`flags any` 指示 PF 不检查标志。

***state***

指定是否为匹配此规则的数据包保留状态信息。

- `no state` — 适用于 TCP、UDP、ICMP。PF 不会以状态化方式跟踪此连接。TCP 连接通常还需配合 `flags any`。
- `keep state` — 适用于 TCP、UDP、ICMP。此选项是所有过滤规则的默认值。
- `modulate state` — 仅适用于 TCP。PF 会为匹配此规则的数据包生成强初始序列号（ISN）。
- `synproxy state` — 代理入站 TCP 连接，帮助保护服务器免受伪造的 TCP SYN 洪水攻击。此选项包含 `keep state` 与 `modulate state` 的功能。

## 默认拒绝

搭建防火墙的推荐做法是采用"默认拒绝"策略，即先拒绝*一切*，再选择性地放行某些流量通过防火墙。此策略之所以推荐，是因为它宁慎勿纵，也让编写规则集更轻松。

要创建默认拒绝的过滤策略，首条过滤规则应为：

```
block all
```

这条规则会阻断所有接口上、任一方向、任意源到任意目的的所有流量。

## 放行流量

流量必须显式放行才能穿过防火墙，否则会被默认拒绝策略丢弃。源/目的端口、源/目的地址、协议等数据包条件就在此发挥作用。每当允许流量穿过防火墙时，规则应写得尽可能严格，以确保只有预期流量得以放行。

若干示例：

```pf
# 放行从本地网络 192.168.0.0/24 进入 dc0 到 OpenBSD 机器 IP 地址
# 192.168.0.1 的流量。同时放行 dc0 上的返回流量。
pass in  on dc0 from 192.168.0.0/24 to 192.168.0.1
pass out on dc0 from 192.168.0.1    to 192.168.0.0/24

# 放行进入 OpenBSD 机器上 Web 服务器的 TCP 流量。
pass in on egress proto tcp from any to egress port www
```

## `quick` 关键字

如前所述，每个数据包都从上到下针对过滤规则集评估。默认情况下，数据包被标记为放行，这条标记可被任何规则改变，且在到达过滤规则末尾前可能反复改变。**最后匹配的规则取胜。** 此原则有一例外：过滤规则上的 `quick` 选项会取消所有后续规则处理，并立即执行指定动作。来看几个例子：

错误：

```pf
block in on egress proto tcp to port ssh
pass  in all
```

此例中 `block` 行或许会被评估，但永远不会生效，因为紧随其后是一条放行一切的行。

更佳：

```pf
block in quick on egress proto tcp to port ssh
pass  in all
```

这些规则的评估略有不同。若 `block` 行匹配，由于 `quick` 选项，数据包会被阻断，规则集其余部分被忽略。

## 保持状态

PF 的一项重要能力是"保持状态"或称"状态化检测"。状态化检测指 PF 跟踪网络连接的状态或进展的能力。通过在状态表中保存每条连接的信息，PF 能迅速判断穿越防火墙的数据包是否属于已建立的连接。若是，则不经规则集评估直接放行。

保持状态有诸多优点，包括更简洁的规则集与更佳的包过滤性能。PF 能将*任一*方向的数据包匹配到状态表条目，意味着不必再编写放行返回流量的过滤规则。由于匹配状态化连接的数据包不必经过规则集评估，PF 处理这些数据包的时间可大幅缩短。

当规则创建状态时，匹配该规则的首个数据包会在发送方与接收方之间建立"状态"。如今，不仅包防火墙属于已建立连接。若是如此，则不经规则集评估直接穿过防火墙。

保持状态有诸多优点，包括更简洁的规则集与更佳的包过滤性能。PF 能将*任一*方向的数据包匹配到状态表条目egress proto tcp from any to any

此规则允许 egress 接口上的任意出站 TCP 流量，并允许回复流量穿过防火墙返回。保持状态可显著提升防火墙性能，因为状态查找远比让数据包经过过滤规则快。

`modulate state` 选项与 `keep state` 类似，区别是仅适用于 TCP 数据包。使用 `modulate state` 时，出站连接的初始序列号（ISN）会被随机化。这对保护由某些不擅长选择 ISN 的操作系统发起的连接很有用。为简化规则集，`modulate state` 选项可用于指定 TCP 以外协议的规则；这种情况下视为 `keep state`。

为出站 TCP、UDP、ICMP 数据包保持状态，并随机化 TCP ISN：

```pf
pass out on egress proto { tcp, udp, icmp } from any to any modulate state
```

保持状态的另一优点是相关的 ICMP 流量也会穿过防火墙。例如，若穿越防火墙的 TCP 连接正被状态化跟踪，而一条引用此 TCP 连接的 ICMP source-quench 消息到达，它会被匹配到相应的状态条目并穿过防火墙。

状态条目的作用域由 [state-policy](options.md#set-state-policy-option) 运行时选项全局控制，并可通过 `if-bound` 与 `floating` 状态选项关键字按规则控制。这些按规则关键字与 `state-policy` 选项联用时的含义相同。例如：

```pf
pass out on egress proto { tcp, udp, icmp } from any to any modulate state (if-bound)
```

此规则规定：数据包若要匹配该状态条目，必须穿越 egress 接口。

## 保持 UDP 状态

有时会听到一种说法："UDP 是无状态协议，无法用 UDP 创建状态！"虽然 UDP 通信会话确实没有状态概念（明确的通信开始与结束），但这并不影响 PF 为 UDP 会话创建状态的能力。对于没有"开始"与"结束"数据包的协议，PF 只跟踪自上次匹配数据包穿越以来经过的时长。超时后状态即被清除。超时值可在 **pf.conf** 文件的[选项](options.md#set-timeout-option-value)部分设置。

## 状态化跟踪选项

创建状态条目的过滤规则可指定若干选项，控制生成的状态条目的行为。可用选项如下：

***max \_number***

将此规则可创建的状态条目数上限限制为 *number*。达到上限后，本会创建状态的数据包将不再匹配此规则，直到现有状态数降至上限以下。

***no state***

阻止规则自动创建状态条目。

***source-track***

此选项启用对每个源 IP 地址创建状态数的跟踪。此选项有两种形式：

- `source-track rule` — 此规则创建的状态数上限受该规则的 `max-src-nodes` 与 `max-src-states` 选项限制。仅此特定规则创建的状态条目计入该规则限额。
- `source-track global` — 所有用此选项的规则创建的状态总数受限。每条规则可指定不同的 `max-src-nodes` 与 `max-src-states` 选项，但任一参与规则创建的状态条目都计入各规则各自的限额。

全局跟踪的源 IP 地址总数可通过 [`src-nodes` 运行时选项](options.md#set-limit-option-value)控制。

***max-src-nodes \_number***

使用 `source-track` 选项时，`max-src-nodes` 限制可同时创建状态的源 IP 地址数。此选项只能与 `source-track rule` 联用。

***max-src-states \_number***

使用 `source-track` 选项时，`max-src-states` 限制每个源 IP 地址可同时创建的状态条目数。此限制的作用域（即仅限此规则创建的状态，还是所有使用 `source-track` 的规则创建的状态）取决于所指定的 `source-track` 选项。

选项放在括号内，紧跟在某个状态关键字（`keep state`、`modulate state` 或 `synproxy state`）之后。多个选项以逗号分隔。`keep state` 选项是所有过滤规则的隐式默认。尽管如此，指定状态化选项时，仍须在选项前使用某个状态关键字。

示例规则：

```pf
pass in on egress proto tcp to $web_server port www keep state   \
                  (max 200, source-track rule, max-src-nodes 100, \
                   max-src-states 3)
```

上面的规则定义了如下行为：

- 此规则可创建的状态数绝对上限为 200。
- 启用源跟踪：仅根据此规则创建的状态限制状态创建。
- 可同时创建状态的节点数上限为 100。
- 每个源 IP 的同时状态数上限为 3。

对已完成三次握手的状态化 TCP 连接，可施加另一组限制。

**max-src-conn** ***number***

限制单个主机可同时建立的、已完成三次握手的 TCP 连接数上限。

**max-src-conn-rate** ***number*** / ***interval***

限制每个时间间隔内新连接的速率为指定数量。

这两个选项都会自动启用 `source-track rule` 选项，且与 `source-track global` 不兼容。

由于这些限制仅施加于已完成三次握手的 TCP 连接，可对违规 IP 地址采取更激进的措施。

**overload** ***<*table*>***

将违规主机的 IP 地址放入指定表。

**flush** ***global***

杀掉此规则匹配的、且由该源 IP 创建的其他所有状态。指定 `global` 时，杀掉匹配此源 IP 的所有状态，不论由哪条规则创建。

示例：

```pf
table <abusive_hosts> persist
block in quick from <abusive_hosts>

pass in on egress proto tcp to $web_server port www flags S/SA keep state \
                                (max-src-conn 100, max-src-conn-rate 15/5, \
                                 overload <abusive_hosts> flush)
```

此举达成如下效果：

- 每个源的连接数上限为 100。
- 5 秒内连接数速率限制为 15。
- 任何突破这些限制的主机 IP 地址都会被放入 `<abusive_hosts>` 表。
- 对任何违规 IP 地址，清除此规则创建的所有状态。

## TCP 标志

基于标志匹配 TCP 数据包，最常用于过滤试图打开新连接的 TCP 数据包。TCP 标志及其含义如下：

- **F** : FIN — Finish；结束会话
- **S** : SYN — Synchronize；表示请求开始会话
- **R** : RST — Reset；断开连接
- **P** : PUSH — Push；数据包立即发送
- **A** : ACK — Acknowledgement；确认
- **U** : URG — Urgent；紧急
- **E** : ECE — Explicit Congestion Notification Echo；显式拥塞通知回显
- **W** : CWR — Congestion Window Reduced；拥塞窗口缩减

要让 PF 在规则评估时检查 TCP 标志，使用 `flags` 关键字，语法如下：

```
flags _check_/_mask_
flags any
```

*mask* 部分告诉 PF 仅检查指定标志，*check* 部分指定头部中哪些标志必须"置位"才能匹配。使用 `any` 关键字允许头部中任意标志组合。

```pf
pass in on egress proto tcp from any to any port ssh flags S/SA
pass in on egress proto tcp from any to any port ssh
```

由于 `flags S/SA` 是默认设置，上述两条规则等价，都放行置 SYN 标志的 TCP 流量，且仅检查 SYN 与 ACK 标志。带 SYN 与 ECE 标志的数据包会匹配上述规则，而带 SYN 与 ACK 或仅带 ACK 的数据包则不会。

使用 `flags` 选项可覆盖默认标志，方法如上。

使用标志时须谨慎——理解自己在做什么以及为什么，对他人的建议也要小心，因为很多建议并不可靠。有人建议"仅当 SYN 标志置位且无其他标志时才创建状态"。这样的规则会以如下形式结尾：

```ini
[...] flags S/FSRPAUEW  _bad idea!!_
```

其理论依据是仅在 TCP 会话开始时创建状态，而会话应以 SYN 标志开始，无其他标志。问题在于，部分站点开始使用 ECN 标志，任何使用 ECN 的站点尝试连接你时都会被此规则拒绝。更好的做法是根本不指定任何标志，让 PF 应用默认标志。若确实需要自行指定标志，下列组合应是安全的：

```ini
[...] flags S/SAFR
```

虽然这既实用又安全，但如果流量同时被规范化，则检查 FIN 与 RST 标志其实并无必要。规范化过程会让 PF 丢弃任何带有非法 TCP 标志组合（如 SYN 与 RST）的入站数据包，并规范化可能产生歧义的组合（如 SYN 与 FIN）。

## TCP SYN 代理

通常，当客户端向服务器发起 TCP 连接时，PF 会将[握手](http://www.inetdaemon.com/tutorials/internet/tcp/3-way_handshake.shtml)数据包按到达顺序在两端之间转发。但 PF 也能代理握手。握手被代理后，PF 自身会与客户端完成握手，再向服务器发起握手，随后在两端之间转发数据包。在 TCP SYN 洪水攻击中，攻击者永远不会完成三次握手，因此攻击者的数据包永远不会到达受保护的服务器，而合法客户端会完成握手并被放行。这将在 PF 内部处理，最小化伪造的 TCP SYN 洪水对受保护服务的影响。但不建议常规使用此选项，因为在服务器无法处理请求或涉及负载均衡器时，它会破坏预期的 TCP 协议行为。

TCP SYN 代理通过过滤规则中的 `synproxy state` 关键字启用。例如：

```pf
pass in on egress proto tcp to $web_server port www synproxy state
```

此处，到 Web 服务器的连接将由 PF 进行 TCP 代理。

由于 `synproxy state` 的工作方式，它也包含 `keep state` 与 `modulate state` 的功能。

PF 在网桥上运行时，SYN 代理不工作。

## 阻断伪造数据包

地址"伪造"是指恶意用户在发送的数据包中假冒源 IP 地址，以隐藏真实地址或冒充网络上的其他节点。一旦用户伪造了地址，便可在不暴露攻击真实源头的情况下发起网络攻击，或试图访问仅限特定 IP 地址的网络服务。

PF 通过 `antispoof` 关键字提供针对地址伪造的某些保护：

```pf
antispoof [log] [quick] for interface [af]
```

***log***

指定匹配的数据包应通过 pflogd 记录。

***quick***

若数据包匹配此规则，则被视为"获胜"规则，规则集评估停止。

***interface***

启用伪造保护的网络接口。也可以是接口的[列表](lists-and-macros.md#lists)。

***af***

启用伪造保护的地址族，`inet` 表示 IPv4，`inet6` 表示 IPv6。

示例：

```pf
antispoof for fxp0 inet
```

加载规则集时，所有 `antispoof` 关键字的出现都会被展开为两条过滤规则。假设 egress 接口 IP 地址为 **10.0.0.1**，子网掩码为 **255.255.255.0**（即 /24），上面的 `antispoof` 规则会展开为：

```pf
block in on ! fxp0 inet from 10.0.0.0/24 to any
block in inet from 10.0.0.1 to any
```

这些规则达成两件事：

- 阻断来自 **10.0.0.0/24** 网络但*未*从 `fxp0` 接口进入的所有流量。由于 **10.0.0.0/24** 网络位于 `fxp0` 接口上，源地址属于该网络块的数据包绝不应从其他接口进入。
- 阻断来自 **10.0.0.1**（`fxp0` 上的 IP 地址）的所有入站流量。主机不应通过外部接口向自身发送数据包，因此任何源地址属于本机的入站数据包都可视为恶意。

**注意**：`antispoof` 规则展开后的过滤规则也会阻断通过环回接口发往本地地址的数据包。最佳实践本就应跳过环回接口的过滤，使用 antispoof 规则时则成为必需：

```pf
set skip on lo0
antispoof for fxp0 inet
```

`antispoof` 应仅用于已分配 IP 地址的接口。在未分配 IP 地址的接口上使用 `antispoof` 会生成如下过滤规则：

```
block drop in on ! fxp0 inet all
block drop in inet all
```

这些规则可能导致阻断*所有*接口上的*所有*入站流量。

## 单播反向路径转发

PF 提供单播反向路径转发（uRPF）特性。当数据包经过 uRPF 检查时，会在路由表中查找该数据包的源 IP 地址。若路由表条目中找到的出站接口与数据包刚刚进入的接口相同，则 uRPF 检查通过。若接口不匹配，则该数据包的源地址可能已被伪造。

可在过滤规则中使用 `urpf-failed` 关键字对数据包执行 uRPF 检查：

```pf
block in quick from urpf-failed label uRPF
```

注意，uRPF 检查仅在路由对称的环境下才有意义。

uRPF 提供与 [antispoof](filter.md#blocking-spoofed-packets) 规则相同的功能。

## 被动操作系统指纹识别

被动操作系统指纹识别（OSFP）是一种根据远程主机 TCP SYN 数据包中的某些特征被动检测其操作系统的方法。此信息可用作过滤规则中的条件。

PF 将 TCP SYN 数据包的特征与[指纹文件](options.md#set-fingerprints-file)对比来判断远程操作系统，默认指纹文件为 [pf.os](https://man.openbsdhandbook.com/pf.os.5/)。PF 启用后，可用如下命令查看当前指纹列表：

```sh
# pfctl -s osfp
```

在过滤规则中，可按操作系统类别、版本或子类型/补丁级别指定指纹。这些项均列于上文 `pfctl` 命令的输出中。要在过滤规则中指定指纹，使用 `os` 关键字：

```pf
pass  in on egress proto tcp from any os OpenBSD
block in on egress proto tcp from any os "Windows 2000"
block in on egress proto tcp from any os "Linux 2.4 ts"
block in on egress proto tcp from any os unknown
```

特殊操作系统类别 `unknown` 用于在操作系统指纹未知时匹配数据包。

**请注意**以下事项：

- 由于伪造或精心构造的数据包被做得像来自特定操作系统，操作系统指纹偶尔会出错。
- 操作系统的某些修订版或补丁级别可能改变协议栈行为，导致其要么不匹配指纹文件中的条目，要么匹配到完全不同的条目。
- OSFP 仅对 TCP SYN 数据包有效；对其他协议或已建立的连接无效。

## IP 选项

默认情况下，PF 阻断设置了 IP 选项的数据包。这会让 nmap 等操作系统指纹识别工具的工作更为困难。若有应用需要放行此类数据包（如多播或 IGMP），可使用 `allow-opts` 指令：

```pf
pass in quick on fxp0 all allow-opts
```

## 过滤规则集示例

下面是一个过滤规则集示例。运行 PF 的机器作为防火墙，连接一个小型内部网络与互联网。此处仅展示过滤规则；排队、[nat](nat.md)、[rdr](https://www.openbsdhandbook.com/pf/rdr) 等在此例中略去。

```pf
int_if  = "dc0"
lan_net = "192.168.0.0/24"

# 包含分配给防火墙的所有 IP 地址的表
table <firewall> const { self }

# 不在环回接口上过滤
set skip on lo0

# 规范化入站数据包
match in all scrub (no-df)

# 设置默认拒绝策略
block all

# 为所有接口启用伪造保护
block in quick from urpf-failed

# 仅允许来自可信计算机 192.168.0.15 的本地网络 SSH 连接。
# 使用 "block return" 以便立即发送 TCP RST 关闭被阻断的连接。
# 使用 "quick" 以免此规则被下方的 "pass" 规则覆盖。
block return in quick on $int_if proto tcp from ! 192.168.0.15 to $int_if port ssh

# 放行进出本地网络的所有流量。
# 由于默认 "keep state" 选项会自动应用，这些规则会创建状态条目。
pass in  on $int_if from $lan_net
pass out on $int_if to   $lan_net

# 在外部（互联网）接口上放行 tcp、udp、icmp 出站流量。
# tcp 连接会被调制，udp/icmp 会被状态化跟踪。
pass out on egress proto { tcp udp icmp } all modulate state

# 允许外部接口上的 SSH 连接进入，前提是目标不是防火墙本身
# （即目标是本地网络上的机器）。记录初始数据包以便事后查证谁在尝试连接。
# 取消最后一行的注释即可使用 tcp syn 代理来代理连接。
pass in log on egress proto tcp to ! <firewall> port ssh # synproxy state
```
