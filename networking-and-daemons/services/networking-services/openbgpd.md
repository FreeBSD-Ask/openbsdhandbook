# OpenBGPD

## 概述

本章提供在 OpenBSD 上使用基本系统守护进程 [bgpd(8)](https://man.openbsdhandbook.com/bgpd.8/) 运行 BGP 的规范、面向生产的模式。内容涵盖 [bgpd.conf(5)](https://man.openbsdhandbook.com/bgpd.conf.5/) 中的配置、使用 [bgpctl(8)](https://man.openbsdhandbook.com/bgpctl.8/) 进行运行时操作、通过 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/) 控制服务，以及使用 [rpki-client(8)](https://man.openbsdhandbook.com/rpki-client.8/) 进行可选的源校验。这些模式适用于外部对等、多供应商边缘、站点或 POP 内的 iBGP，以及通过团体属性与本地优先级表达策略。

## 设计要点

- **策略优先。** 默认拒绝，显式放行预期的导入与通告。过滤器紧邻邻居配置，全局默认保持保守。
- **确定性优先级。** 在 AS 内为入站路径选择设置 `localpref`；使用团体属性进行结构化的策略信令。
- **规模边界。** 小型组网中全互联 iBGP 即可；拓扑较大时引入路由反射器，并尽量精简反射器状态。
- **RPKI。** 对接收路由的源进行校验；丢弃 `roa invalid`。调度 [rpki-client(8)](https://man.openbsdhandbook.com/rpki-client.8/) 刷新 ROA。
- **可观测性。** 将 `bgpctl show summary`、`show neighbor` 与 `show rib {in,out}` 视为每次变更与故障处置流程的必备环节。
- **变更控制。** 调整上游会话时，分阶段编辑、做语法检查，并保留回滚路径（控制台或定时回退）。

## 配置

### 1) 服务生命周期与骨架

创建包含本地 AS 与路由器 ID 的最小骨架，并启用守护进程。

```
## /etc/bgpd.conf — 最小骨架
AS 64512
router-id 198.51.100.2
```

```sh
# rcctl enable bgpd
  # 开机启动
# bgpd -n -f /etc/bgpd.conf
  # 语法检查（不启动）
# rcctl start bgpd
  # 启动守护进程
```

### 2) 单宿 eBGP 接入上游（通告特定前缀）

前提：

- 公共前缀：**198.51.100.0/24**、**2001:db8:100::/48**
- 上游邻居：**203.0.113.1**，邻居 AS `64500`
- eBGP 源地址：**198.51.100.2**

```
## /etc/bgpd.conf — 单 ISP eBGP，显式出站与受控入站
AS 64512
router-id 198.51.100.2

# 待通告的前缀
prefix-set "our-net" {
    198.51.100.0/24,
    2001:db8:100::/48
}

neighbor 203.0.113.1 {
    remote-as 64500
    descr "ISP-A"
    local-address 198.51.100.2
    enforce neighbor-as yes
    max-prefix 500000 restart        # 按供应商指引调整
    announce none                     # 出站由下方 match 规则控制
}

# 入站：接受上游路由（受 max-prefix 与后续过滤器约束）
allow from neighbor 203.0.113.1

# 出站：仅向该邻居通告已批准的前缀
match to neighbor 203.0.113.1 prefix-set "our-net" announce
```

重载并验证：

```sh
# bgpctl reload
  # 平滑重配置
$ bgpctl show summary
  # 会话状态、前缀计数、逐邻居统计
$ bgpctl show neighbor 203.0.113.1
  # 能力与最近错误（若有）
$ bgpctl show rib out neighbor 203.0.113.1
  # 应仅通告已批准的前缀
```

### 3) 双宿 eBGP（用 localpref 偏向一个上游；出站打团体属性）

前提：

- 第二上游：**198.51.100.1**，AS `64496`
- 偏好：来自 `ISP-A` 的路由应通过更高 `localpref` 优先于 `ISP-B`
- 出站标记：在通告的路由上设置参考团体属性（仅作示例）

```
## /etc/bgpd.conf — 两家供应商；确定性偏好与出站标记
AS 64512
router-id 198.51.100.2

prefix-set "our-net" { 198.51.100.0/24, 2001:db8:100::/48 }

neighbor 203.0.113.1 {
    remote-as 64500
    descr "ISP-A"
    local-address 198.51.100.2
    enforce neighbor-as yes
    max-prefix 500000 restart
    announce none
}

neighbor 198.51.100.1 {
    remote-as 64496
    descr "ISP-B"
    local-address 198.51.100.2
    enforce neighbor-as yes
    max-prefix 500000 restart
    announce none
}

# 接受双方的入站
allow from neighbor 203.0.113.1
allow from neighbor 198.51.100.1

# AS 内偏好 ISP-A
match from neighbor 203.0.113.1 set localpref 200
match from neighbor 198.51.100.1 set localpref 150

# 出站：向双方通告同一集合，并附加参考团体属性
match to neighbor 203.0.113.1 prefix-set "our-net" set community 64512:100 announce
match to neighbor 198.51.100.1 prefix-set "our-net" set community 64512:100 announce
```

验证重点：

```sh
$ bgpctl show rib as 64500
  # 从 ISP-A 学到的路由（AS 路径视图）
$ bgpctl show rib | egrep '198\.51\.100\.0/24|2001:db8:100::/48'
  # 导出前存在于 RIB 中的始发前缀
$ bgpctl show neighbor | egrep 'LocalPref|Community'
  # 策略标记按预期可见
```

> 通告要求路由已存在于路由表中（例如通过静态路由或回环分配）。在 `bgpd` 导出之前，须先安装确定性的始发路由（服务 IP 用主机 /32 或 /128，地址块用聚合路由）。

### 4) AS 内 iBGP（小型组网，全互联）

前提：

- 两台路由器：`R1`（本机）与位于 **10.0.0.2** 的 `R2`，均属 AS `64512`
- 全互联 iBGP；通过对从该 ISP 学到的路由施加更高 `localpref`，使 R1 作为通往 `ISP-A` 的出口优先

```
## /etc/bgpd.conf — 添加一个简单 iBGP 对等体
AS 64512
router-id 10.0.0.1

neighbor 10.0.0.2 {
    remote-as 64512
    descr "iBGP-R2"
    announce none
}

# iBGP 接受
allow from neighbor 10.0.0.2

# 可选：将交换限制为内部路由
prefix-set "internal" { 10.10.0.0/16, 10.20.0.0/16 }
match to neighbor 10.0.0.2 prefix-set "internal" announce
```

验证重点：

```sh
$ bgpctl show summary
  # iBGP 会话应为 Established
$ bgpctl show rib neighbor 10.0.0.2
  # 从对等体接受的路由
$ bgpctl show rib out neighbor 10.0.0.2
  # 向对等体通告的路由（经 "internal" 过滤）
```

> 拓扑较大时，引入路由反射器以避免全互联。反射器应尽量简单，并将重新通告限制在必需的地址族与范围内。

### 5) RPKI 源校验（使用 rpki-client）

定期运行 [rpki-client(8)](https://man.openbsdhandbook.com/rpki-client.8/) 获取并校验 ROA，再由 `bgpd` 加载生成的集合。按策略拒绝 `roa invalid`。

```sh
# rcctl enable rpki_client
# rcctl start rpki_client
  # 启动周期性 ROA 获取/校验；产物位于 /var/db/rpki-client/
# rpki-client -vv
  # 按需运行，用于初始填充与诊断
```

bgpd 配置：

```pf
## /etc/bgpd.conf — 加载 ROA 并丢弃无效项
AS 64512
router-id 198.51.100.2

roa-set {
    /var/db/rpki-client/openbgpd
}

# 策略：不接受按 ROA 判定为无效源的路由
deny from any roa invalid

# 其余导入按邻居规则放行
```

观察 RPKI 状态：

```sh
$ bgpctl show rib roa invalid
  # 因无效源被拒绝的路由
$ bgpctl show rib roa unknown | head
  # 无覆盖 ROA 的路由（仅供参考）
```

### 6) AS 内任播服务通告

对于内部任播服务（例如解析器 **10.10.255.53/32**），在每个服务节点上始发该地址（lo0 别名或静态路由），并通过 iBGP 通告。随后由 ECMP 或策略引导客户端。

```
## /etc/bgpd.conf — 通过 iBGP 始发任播主机路由
AS 64512
router-id 10.0.0.1

network 10.10.255.53/32
  # 导出前路由须已存在于 RIB（lo0 别名或静态）
```

## 验证

使用 [bgpctl(8)](https://man.openbsdhandbook.com/bgpctl.8/) 的核心命令：

```sh
$ bgpctl show summary
  # 会话状态与计数器
$ bgpctl show neighbor <addr>
  # 能力、定时器、最近错误
$ bgpctl show rib
  # Loc-RIB；附加 'in' 或 'out' 配合 'neighbor <addr>' 查看逐对等体视图
$ bgpctl show rib community 64512:100
  # 按团体属性过滤；便于策略检查
$ bgpctl log verbose
  # 排障时临时提高 bgpd(8) 日志详细度
```

链路层与系统检查：

```sh
# tcpdump -ni egress port 179
  # 观察链路上的 BGP OPEN/KEEPALIVE/UPDATE
$ route -n show -inet ; route -n show -inet6
  # 已通告前缀对应的始发/静态路由是否存在
```

## 排障

- **会话卡在 Idle/Active。** 确认 IP 可达性、`local-address`，以及 PF 放行 TCP/179。验证 `remote-as` 与对端配置的本地 AS 一致，且 `enforce neighbor-as` 未阻断预期路径。
- **无通告发出。** 路由须已存在于 RIB（`network` 语句或静态路由）。确认有显式 `match ... announce` 规则指向该邻居。检查 `bgpctl show rib out neighbor ...`。
- **对等体发来过多路由。** 提高或添加 `max-prefix`，配 `restart` 与 `warning` 阈值。用 prefix-list 或 `prefix-set` 过滤器收窄范围。
- **入站路径未获偏好。** 检查接收路由的 `localpref`。确认是否有更具体路由或 MED 覆盖了预期意图。
- **RPKI 丢弃了意外路由。** 检查 `bgpctl show rib roa invalid`，并与上级 ROA 仓库的当前 ROA 交叉核对。放宽策略前先排查 ROA 状态。
- **重载后抖动。** 使用 `bgpctl reload` 进行平滑策略更新。结构性变更（增删对等体）应在维护窗口或非高峰时段执行。

## 另请参阅

- [网络](../../../install-and-configure/networking.md)
- [高级网络](../../advanced-networking/README.md)
- 手册页：[bgpd(8)](https://man.openbsdhandbook.com/bgpd.8/)
  、[bgpd.conf(5)](https://man.openbsdhandbook.com/bgpd.conf.5/)
  、[bgpctl(8)](https://man.openbsdhandbook.com/bgpctl.8/)
  、[rpki-client(8)](https://man.openbsdhandbook.com/rpki-client.8/)
- 相关：[MPLS 与标签分发](../../advanced-networking/mpls.md)
- 相关：[参考架构](../../advanced-networking/reference-architectures.md)
