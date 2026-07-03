# PF 列表与宏

## 列表

列表允许在一条规则中指定多个相似条件，例如多个协议、端口号、地址等。因此，不必为每个需阻断的 IP 地址单独写一条过滤规则，可将这些 IP 地址放入列表中用一条规则写出。列表通过在 `{ }` 括号内指定条目来定义。

pfctl 解析配置文件并在加载规则集时遇到列表，会为列表中的每个条目分别创建一条规则。例如：

```pf
block out on fxp0 from { 192.168.0.1, 10.5.32.6 } to any
```

会展开为：

```pf
block out on fxp0 from 192.168.0.1 to any
block out on fxp0 from 10.5.32.6 to any
```

一条规则中可指定多个列表：

```pf
match in on fxp0 proto tcp to port { 22 80 } rdr-to 192.168.0.6
block out on fxp0 proto { tcp udp } from { 192.168.0.1, 10.5.32.6 } \
   to any port { ssh https }
```

列表条目之间的逗号可省略。

列表也可包含嵌套列表：

```pf
trusted = "{ 192.168.1.2 192.168.5.36 }"
pass in inet proto tcp from { 10.10.0.0/24 $trusted } to port 22
```

当心如下结构，被称为"取反列表"，是常见错误：

```pf
pass in on fxp0 from { 10.0.0.0/8, !10.1.2.3 }
```

虽然意图通常是匹配"**10.0.0.0/8** 内除 **10.1.2.3** 外的任何地址"，但此规则会展开为：

```pf
pass in on fxp0 from 10.0.0.0/8
pass in on fxp0 from !10.1.2.3
```

这会匹配任何可能的地址。应改用[表](tables.md)。

## 宏

宏是用户定义的变量，可保存 IP 地址、端口号、接口名等。宏可降低 PF 规则集的复杂度，也使规则集的维护更为轻松。

宏名须以字母开头，可含字母、数字与下划线。宏名不能是 `pass`、`out`、`queue` 等保留字。

```pf
ext_if = "fxp0"

block in on $ext_if from any to any
```

这会创建名为 `ext_if` 的宏。宏创建后被引用时，其名前加 `$` 字符。

宏也可展开为列表，例如：

```
friends = "{ 192.168.1.1, 10.0.2.5, 192.168.43.53 }"
```

宏可递归定义。由于宏在引号内不会展开，须使用如下语法：

```
host1      = "192.168.1.1"
host2      = "192.168.1.2"
all_hosts  = "{" $host1 $host2 "}"
```

宏 `$all_hosts` 现在展开为 192.168.1.1、192.168.1.2。
