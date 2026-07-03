# 搭建简易路由器与防火墙

## 概览

本章介绍如何将 OpenBSD 配置为一台小型路由器与防火墙，使用两块网卡：一块 **WAN** 接口连接运营商，一块 **LAN** 接口连接本地网络。配置内容包括启用 IPv4 转发、网络地址转换（NAT）、有状态防火墙、通过 DHCP 分配 IPv4 地址，以及本地 DNS 缓存。

**路由器** 在不同网络之间转发 IP 包。**防火墙** 按策略控制哪些包允许通过。**NAT** 在出口处把地址转换为公网地址，使多台本地主机共用一个公网地址。OpenBSD 使用 **Packet Filter**（**pf(4)**）完成过滤、NAT 与流量规范化。参见 [pf(4)](https://man.openbsdhandbook.com/pf.4/)
。

除特别说明外，本章命令均需 root 权限。

本章示例使用两块接口：`re0` 作为 WAN，`re1` 作为 LAN。请按系统中实际存在的接口名替换，可用 [% ifconfig -A](https://man.openbsdhandbook.com/ifconfig.8/)
查看。

## 网络拓扑与前提假设

示例假设如下：

- WAN 接口：`re0`，由运营商通过 DHCP 配置；若运营商要求静态地址，则配置为静态地址。
- LAN 接口：`re1`，配置 IPv4 子网 `10.0.0.0/24`。路由器在 LAN 侧使用 `10.0.0.1`。

请按目标环境调整地址与接口名。

## 启用 IP 转发

用 [sysctl(8)](https://man.openbsdhandbook.com/sysctl.8/)
在运行时启用 IPv4 与可选的 IPv6 包转发，并将设置持久化到 [sysctl.conf(5)](https://man.openbsdhandbook.com/sysctl.conf.5/)
。

```sh
# sysctl net.inet.ip.forwarding=1
# sysctl net.inet6.ip6.forwarding=1
# printf 'net.inet.ip.forwarding=1\n' >> /etc/sysctl.conf
# printf 'net.inet6.ip6.forwarding=1\n' >> /etc/sysctl.conf
```

LAN 侧的 IPv6 自动配置需要路由通告。如有需要，配置 [rad(8)](https://man.openbsdhandbook.com/rad.8/)
或 dhcp6d(8)。本章聚焦 IPv4 服务。

## 配置网络接口

配置 WAN 接口。若运营商通过 DHCP 分配地址，按 [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
写入接口配置文件：

```sh
# printf 'inet autoconf\n' > /etc/hostname.re0
```

若运营商分配静态地址，按需设置地址、子网掩码与广播地址：

```sh
# cat > /etc/hostname.re0 <<'EOF'
inet 203.0.113.10 255.255.255.0 203.0.113.255
EOF
```

为 LAN 接口配置静态地址：

```sh
# cat > /etc/hostname.re1 <<'EOF'
inet 10.0.0.1 255.255.255.0 10.0.0.255
EOF
```

用 [netstart(8)](https://man.openbsdhandbook.com/netstart.8/)
应用接口配置：

```sh
# sh /etc/netstart
```

用 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
核对地址：

```sh
$ ifconfig re0
$ ifconfig re1
```

## 配置 DHCP 服务

DHCP 服务器 [dhcpd(8)](https://man.openbsdhandbook.com/dhcpd.8/)
为 LAN 客户端分配 IPv4 地址与选项。按 [dhcpd.conf(5)](https://man.openbsdhandbook.com/dhcpd.conf.5/)
为 `10.0.0.0/24` 网络创建 `/etc/dhcpd.conf`。本例将默认路由与 DNS 服务器都设为路由器自身，并划分一个动态地址池。

```
subnet 10.0.0.0 netmask 255.255.255.0 {
  option routers 10.0.0.1;
  option domain-name-servers 10.0.0.1;
  range 10.0.0.51 10.0.0.250;
}
```

可选：按硬件（MAC）地址分配固定地址：

```
host server1 {
  fixed-address 10.0.0.30;
  hardware ethernet 00:00:00:00:00:00;
}
```

将 `dhcpd` 设为开机自启，限定在 LAN 接口上运行，并用 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
启动：

```sh
# rcctl enable dhcpd
# rcctl set dhcpd flags re1
# rcctl start dhcpd
# rcctl check dhcpd
```

关键指令说明：

- `subnet` 与 `netmask` 定义所服务的 IPv4 网络。
- `option routers` 设定客户端的默认网关。
- `option domain-name-servers` 向客户端通告 DNS 解析器。
- `range` 定义可租借的动态地址池。
- `fixed-address` 与 `hardware ethernet` 将静态租约绑定到客户端 MAC 地址。

## 用 PF 配置防火墙与 NAT

用 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
为 **pf(4)** 定义防火墙规则与 NAT 策略。创建 `/etc/pf.conf`，写入以下基线策略。请按环境调整接口名与网络。

```pf
# /etc/pf.conf - 面向 LAN 的最小化路由器/防火墙（含 NAT）

# 接口与网络
lan_if = "re1"
lan_net = "10.0.0.0/24"

# 不可路由与保留地址空间
table <martians> { \
  0.0.0.0/8, 10.0.0.0/8, 127.0.0.0/8, 169.254.0.0/16, \
  172.16.0.0/12, 192.0.0.0/24, 192.0.2.0/24, 198.18.0.0/15, \
  198.51.100.0/24, 203.0.113.0/24, 224.0.0.0/3, 192.168.0.0/16 \
}

# 默认行为
set block-policy drop
set loginterface egress
set skip on lo

# 规范化流量
match in all scrub (no-df random-id max-mss 1440)

# 为 LAN 到当前出口地址做 NAT
match out on egress inet from $lan_net to any nat-to (egress:0)

# 防欺骗
antispoof quick for { egress, $lan_if }
block in  quick on egress from <martians> to any
block return out quick on egress from any to <martians>

# 默认拒绝
block all

# 允许已建立与相关联的流量
pass out on egress inet keep state

# 允许 LAN 访问任意地址
pass in on $lan_if inet from $lan_net to any keep state
```

校验并载入规则集，确保 PF 已启用。用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
检查与载入规则，用 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
将 PF 设为开机自启：

```sh
# pfctl -nf /etc/pf.conf
# pfctl -f /etc/pf.conf
# rcctl enable pf
# pfctl -e
# pfctl -sr
# pfctl -sn
```

`egress` 关键字匹配承载默认路由的接口，详见 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
。该策略丢弃来自互联网的未经请求入站流量，允许 LAN 主动发起连接，并对出站 IPv4 流量执行 NAT。

## 用 Unbound 配置本地 DNS 缓存

[unbound(8)](https://man.openbsdhandbook.com/unbound.8/)
是一款可验证的递归 DNS 解析器。将其配置为监听路由器自身与 LAN 地址，并把查询转发到选定的上游解析器。按 [unbound.conf(5)](https://man.openbsdhandbook.com/unbound.conf.5/)
创建 `/var/unbound/etc/unbound.conf`：

```
server:
  interface: 10.0.0.1
  interface: 127.0.0.1
  interface: ::1

  access-control: 10.0.0.0/24 allow
  access-control: 127.0.0.0/8 allow
  access-control: ::1 allow
  access-control: 0.0.0.0/0 refuse
  access-control: ::0/0 refuse

  hide-identity: yes
  hide-version: yes

forward-zone:
  name: "."
  forward-addr: 64.6.64.6        # Verisign
  forward-addr: 94.75.228.29     # CCC
  forward-first: yes
```

用 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
启用并启动服务：

```sh
# rcctl enable unbound
# rcctl start unbound
# rcctl check unbound
```

LAN 上的客户端会通过 DHCP 收到 `10.0.0.1` 作为 DNS 服务器。如有需要，路由器自身也可将 [/etc/resolv.conf](https://man.openbsdhandbook.com/resolv.conf.5/)
指向 `127.0.0.1`，使用本地解析器。

## 验证

配置完成后，将一台客户端接入 LAN，确认其能从配置的地址池获得地址、默认路由 `10.0.0.1` 以及 DNS 服务器 `10.0.0.1`。

从客户端验证 IP 连通性与名称解析。用 [ping(8)](https://man.openbsdhandbook.com/ping.8/)
对公网 IP 做基本可达性测试，用 [ftp(1)](https://man.openbsdhandbook.com/ftp.1/)
按主机名获取资源以检验 DNS：

```sh
$ ping -n 8.8.8.8
$ ftp -o - https://example.com/ >/dev/null
```

在路由器上，用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
确认 NAT 与状态表存在：

```sh
# pfctl -ss
# pfctl -s state
```

本手册提供简明的 `pfctl` 速查表，见 [/pf/cheat\_sheet/](../../networking-and-daemons/pf/cheat_sheet.md)
。详细参考请查阅本手册托管的手册页：[pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
、[pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
、[dhcpd.conf(5)](https://man.openbsdhandbook.com/dhcpd.conf.5/)
、[dhcpd(8)](https://man.openbsdhandbook.com/dhcpd.8/)
、[unbound.conf(5)](https://man.openbsdhandbook.com/unbound.conf.5/)
、[unbound(8)](https://man.openbsdhandbook.com/unbound.8/)
、[hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
、[sysctl(8)](https://man.openbsdhandbook.com/sysctl.8/)
与 [sysctl.conf(5)](https://man.openbsdhandbook.com/sysctl.conf.5/)
。
