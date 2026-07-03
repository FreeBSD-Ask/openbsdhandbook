# 加固与运维安全

## 概述

本章提供 OpenBSD 网络系统的加固默认值与安全运维规程。内容涵盖：[pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/) 中的反伪造与反向路径过滤；用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/) 进行有纪律的规则部署；用 [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/) 进行最小权限管理；通过 [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/) 安全暴露管理平面；守护进程的密钥文件卫生（例如 [iked.conf(5)](https://man.openbsdhandbook.com/iked.conf.5/)）；以及用 [syspatch(8)](https://man.openbsdhandbook.com/syspatch.8/) 打补丁。强调可预测的变更、回滚路径与可审计的配置。

## 设计考量

- **威胁模型。** 首要保护管理平面。限制管理来源，密钥优先于密码，对策略未要求的全部东西向流量默认拒绝。
- **伪造与路径对称。** 用基于接口的反伪造与单播 RPF 检查强制源地址有效性。
- **规则安全。** 通过语法检查与临时回滚计划验证并分阶段部署 PF 变更。维护一个故障安全锚点，在部署期间保留带外访问。
- **密钥与归属。** 密钥与 PSK 仅 root 可读。优先采用守护进程自带的 chroot 与权限分离。
- **补丁与溯源。** 及时应用勘误。将配置纳入版本控制，并在提交中记录变更意图。
- **可操作日志。** 将日志转发出主机；保持本地轮转合理，并在关联事件前校准时钟。

## 配置

### 1) 带反伪造、uRPF 与 bogon 表的加固 PF 骨架

下例是一个保守的基线。假定 `em0` 为 WAN，`em1`/`em2` 为内部接口。它拒绝明显伪造的源地址，并阻断在接收接口上未通过反向路径检查的数据包。

```pf
## /etc/pf.conf — 带伪造控制的加固基线

set skip on lo

# 接口
wan   = "em0"
inside= "{ em1, em2 }"

# -------------------------
# 表
# -------------------------
# 常见伪造 IPv4 源（按需调整；在运维文档中保持最新）
table <bogons4> persist { \
    0.0.0.0/8, 10.0.0.0/8, 100.64.0.0/10, 127.0.0.0/8, \
    169.254.0.0/16, 172.16.0.0/12, 192.0.2.0/24, 192.168.0.0/16, \
    198.18.0.0/15, 198.51.100.0/24, 203.0.113.0/24, 224.0.0.0/4, 240.0.0.0/4 }

table <bogons6> persist { \
    ::/128, ::1/128, ::ffff:0:0/96, 64:ff9b::/96, \
    100::/64, 2001:db8::/32, fc00::/7, fe80::/10, ff00::/8 }

# 内网（按你的部署调整）
table <inside4> persist { 10.0.0.0/8, 192.168.0.0/16 }
table <inside6> persist { 2001:db8:10::/48 }

# -------------------------
# 规范化
# -------------------------
scrub in all

# -------------------------
# 基础策略
# -------------------------
block all

# 在所有相关接口上反伪造
antispoof quick for { $wan, $inside }

# 反向路径过滤：丢弃源地址无法经接收接口路由回去的流量
block in quick on $wan from urpf-failed
block in quick on $inside from urpf-failed

# WAN 上的 bogon 过滤
block in quick on $wan inet from <bogons4> to any
block in quick on $wan inet6 from <bogons6> to any

# 允许内网到任意地址（按策略细化；示例如下）
pass in on $inside inet  from <inside4> to any keep state
pass in on $inside inet6 from <inside6> to any keep state

# 防火墙自身的出站
pass out on egress proto { tcp udp icmp } keep state

# 示例：发布的 HTTPS 服务；应用 synproxy 与限制
web_svc = "10.10.10.50"
rdr on $wan proto tcp from any to ($wan) port 443 -> $web_svc
pass in on $wan proto tcp to $web_svc port 443 \
    flags S/SA synproxy state (max-src-conn 50, max-src-conn-rate 30/5)
```
> 在运维手册中保持 bogon 列表准确。表更新成本低，事件期间易于审计。

### 2) 故障安全锚点与安全 PF 部署

维护一个最小、最先加载的锚点，保证从受限来源的管理访问。在任何限制性规则之前加载它，并保持其精简。

```pf
## /etc/pf.failsafe.conf — 仅允许来自管理 /32 的 SSH
mgmt = "198.51.100.10"

pass in quick on egress proto tcp from $mgmt to (egress) port 22 keep state
```

在主策略顶部引用它：

```pf
## 摘录 — /etc/pf.conf 顶部
anchor "failsafe"
load anchor "failsafe" from "/etc/pf.failsafe.conf"
```

**带验证与回滚定时器的分阶段重载**（路由器上的本地 shell）：

```sh
# install -b /etc/pf.conf
  # 自动备份为 /etc/pf.conf~
# pfctl -nf /etc/pf.conf
  # 语法检查；不加载
# at now + 5 minutes <<< 'pfctl -f /etc/pf.conf~'
  # 5 分钟后将 PF 回滚到上一规则集（需要 atd(8)）
# pfctl -f /etc/pf.conf
  # 原子加载新规则
# atq
  # 确认回滚任务已排队
# atrm <jobid>
  # 确认访问与功能后取消回滚
```

若 `at(1)` 不可用，加载新策略时保持第二个会话开启（串口/IPMI/ILO），在关闭带外路径前进行端到端验证。

### 3) 管理平面访问：SSH 与 doas

限制 SSH 暴露并禁用弱选项。特权提升须显式且最小化。

**sshd(8) 加固**，在 [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/) 中：

```
## /etc/ssh/sshd_config — 最小加固示例
ListenAddress 10.10.10.1
AddressFamily inet
Protocol 2
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
ClientAliveInterval 30
ClientAliveCountMax 2
AllowUsers netops
# 可选：用 Match 限制来源子网
# Match Address 198.51.100.0/24
#     AllowUsers netops
```

```sh
# rcctl reload sshd
  # 应用更改且不断开既有会话
```

**doas(1)** 策略，在 [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/) 中：

```sh
## /etc/doas.conf — 日常运维的最小权限
permit persist keepenv :wheel as root cmd pfctl
permit persist keepenv :wheel as root cmd rcctl
permit persist keepenv :wheel as root cmd tail args -f /var/log/messages
# 默认拒绝；按需枚举更多命令
```

### 4) 密钥文件卫生与守护进程姿态

确保私钥与 PSK 仅 root 可读。许多基本系统守护进程已自带 chroot 与权限分离；保持默认不变。

```sh
# chmod 600 /etc/iked.conf
  # iked(8) 的 PSK 与身份
# chmod 600 /etc/ssl/private/*.key
  # httpd(8) 等守护进程使用的 TLS 私钥
# chown -R root:wheel /etc/ssl/private
# rcctl check iked
  # 在当前 chroot/权限分离模型下确认守护进程状态
```

### 5) 补丁与溯源

用 [syspatch(8)](https://man.openbsdhandbook.com/syspatch.8/) 应用勘误。升级时用 [signify(1)](https://man.openbsdhandbook.com/signify.1/) 校验套件签名。规划维护窗口，并在补丁说明要求时重启。

```sh
# syspatch
  # 应用可用的二进制补丁；查看 /var/syspatch 日志
# signify -C -p /etc/signify/openbsd-78-base.pub -x SHA256.sig
  # 升级时校验套件签名（按发行说明调整密钥/文件名）
```

> 将 `/etc` 纳入版本控制（例如 `rc.conf.local`、`pf.conf`、`vm.conf`、`rad.conf`），并在每次提交中附上可读的变更理由。

## 验证

- **PF 暴露与计数器**：

```sh
$ pfctl -vvsr | egrep 'failsafe|urpf|antispoof|bogons'
  # 确认加固规则的存在与顺序
$ pfctl -s state | wc -l
  # 状态数；在变更窗口期间关注
$ pfctl -s Interfaces
  # PF 所见的接口列表与状态
```

- **表与查询**：

```sh
$ pfctl -t bogons4 -T show | head
  # 验证 bogon 表内容
$ pfctl -t inside4 -T test 192.168.1.10
  # 内网地址应返回 "in table"
```

- **管理平面检查**：

```sh
$ ssh -o PreferredAuthentications=publickey netops@10.10.10.1 true
  # 非交互式密钥认证成功
$ ssh -o PreferredAuthentications=password netops@10.10.10.1 true
  # 若 PasswordAuthentication no，应失败
```

- **补丁状态**：

```sh
$ syspatch -l
  # 列出已安装的补丁
```

## 故障排查

- **PF 重载后被锁出。** 使用带外控制台。确认故障安全锚点在限制性规则之前加载，且管理来源匹配 `AllowUsers` 与 PF 条件。须快速恢复时，可从控制台用 `pfctl -d` 停用过滤，修复策略后用 `pfctl -e` 重新启用。
- **合法流量在 WAN 上被丢弃。** 用 `tcpdump -ni $wan` 检查 `urpf-failed` 命中，同时关联 PF 计数器。返回路径失败通常表明非对称路由；为这些流调整策略，或从受影响接口移除严格 uRPF 规则。
- **Bogon 表阻断了预期对端。** 文档/演示前缀（例如 `192.0.2.0/24`）偶尔出现在实验室对等中。临时移除这些条目，或为生产环境维护单独的 `<bogons4_prod>` 表。
- **长操作期间 SSH 断开。** 使用 `ClientAliveInterval` 与 keep-alive；考虑在带外控制台上通过 `tmux` 或 `screen` 会话执行破坏性变更。
- **守护进程拒绝读取密钥。** 确认文件权限 `600` 且路径正确。许多守护进程要求密钥位于 `/etc/ssl/private` 且仅 root 可读。查看 `/var/log/daemon` 中的守护进程日志与服务特定文件。
- **补丁要求重启。** 部分 `syspatch(8)` 更新会修补内核或核心库。安排短暂的重启窗口，并在 `rcctl` 下验证服务恢复。

## 参见

- [网络](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- 相关：[**高可用与状态复制**](high-availability.md)
- 相关：[**遥测、日志与流导出**](telemetry-logging-flow.md)
- 相关：[**故障排查手册**](troubleshooting-playbooks.md)
