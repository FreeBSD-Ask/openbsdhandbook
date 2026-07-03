# PF 日志

## 简介

PF 记录数据包时，会将数据包头部的副本连同一些额外数据（如数据包穿越的接口、PF 采取的动作——放行或阻断——等）发送到 pflog 接口。pflog 接口让用户态应用得以从内核接收 PF 的日志数据。系统启动时若启用了 PF，pflogd 守护进程会随之启动。默认情况下，pflogd 监听 `pflog0` 接口，并将所有日志数据写入 **/var/log/pflog** 文件。

## 记录数据包

要记录穿越 PF 的数据包，必须使用 `log` 关键字。`log` 关键字会让所有匹配该规则的数据包都被记录。若规则[创建状态](filter.md#keeping-state)，则仅记录看到的首个数据包（即触发状态创建的那个）。

`log` 关键字可接受的选项如下：

***all***

让所有匹配的数据包都被记录，而不只是首个数据包。对创建状态的规则很有用。

**to** ***pflogN***

让所有匹配的数据包记录到指定的 pflog 接口。例如，使用 [spamlogd(8)](https://man.openbsdhandbook.com/spamlogd.8/) 时，PF 可将所有 SMTP 流量记录到专用 pflog 接口，再让 spamlogd 守护进程监听该接口。这样主 PF 日志文件就不会混入 SMTP 流量——这些流量本不需要记录。使用 ifconfig 创建 pflog 接口。默认日志接口 `pflog0` 会自动创建。

***user***

让数据包源端或目的端套接字（取本地一端）所属的用户 ID 与组 ID，连同标准日志信息一并记录。

选项放在 `log` 关键字后的括号内；多个选项可用逗号或空格分隔。

```pf
pass in log (all, to pflog1) on egress inet proto tcp to egress port 22
```

## 读取日志文件

pflogd 写入的日志文件为二进制格式，无法用文本编辑器读取，须改用 [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)。

查看日志文件：

```
# tcpdump -n -e -ttt -r /var/log/pflog
```

注意，用 tcpdump(8) 查看 pflog 文件*并不*提供实时显示。要实时显示被记录的数据包，使用 `pflog0` 接口：

```
# tcpdump -n -e -ttt -i pflog0
```

**注意**：检查日志时，对 tcpdump 的详细协议解码（通过 `-v` 命令行选项启用）须格外小心。tcpdump 的协议解码器历史上存在安全漏洞。至少在理论上，通过日志设备记录的部分数据包载荷可发起延迟攻击。推荐做法是先将日志文件从防火墙机器移出，再以这种方式检查。

访问日志的权限也须妥善保护。默认情况下，pflogd 会在日志文件中记录数据包的 160 字节。获取日志访问权可能意味着部分敏感数据包载荷的泄露。

## 过滤日志输出

由于 pflogd 以 tcpdump 二进制格式记录，检查日志时可使用 tcpdump 的全部功能。例如，仅查看匹配某端口的数据包：

```
# tcpdump -n -e -ttt -r /var/log/pflog port 80
```

还可进一步限定，只显示特定主机与端口组合的数据包：

```
# tcpdump -n -e -ttt -r /var/log/pflog port 80 and host 192.168.1.3
```

从 `pflog0` 接口读取时也可同样处理：

```
# tcpdump -n -e -ttt -i pflog0 host 192.168.4.2
```

注意，这不会影响哪些数据包被记录到 pflogd 日志文件；上述命令仅在数据包被记录时显示它们。

除使用标准 [tcpdump](https://man.openbsdhandbook.com/tcpdump.8/) 过滤规则外，tcpdump 过滤语言还为读取 pflogd 输出做了扩展：

- `ip` — 地址族为 IPv4。
- `ip6` — 地址族为 IPv6。
- `on *int*` — 数据包穿越了接口 *int*。
- `ifname *int*` — 与 `on *int*` 相同。
- `ruleset *name*` — 数据包匹配所在的[规则集/锚点](anchors.md)。
- `rulenum *num*` — 数据包匹配的过滤规则编号为 *num*。
- `action *act*` — 对数据包采取的动作。可能动作为 `pass` 与 `block`。
- `reason *res*` — 采取该动作的原因。可能原因有 `match`、`bad-offset`、`fragment`、`short`、`normalize`、`memory`、`bad-timestamp`、`congestion`、`ip-option`、`proto-cksum`、`state-mismatch`、`state-insert`、`state-limit`、`src-limit` 与 `synproxy`。
- `inbound` — 数据包为入站。
- `outbound` — 数据包为出站。

示例：

```
# tcpdump -n -e -ttt -i pflog0 inbound and action block and on wi0
```

此命令实时显示在 wi0 接口上被阻断的入站数据包日志。
