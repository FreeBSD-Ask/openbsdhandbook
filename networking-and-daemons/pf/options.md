# PF 选项

## 运行时选项

选项用于控制 PF 的运作。选项在 `pf.conf` 中通过 `set` 指令指定。

#### **set block-policy** ***option***

为指定 `block` 动作的[过滤](filter.md)规则设置默认行为。

- `drop` — 数据包被静默丢弃。
- `return` — 对被阻断的 TCP 数据包返回 TCP RST 数据包，对所有其他数据包返回 ICMP Unreachable 数据包。

注意，单条过滤规则可覆盖默认响应。默认为 `drop`。

#### **set debug** ***option***

设置 pf 的调试级别。可选值包括 `emerg`、`alert`、`crit`、`err`、`warning`、`notice`、`info` 与 `debug`。

#### **set fingerprints** ***file***

设置加载操作系统指纹的文件。供[被动操作系统指纹识别](filter.md#passive-operating-system-fingerprinting)使用。默认为 **/etc/pf.os**。

#### **set limit** ***option value***

设置 pf 运作的各类限制。这些值的当前设置可用 `pfctl -s memory` 查看。

- `frags` — 用于数据包重组（scrub 规则）的内存池中条目数的上限。默认为 5000。
- `src-nodes` — 用于跟踪源 IP 地址（由 `sticky-address` 与 `source-track` 选项生成）的内存池中条目数的上限。默认为 10000。
- `states` — 用于状态表条目（指定 `keep state` 的[过滤](filter.md)规则）的内存池中条目数的上限。默认为 100000。
- `tables` — 可创建的[表](tables.md)数量上限。默认为 1000。
- `table-entries` — 所有表中可存储地址总数的上限。默认为 200000。若系统物理内存少于 100MB，默认设为 100000。

#### **set loginterface** ***interface***

设置 PF 应为其采集统计信息（如进出字节数、放行/阻断的数据包数）的接口。一次只能为*一个*接口采集统计信息。注意，`match`、`bad-offset` 等计数器与状态表计数器不论是否设置 `loginterface` 都会记录。要关闭此选项，设为 `none`。默认为 `none`。

#### **set optimization** ***option***

针对以下网络环境之一优化 PF：

- `normal` — 适用于几乎所有网络。
- `high-latency` — 高延迟网络，如卫星连接。
- `aggressive` — 激进地从状态表中过期连接。这能大幅降低繁忙防火墙的内存需求，但有可能过早丢弃空闲连接。
- `conservative` — 极保守的设置。避免丢弃空闲连接，代价是更高的内存占用与略微增加的处理器占用。

默认为 `normal`。

#### **set ruleset-optimization** ***option***

控制 PF 规则集优化器的运作。

- `none` — 完全禁用优化器。
- `basic` — 启用以下规则集优化：
  1. 移除重复规则
  2. 移除是另一条规则子集的规则
  3. 在有利时将多条规则合并为一张表
  4. 重排规则以提升评估性能
- `profile` — 以当前加载的规则集作为反馈档案，按实际网络流量调整 quick 规则的排序。

自 OpenBSD 4.2 起，默认为 `basic`。更完整的描述请见 [pf.conf](https://man.openbsdhandbook.com/pf.conf.5/)。

#### **set skip on** ***interface***

在 *interface* 上跳过*所有* PF 处理。这在不需要过滤、规范化、排队等的环回接口上很有用。此选项可多次使用。默认未设置。

#### **set state-policy** ***option***

设置 PF 保持状态时的行为。此行为可按规则覆盖。见[保持状态](filter.md#keeping-state)。

- `if-bound` — 状态绑定到其创建所在的接口。若流量匹配某状态表条目，但未穿越该状态条目中记录的接口，则匹配被拒绝。数据包随后必须匹配某条过滤规则，否则会被丢弃/拒绝。
- `floating` — 状态可匹配任意接口上的数据包。只要数据包匹配某状态条目，且通过方向与状态创建时在接口上的方向相同，穿越哪个接口都无所谓。会放行。

默认为 `floating`。

#### **set timeout** ***option value***

设置各类超时（以秒计）。

- `interval` — 清理过期状态与数据包分片之间的间隔秒数。默认为 `10`。
- `frag` — 未重组分片过期前的秒数。默认为 `30`。
- `src.track` — 最后一个状态过期后，[源跟踪](filter.md#stateful-tracking-options)条目在内存中保留的秒数。默认为 `0`。

示例：

```pf
set timeout interval 10
set timeout frag 30
set limit { frags 5000, states 2500 }
set optimization high-latency
set block-policy return
set loginterface dc0
set fingerprints "/etc/pf.os.test"
set skip on lo0
set state-policy if-bound
```
