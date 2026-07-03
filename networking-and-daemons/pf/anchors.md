# PF 锚点

## 简介

除主规则集外，PF 还可评估子规则集。由于子规则集可借由 pfctl 即时操作，它为动态修改活动规则集提供了便利的方式。[表](tables.md)用于保存动态地址列表，子规则集则用于保存动态规则集合。子规则集通过 `anchor` 附加到主规则集上。

锚点可嵌套，使子规则集得以串联。锚点规则的评估相对于其加载所在的锚点。例如，主规则集中的锚点规则会创建以主规则集为父级的锚点附加点；以 `load anchor` 指令从文件加载的锚点规则，则创建以该锚点为父级的锚点附加点。

## 锚点

锚点是若干规则、表与其他锚点的集合，并赋予一个名称。当 PF 在主规则集中遇到 `anchor` 规则时，会像评估主规则集中的规则那样，评估该锚点中的规则。随后处理会回到主规则集继续，除非数据包匹配了使用 `quick` 选项的过滤规则；这种情况下匹配被视为最终结果，会终止对锚点与主规则集中规则的评估。

例如：

```pf
block on     egress
pass  out on egress

anchor goodguys
```

该规则集在 egress 接口上为进出流量设置默认拒绝策略，然后状态化地放行出站流量，并创建名为 `goodguys` 的锚点规则。可通过三种方式向锚点填充规则：

- 使用 `load` 规则
- 使用 pfctl
- 在主规则集内联指定规则

`load` 规则让 pfctl 从文本文件读取规则，填充指定锚点。`load` 规则必须放在 `anchor` 规则之后。

例如：

```pf
anchor goodguys
load anchor goodguys from "/etc/anchor-goodguys-ssh"
```

若要使用 pfctl 向锚点添加规则，可使用如下命令：

```sh
# echo "pass in proto tcp from 192.0.2.3 to any port 22" | pfctl -a goodguys -f -
```

规则也可保存到文本文件再加载。例如，可将下面两行追加到 **/etc/anchor-goodguys-www** 文件：

```pf
pass in proto tcp from 192.0.2.3 to any port 80
pass in proto tcp from 192.0.2.4 to any port { 80 443 }
```

然后应用：

```sh
# pfctl -a goodguys -f /etc/anchor-goodguys-www
```

若要从主规则集直接加载规则，可将锚点规则放入花括号包围的块中：

```pf
anchor "goodguys" {
        pass in proto tcp from 192.168.2.3 to port 22
}
```

内联锚点也可包含更多锚点。

```pf
allow = "{ 192.0.2.3 192.0.2.4 }"

anchor "goodguys" {
  anchor {
       pass in proto tcp from 192.0.2.3 to port 80
  }
  pass in proto tcp from $allow to port 22
}
```

对于内联锚点，锚点名称可省略。注意上例中嵌套锚点未命名。宏 `$allow` 在锚点之外（即主规则集中）定义，随后在锚点内使用。

加载到锚点中的规则与加载到主规则集中的规则使用相同语法与选项。需要注意：除非使用内联锚点，否则所用[宏](lists-and-macros.md#macros)也必须在锚点内定义；父规则集中定义的宏在锚点中*不可见*。

由于锚点可嵌套，可指定某锚点内所有子锚点都被评估：

```pf
anchor "spam/*"
```

此语法会让附加到 `spam` 锚点的每个锚点中的每条规则都被评估。子锚点按字母顺序评估，但不会递归向下深入。锚点的评估总是相对于其定义所在的锚点。

每个锚点及主规则集彼此独立存在。对某一规则集执行的操作（如清除规则）不会影响其他规则集。此外，从主规则集移除锚点附加点并不会销毁该锚点或其附加的子锚点。只有用 pfctl 清除锚点中的所有规则，且锚点内没有子锚点时，锚点才会被销毁。

## 锚点选项

`anchor` 规则也可选择性地指定接口、协议、源地址、目的地址、标签等，语法与其他规则相同。提供此类信息时，仅当数据包匹配 `anchor` 规则的定义，`anchor` 规则才会被处理。

例如：

```pf
block          on egress
pass       out on egress
anchor ssh in  on egress proto tcp to port 22
```

`ssh` 锚点中的规则仅对从 egress 接口进入、目标端口为 22 的 TCP 数据包评估。随后像这样向锚点添加规则：

```sh
# echo "pass in from 192.0.2.10 to any" | pfctl -a ssh -f -
```

因此，即便过滤规则未指定接口、协议或端口，主机 **192.0.2.10** 也只能通过 SSH 连接，这是 `anchor` 规则定义决定的。

同样的语法也可用于内联锚点。

```pf
allow = "{ 192.0.2.3 192.0.2.4 }"
anchor "goodguys" in proto tcp {
   anchor proto tcp to port 80 {
      pass from 192.0.2.3
   }
   anchor proto tcp to port 22 {
      pass from $allow
   }
}
```

## 操作锚点

锚点操作通过 pfctl 完成。无需重载主规则集即可向锚点添加或移除规则。

列出 `ssh` 锚点中的所有规则：

```sh
# pfctl -a ssh -s rules
```

清除同一锚点中的所有规则：

```sh
# pfctl -a ssh -F rules
```

完整命令列表请见 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)。
