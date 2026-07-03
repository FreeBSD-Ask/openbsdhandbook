# VPN 与加密隧道

## 摘要

本章介绍 OpenBSD 上的站点到站点虚拟专用网络（VPN）与主机到主机加密隧道。重点介绍使用基本系统 [iked(8)](https://man.openbsdhandbook.com/iked.8/)
与内核 IPsec 协议栈 [ipsec(4)](https://man.openbsdhandbook.com/ipsec.4/)
的 IKEv2，并提供 NAT 穿透、选择器设计与可观测性的实用模式。同时给出使用内核接口 [wg(4)](https://man.openbsdhandbook.com/wg.4/)
的 WireGuard 风格隧道最小示例。运行时管理使用 [ikectl(8)](https://man.openbsdhandbook.com/ikectl.8/)
，包过滤由 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
与 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
处理，链路检查使用 [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
。IPsec 的加密接口为 [enc(4)](https://man.openbsdhandbook.com/enc.4/)
。

这些模式可用于在不可信网络上互联站点、跨运营商安全发布内部服务，或用现代加密替换遗留隧道。

## 设计考量

- **选择器与范围。** 按网络定义流量（例如 `10.0.0.0/24` ↔ `10.20.0.0/24`），不要用 `any`。选择器尽量精简，且在两端对称。
- **认证。** 站点到站点场景下预共享密钥（PSK）最简单。证书更易于扩展，在需要第三方信任或吊销时优先使用。
- **NAT 穿透。** 在 WAN 上放行 UDP 端口 500 与 4500 以及 ESP 协议。IKEv2 NAT-T 在检测到地址转换时自动启用。
- **路由。** 对于需要路由的站点，启用 IPv4/IPv6 转发，并添加静态路由（或在隧道内运行动态路由）。
- **性能。** 优先选用 AES-GCM 提案，将完整性校验交由加密算法承担。封装跨经受限链路时须留意 MTU 与 MSS。
- **运维。** 确保所有对端时间同步（例如使用 [ntpd(8)](https://man.openbsdhandbook.com/ntpd.8/)
  ）。发布期间记录 IKE 与 PF 事件。

## 配置

示例假设：

- **站点 A** WAN：`198.51.100.10`，LAN：`10.0.0.0/24`
- **站点 B** WAN：`203.0.113.10`，LAN：`10.20.0.0/24`

请将接口名与地址调整为与你的环境一致。

### 基础系统设置（两个站点）

```sh
# printf '%s\n' \
    'net.inet.ip.forwarding=1' \
    'net.inet6.ip6.forwarding=1' \
    >> /etc/sysctl.conf
  # 为穿越防火墙/路由器的被路由流量启用 L3 转发

# sysctl net.inet.ip.forwarding=1 net.inet6.ip6.forwarding=1
  # 立即应用
```

### PF 放行 IKEv2 与 ESP（两个站点）

在 WAN 上放行 IKE（UDP 500/4500）与 ESP，并在内侧放行受保护子网间的流量。语法定义见 [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
。

```pf
## /etc/pf.conf — minimal allowances for IKEv2/IPsec

set skip on lo

wan     = "em0"           # adjust
lan_net = "{ 10.0.0.0/24, 10.20.0.0/24 }"

block all

# IKE and IPsec on the WAN
pass in on $wan proto udp to ($wan) port { 500, 4500 } keep state
pass in on $wan proto esp keep state
pass out on $wan proto { udp, esp } keep state

# Permit protected subnets after decryption
pass on enc0 from 10.0.0.0/24 to 10.20.0.0/24 keep state
pass on enc0 from 10.20.0.0/24 to 10.0.0.0/24 keep state
```

用 [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
重载并确认。

```sh
# pfctl -f /etc/pf.conf
# pfctl -sr | egrep 'udp .* (500|4500)|proto esp|enc0'
```

### 使用 PSK 的站点到站点 IKEv2

配置存放于 [iked.conf(5)](https://man.openbsdhandbook.com/iked.conf.5/)
。一端发起（`active`），另一端监听（`passive`）。下方提案使用 AES-GCM。

#### 站点 A（发起方）

```sh
## /etc/iked.conf — Site A

ikev2 "s2s-a2b" active esp from 10.0.0.0/24 to 10.20.0.0/24 \
    peer 203.0.113.10 \
    ikesa enc aes-256-gcm prf sha256 group 14 \
    childsa enc aes-256-gcm \
    psk "change-this-shared-secret"
```

#### 站点 B（响应方）

```sh
## /etc/iked.conf — Site B

ikev2 "s2s-b2a" passive esp from 10.20.0.0/24 to 10.0.0.0/24 \
    ikesa enc aes-256-gcm prf sha256 group 14 \
    childsa enc aes-256-gcm \
    psk "change-this-shared-secret"
```

在两端用 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
启用并启动 [iked(8)](https://man.openbsdhandbook.com/iked.8/)
：

```sh
# rcctl enable iked
  # 开机启动
# rcctl start iked
  # 立即启动（发起方将主动拨向响应方）
```

若任一端位于 NAT 后，IKEv2 会自动把 ESP 封装到 UDP 4500（NAT-T）。请确保上游放行这些端口。

### 使用 wg(4) 的最小 WireGuard 风格隧道

[wg(4)](https://man.openbsdhandbook.com/wg.4/)
接口提供紧凑的静态密钥隧道。密钥使用 Curve25519，按 wg(4) 文档生成，并将私钥放在各主机上（例如 `/etc/wg/private.key`）。接口属性由 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
设置。

示例：在主机间建立点对点 `/30`，内侧地址为 `10.99.0.1/30`（站点 A）与 `10.99.0.2/30`（站点 B），UDP 端口 `51820`。

#### 站点 A

```sh
# ifconfig wg0 create
  # 创建接口
# ifconfig wg0 inet 10.99.0.1/30
  # 分配内侧地址
# ifconfig wg0 wgport 51820
  # 在 UDP/51820 上监听
# ifconfig wg0 wgkey `cat /etc/wg/private.key`
  # 加载本机私钥
# ifconfig wg0 wgpeer <SITE_B_PUBLIC_KEY> wgendpoint 203.0.113.10 51820
  # 配置对端公钥与端点
# ifconfig wg0 wgpeer <SITE_B_PUBLIC_KEY> wgaip 10.99.0.2/32
  # 对端的 Allowed-IPs（对端内侧地址）
# route add -net 10.20.0.0/24 10.99.0.2
  # 如需，通过隧道路由远端 LAN
```

#### 站点 B

```sh
# ifconfig wg0 create
# ifconfig wg0 inet 10.99.0.2/30
# ifconfig wg0 wgport 51820
# ifconfig wg0 wgkey `cat /etc/wg/private.key`
# ifconfig wg0 wgpeer <SITE_A_PUBLIC_KEY> wgendpoint 198.51.100.10 51820
# ifconfig wg0 wgpeer <SITE_A_PUBLIC_KEY> wgaip 10.99.0.1/32
# route add -net 10.0.0.0/24 10.99.0.1
```

在 PF 中放行各 WAN 上的 WireGuard UDP 端口，并按需在 `wg0` 上放行内侧前缀。

## 验证

- **IKEv2 状态与流**，使用 [ikectl(8)](https://man.openbsdhandbook.com/ikectl.8/)
  ：

```sh
$ ikectl show sa
  # IKE_SA 与 CHILD_SA 状态、SPI、生命周期
$ ikectl show flows
  # 当前已安装的流量选择器
```

- **链路上的封装流量**，使用 [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
  ：

```sh
# tcpdump -ni em0 port 500 or port 4500 or proto esp
  # WAN 上的 IKE_SA 协商与 ESP/NAT-T
# tcpdump -ni enc0 host 10.20.0.50
  # IPsec 在 enc0 上的明文（解密后）流量
# tcpdump -ni em0 udp port 51820
  # WireGuard UDP 传输
```

- **路径测试：**

```sh
$ ping -n -c 3 10.20.0.1
  # 从站点 A 后方主机到站点 B 后方主机（IKEv2）
$ ping -n -c 3 10.99.0.2
  # 隧道端点之间（WireGuard）
```

- **PF 与路由：**

```sh
$ pfctl -vvsr | egrep 'udp .* (500|4500)|proto esp|udp .* 51820'
  # 确认放行规则
$ netstat -rn | egrep '10\.99\.0\.|10\.20\.0\.'
  # 确认内侧/隧道路由
```

## 故障排查

- **IKE 未协商。** 验证 UDP 500/4500 可达性，以及响应方处于 `passive` 模式。查看 `/var/log/daemon` 中的 `iked` 日志。
- **选择器不匹配。** 若一端使用 `10.0.0.0/24 ↔ 10.20.0.0/24`，另一端使用 `10.0.0.0/24 ↔ 10.0.0.0/24`，CHILD\_SA 将无法安装。对齐 `from`/`to` 网络。检查 `ikectl show flows`。
- **NAT 或路径非对称。** 确保上游透传 UDP 4500 不变，且回流流量走相同出口。对于 IPsec，`enc0` 规则应放行受保护子网。
- **MTU 黑洞。** 若大传输卡住，在 PF 中测试期间钳制 MSS（`scrub in max-mss ...`），测量路径 MTU 后再精调。
- **时钟漂移。** 证书与 IKE 生命周期对时间敏感。确保 NTP 正常工作，使用 [ntpd(8)](https://man.openbsdhandbook.com/ntpd.8/)
  。
- **WireGuard 握手失败。** 确认公钥正确、`wgport` 匹配、`wgendpoint` 可达，且双方都将对方的内侧 `/32` 作为 Allowed-IP。用 `ifconfig wg0` 查看当前对端与统计。

## 另请参见

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- 相关：[**经典轻量隧道**](classic-tunnels.md)
- 相关：[**加固与运维安全**](hardening-operations.md)
