# Classic and Lightweight Tunnels

## Synopsis

This chapter covers lightweight, in-base tunneling mechanisms for pragmatic overlays and migrations: [gif(4)](https://man.openbsdhandbook.com/gif.4/)
(generic IP-in-IP), [gre(4)](https://man.openbsdhandbook.com/gre.4/)
(Generic Routing Encapsulation), and [etherip(4)](https://man.openbsdhandbook.com/etherip.4/)
(Layer-2 Ethernet over IP). These interfaces interoperate well with non-OpenBSD devices and are suitable when cryptography is not required, when you must carry non-IP protocols (GRE), or when you must extend L2 domains (EtherIP). Boot-time configuration uses [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
; runtime management uses [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
. Filtering and allowances live in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
and are applied with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
.

Use these mechanisms for temporary migrations, quick overlays between sites, lab environments, or when you need simple encapsulation without the control-plane complexity of a full VPN.

## Design Considerations

- **Encapsulation choice.**
  - **gif:** IP-in-IP only (no L2); smallest header; good for routing IP subnets between known endpoints.
  - **gre:** Flexible payload (commonly IP); widely supported on routers and appliances.
  - **etherip:** Bridges Ethernet frames across an IP path; use sparingly and avoid L2 loops.
- **Security.** These mechanisms provide **no encryption or integrity**. Use them only on trusted or controlled underlays, or combine with [pf(4)](https://man.openbsdhandbook.com/pf.4/)
  filtering. For cryptographic tunnels, prefer IKEv2 or [wg(4)](https://man.openbsdhandbook.com/wg.4/)
  as covered in the previous chapter.
- **Forwarding and routing.** Enable IPv4/IPv6 forwarding when routing subnets through the tunnel. Add explicit routes for remote prefixes.
- **MTU and fragmentation.** Encapsulation reduces effective MTU. Set a conservative MTU initially and refine after measuring path MTU.
- **Symmetry and addressing.** Tunnels are point-to-point. Configure the correct local/remote order at both ends.
- **Filtering.** Permit the relevant IP protocols on the underlay: IP protocol 4 (gif), 47 (GRE), and 97 (EtherIP).
- **Operational scope.** Prefer routed overlays (gif/gre) over L2 extension. Use [bridge(4)](https://man.openbsdhandbook.com/bridge.4/)
  with EtherIP only when you must extend broadcast domains.

## Configuration

Assume two sites:

- **Site A** WAN: `198.51.100.10`
- **Site B** WAN: `203.0.113.10`

LANs:

- **A LAN:** `10.0.0.0/24`
- **B LAN:** `10.20.0.0/24`

### Common system settings (both sites)

Enable L3 forwarding so routed prefixes traverse the tunnel. Persist with [sysctl.conf(5)](https://man.openbsdhandbook.com/sysctl.conf.5/)
.

```sh
# printf '%s\n' \
    'net.inet.ip.forwarding=1' \
    'net.inet6.ip6.forwarding=1' >> /etc/sysctl.conf
  # Persist IPv4/IPv6 forwarding

# sysctl net.inet.ip.forwarding=1 net.inet6.ip6.forwarding=1
  # Apply immediately
```

Allow encapsulation protocols on the WAN in PF. Adjust the interface and policy as required.

```pf
## /etc/pf.conf — underlay allowances for lightweight tunnels

set skip on lo
wan = "em0"   # adjust to your WAN interface

block all
pass out on $wan keep state

# Encapsulation protocols (permit from known peer for tighter policy)
pass in on $wan proto { 4, 47, 97 } from 203.0.113.10 to ($wan) keep state
pass out on $wan proto { 4, 47, 97 } to 203.0.113.10 keep state
```

Reload with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
.

```sh
# pfctl -f /etc/pf.conf
```

### A. gif(4): IP-in-IP routed overlay

Create `gif0` at both sites, assign a point-to-point address pair, and route the remote LAN via the tunnel.

#### Site A

```
# /etc/hostname.gif0
tunnel 198.51.100.10 203.0.113.10
inet 10.254.0.1 255.255.255.252 10.254.0.2
mtu 1400
up
```

```sh
# sh /etc/netstart gif0
  # Activate the tunnel
# route add -net 10.20.0.0/24 10.254.0.2
  # Route B LAN via the tunnel
```

#### Site B

```
# /etc/hostname.gif0
tunnel 203.0.113.10 198.51.100.10
inet 10.254.0.2 255.255.255.252 10.254.0.1
mtu 1400
up
```

```sh
# sh /etc/netstart gif0
# route add -net 10.0.0.0/24 10.254.0.1
```

### B. gre(4): Generic Routing Encapsulation

GRE is commonly required when the peer expects GRE specifically (for example, a router or cloud gateway). Configuration is analogous to gif.

#### Site A

```
# /etc/hostname.gre0
tunnel 198.51.100.10 203.0.113.10
inet 10.254.1.1 255.255.255.252 10.254.1.2
mtu 1400
up
```

```sh
# sh /etc/netstart gre0
# route add -net 10.20.0.0/24 10.254.1.2
```

#### Site B

```
# /etc/hostname.gre0
tunnel 203.0.113.10 198.51.100.10
inet 10.254.1.2 255.255.255.252 10.254.1.1
mtu 1400
up
```

```sh
# sh /etc/netstart gre0
# route add -net 10.0.0.0/24 10.254.1.1
```

> **Note:** If the far-end requires GRE keys or additional attributes, configure them per [gre(4)](https://man.openbsdhandbook.com/gre.4/)
> using `ifconfig gre0 ...` options supported by that implementation.

### C. etherip(4): Layer-2 extension over IP

EtherIP transports Ethernet frames between sites. Use it only when routed overlays are not viable (for example, when you must carry non-IP broadcasts or a VLAN intact). Avoid loops and keep broadcast domains constrained.

#### Site A

```
# /etc/hostname.etherip0
tunnel 198.51.100.10 203.0.113.10
up
```

Create a bridge to join the local LAN interface (`em1`) to the tunnel. Bridge parameters are managed by [bridge(4)](https://man.openbsdhandbook.com/bridge.4/)
.

```
# /etc/hostname.bridge0
add em1
add etherip0
up
```

Activate:

```sh
# sh /etc/netstart etherip0 bridge0
```

#### Site B

Mirror the configuration with reversed tunnel endpoints.

```
# /etc/hostname.etherip0
tunnel 203.0.113.10 198.51.100.10
up
```
```
# /etc/hostname.bridge0
add em1
add etherip0
up
```

```sh
# sh /etc/netstart etherip0 bridge0
```

> **Caution:** L2 extension spans broadcast and failure domains. Ensure there is exactly one active path between the bridged segments. If you must introduce redundancy, design and test loop prevention per [bridge(4)](https://man.openbsdhandbook.com/bridge.4/)
> before production.

## Verification

- **Interface status** with [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
  :

```sh
$ ifconfig gif0
  # Expect "UP" and the correct tunnel src/dst and P2P addresses
$ ifconfig gre0
$ ifconfig etherip0
$ ifconfig bridge0
  # For EtherIP, ensure both members (em1, etherip0) are in the bridge
```

- **Routing and reachability** with [route(8)](https://man.openbsdhandbook.com/route.8/)
  and `ping(8)`:

```sh
$ netstat -rn | egrep '10\.254\.|10\.0\.0\.|10\.20\.0\.'
  # Confirm tunnel /30 and remote LAN routes
$ ping -n -c 3 10.20.0.1
  # From Site A LAN host to Site B LAN host (gif/gre)
```

- **On-the-wire confirmation** with [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
  :

```sh
# tcpdump -ni em0 proto 4
  # gif (IP-in-IP) packets on the WAN
# tcpdump -ni em0 proto 47
  # GRE packets on the WAN
# tcpdump -ni em0 proto 97
  # EtherIP frames on the WAN
```

- **PF rules and counters** with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
  :

```sh
$ pfctl -vvsr | egrep 'proto (4|47|97)'
  # Verify allowances are loaded and counters increment under test
```

## Troubleshooting

- **No tunnel traffic on the WAN.** Check that PF permits the correct protocol (4, 47, or 97) to and from the peer address. Use `tcpdump` on the WAN to confirm encapsulated packets.
- **Encapsulation up, but no routing.** Ensure `net.inet.ip.forwarding=1` and that static routes to the remote LAN point at the tunnel neighbor. Confirm with `netstat -rn`.
- **Asymmetric or intermittent reachability.** Verify both ends use matching local/remote `tunnel` endpoints and the correct /30 addressing. Asymmetry commonly results from reversed arguments.
- **Fragmentation or stalls.** Lower the tunnel MTU (for example, to 1400) and retest. Measure PMTU after stability and increase cautiously.
- **Broadcast storms over EtherIP.** EtherIP extends the broadcast domain. Confirm there is only one L2 path and avoid bridging additional WAN links without loop prevention.
- **Interoperability with non-OpenBSD peers.** For GRE, check peer-specific requirements (keys, keepalives, or DF handling) and adjust `ifconfig gre0 ...` options accordingly.

## See Also

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- Related: [**VPN and Cryptographic Tunneling**](vpn-and-crypto-tunneling.md)
- Related: [**IPv6 at Scale**](ipv6-at-scale.md)
