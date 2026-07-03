# Reference Architectures

## Synopsis

This chapter assembles end-to-end reference designs built from components covered earlier. It presents three canonical patterns:

1. **Redundant Internet Edge** with [carp(4)](https://man.openbsdhandbook.com/carp.4/)
   , [pfsync(4)](https://man.openbsdhandbook.com/pfsync.4/)
   , per-uplink NAT and policy routing in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
   , managed by [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
   .
2. **Campus/Branch Routed Access** using VLAN SVIs, Router Advertisements with [rad(8)](https://man.openbsdhandbook.com/rad.8/)
   , internal recursion via [unbound(8)](https://man.openbsdhandbook.com/unbound.8/)
   , authoritative zones with [nsd(8)](https://man.openbsdhandbook.com/nsd.8/)
   , and DHCP with [dhcpd(8)](https://man.openbsdhandbook.com/dhcpd.8/)
   .
3. **Regional POP** with either an MPLS LDP core via [ldpd(8)](https://man.openbsdhandbook.com/ldpd.8/)
   and [mpe(4)](https://man.openbsdhandbook.com/mpe.4/)
   , or an encrypted hub-and-spoke using [iked(8)](https://man.openbsdhandbook.com/iked.8/)
   or [wg(4)](https://man.openbsdhandbook.com/wg.4/)
   . BGP policy lives in the OpenBGPD chapter.

Each pattern includes minimal, copy-pasteable configuration and verification guidance.

## Design Considerations

- **Failure domains.** Keep state sync and control traffic on a private link. Avoid bridging across routers unless strictly required.
- **Configuration parity.** Maintain identical PF and service configs on HA pairs; only device-specific interface addressing should differ.
- **Change management.** Use atomic PF reloads and a rollback timer during first deployment. Keep a failsafe anchor that preserves management access.
- **Addressing plan.** Allocate deterministic per-VLAN /24 (IPv4) and /64 (IPv6) with structured mapping to sites and roles.
- **Observability.** Enable [pflogd(8)](https://man.openbsdhandbook.com/pflogd.8/)
  and (optionally) [pflow(4)](https://man.openbsdhandbook.com/pflow.4/)
  toward a collector; keep clocks accurate via [ntpd(8)](https://man.openbsdhandbook.com/ntpd.8/)
  .

---

## Pattern 1 — Redundant Internet Edge (Two Firewalls, Two ISPs)

**Scope.** Two OpenBSD firewalls in HA; two WAN uplinks (ISP-A and ISP-B); deterministic outbound steering and symmetric inbound using `route-to` and `reply-to`.

### Topology and Interface Assumptions

- `em0` = ISP-A, `em3` = ISP-B, `em1` = LAN, `em2` = SYNC
- CARP VIPs: WAN-A `198.51.100.2/29` (vhid 10), WAN-B `203.0.113.2/29` (vhid 20), LAN `10.10.10.1/24` (vhid 30)
- Node A preferred (advskew 0), Node B backup (advskew 100)

### System and Interfaces (Node A; mirror with advskew 100 on Node B)

```sh
# printf '%s\n' 'net.inet.carp.preempt=1' >> /etc/sysctl.conf
# sysctl net.inet.carp.preempt=1
  # Prefer the designated MASTER after repair

# cat > /etc/hostname.em2 <<'EOF'
inet 10.255.255.2 255.255.255.252
EOF
  # State-sync /30

# cat > /etc/hostname.pfsync0 <<'EOF'
up syncdev em2
EOF
  # pfsync over SYNC

# cat > /etc/hostname.em0 <<'EOF'
inet 198.51.100.3 255.255.255.248
EOF
# cat > /etc/hostname.em3 <<'EOF'
inet 203.0.113.3 255.255.255.248
EOF
# cat > /etc/hostname.em1 <<'EOF'
inet 10.10.10.2 255.255.255.0
EOF

# cat > /etc/hostname.carp0 <<'EOF'
inet 198.51.100.2 255.255.255.248 vhid 10 carpdev em0 pass sharedsecret advskew 0
EOF
# cat > /etc/hostname.carp1 <<'EOF'
inet 203.0.113.2 255.255.255.248 vhid 20 carpdev em3 pass sharedsecret advskew 0
EOF
# cat > /etc/hostname.carp2 <<'EOF'
inet 10.10.10.1 255.255.255.0 vhid 30 carpdev em1 pass sharedsecret advskew 0
EOF

# sh /etc/netstart em0 em1 em2 em3 carp0 carp1 carp2 pfsync0
  # Activate without reboot
```

### PF Policy (both nodes)

```pf
## /etc/pf.conf — Edge HA + Multi-WAN

set skip on { lo, pfsync0 }

# Interfaces and macros
lan   = "em1"
wan_a = "em0"
wan_b = "em3"
lan_net = "10.10.10.0/24"
gw_a = "198.51.100.1"
gw_b = "203.0.113.1"
web_svc = "10.10.10.50"

# Normalization
scrub in all max-mss 1452

# Allow HA control-plane
pass on $lan proto carp
pass on $wan_a proto carp
pass on $wan_b proto carp
pass on em2 proto pfsync

# NAT per uplink
match out on $wan_a inet from $lan_net to any nat-to ($wan_a)
match out on $wan_b inet from $lan_net to any nat-to ($wan_b)

# Base policy
block all

# Inbound to firewall itself with symmetric reply
pass in on $wan_a proto { tcp udp icmp } to ($wan_a) reply-to ($wan_a $gw_a) keep state
pass in on $wan_b proto { tcp udp icmp } to ($wan_b) reply-to ($wan_b $gw_b) keep state

# Outbound policy from LAN:
# - Bulk to ISP-B
# - Default to ISP-A
pass in on $lan proto { tcp udp } from $lan_net to any port { 873, 6881:6999 } \
    route-to ($wan_b $gw_b) keep state
pass in on $lan from $lan_net to any \
    route-to ($wan_a $gw_a) keep state

# Inbound service on both ISPs (HTTPS)
rdr on $wan_a proto tcp to ($wan_a) port 443 -> $web_svc
pass in on $wan_a proto tcp to $web_svc port 443 reply-to ($wan_a $gw_a) keep state
rdr on $wan_b proto tcp to ($wan_b) port 443 -> $web_svc
pass in on $wan_b proto tcp to $web_svc port 443 reply-to ($wan_b $gw_b) keep state

# LAN to firewall services (SSH/HTTPS example)
pass in on $lan proto tcp from $lan_net to ($lan) port { 22, 443 } keep state

# Outbound from the firewall
pass out on { $wan_a, $wan_b } proto { tcp udp icmp } from (self) to any keep state
```

**Verification (either node):**

```sh
$ ifconfig carp0 carp1 carp2 | egrep 'vhid|status'
  # MASTER/BACKUP per VIP
$ pfctl -vvsr | egrep 'route-to|reply-to|proto carp|pfsync'
  # Policy pins and HA allowances
$ pfctl -vvss | head
  # Replicated states present on both nodes
$ tcpdump -ni pfsync0
  # Flow updates on SYNC
```

---

## Pattern 2 — Campus/Branch Routed Access with Local Services

**Scope.** Two routers provide VLAN gateways with CARP, advertise IPv6 via `rad(8)`, and offer internal recursion (Unbound), authoritative zones (NSD), DHCP, and anycast resolver placement.

### Gateways and CARP (R1/R2)

Assume trunk `em1`, VLAN 10 Users `10.10.10.0/24`, VLAN 20 Servers `10.20.20.0/24`, VIPs `.1`, router unicast `.2` (R1) / `.3` (R2).

```
# /etc/hostname.vlan10 (R1)
vlandev em1
vlan 10
inet 10.10.10.2 255.255.255.0
up
# /etc/hostname.vlan20 (R1)
vlandev em1
vlan 20
inet 10.20.20.2 255.255.255.0
up
# /etc/hostname.carp0 (R1)
inet 10.10.10.1 255.255.255.0 vhid 10 carpdev vlan10 pass shared advskew 0
# /etc/hostname.carp1 (R1)
inet 10.20.20.1 255.255.255.0 vhid 20 carpdev vlan20 pass shared advskew 0
```

Mirror on R2 with `.3` and `advskew 100`.

### Router Advertisements and DNS on a Services Node

`rad(8)` advertises /64s; Unbound serves recursion; NSD serves your zone; DHCP serves IPv4.

```
## /etc/rad.conf
interface vlan10
    prefix 2001:db8:10:10::/64
    rdnss 2001:db8:10:10::53
    dnssl corp.example
interface vlan20
    prefix 2001:db8:10:20::/64
    rdnss 2001:db8:10:20::53
    dnssl corp.example
```
```
## /var/unbound/etc/unbound.conf (excerpt)
server:
    interface: 10.10.10.53
    interface: 10.20.20.53
    access-control: 10.10.10.0/24 allow
    access-control: 10.20.20.0/24 allow
    auto-trust-anchor-file: "/var/unbound/db/root.key"
```
```
## /etc/nsd.conf (excerpt)
server:
    ip-address: 10.20.20.53
    zonesdir: "/var/nsd/zones"
zone:
    name: "corp.example"
    zonefile: "corp.example.zone"
```
```
## /etc/dhcpd.conf (excerpt)
subnet 10.10.10.0 netmask 255.255.255.0 {
    range 10.10.10.100 10.10.10.200;
    option routers 10.10.10.1;
    option domain-name-servers 10.10.10.53;
}
subnet 10.20.20.0 netmask 255.255.255.0 {
    range 10.20.20.100 10.20.20.200;
    option routers 10.20.20.1;
    option domain-name-servers 10.20.20.53;
}
```

### Anycast Resolver Address

Publish `10.10.255.53/32` from two resolvers and ECMP it on R1/R2.

```sh
# ifconfig lo0 alias 10.10.255.53/32
  # On each resolver

# route add -mpath -host 10.10.255.53 10.20.20.11
# route add -mpath -host 10.10.255.53 10.20.20.12
  # On each router; confirm with 'route -n show'
```

**Verification:**

```sh
$ ifconfig vlan10 vlan20 carp0 carp1 | egrep 'status|vhid'
$ drill @10.10.10.53 openbsd.org A
$ ping -n -c 3 10.10.255.53
$ ndp -an | head
```

---

## Pattern 3 — Regional POP

Two options are common:

### A) MPLS LDP Core with L3 VPNs

Enable MPLS on core links, run `ldpd(8)`, and connect customer VRFs via [mpe(4)](https://man.openbsdhandbook.com/mpe.4/)
. BGP VPN policy is defined in OpenBGPD.

```sh
# ifconfig em0 mpls
# ifconfig em1 mpls
# rcctl enable ldpd ; rcctl start ldpd
```

```
## /etc/ldpd.conf (core interfaces)
router-id 192.0.2.1
address-family ipv4 {
    interface em0
    interface em1
}
```

Bind a customer VRF (rdomain 10) to MPLS with `mpe0`:

```sh
# ifconfig em2 rdomain 10 inet 10.10.10.1/24
# ifconfig mpe0 create
# ifconfig mpe0 rdomain 10 mplslabel 2000 inet 192.198.0.1/32 up
```

Announce customer prefixes via OpenBGPD VPN AFI/SAFI (see the OpenBGPD chapter for `bgpd.conf` `vpn` stanza and iBGP toward other PEs).

**Verification:**

```sh
$ ldpctl show neighbors
$ ldpctl show lib | head
$ route -T 10 -n show
```

### B) Encrypted Hub-and-Spoke (IKEv2)

For smaller POPs, use `iked(8)` to connect spokes to a regional hub.

```sh
## /etc/iked.conf — spoke to hub (selectors per site)
ikev2 "spoke-hub" active esp from 10.30.0.0/24 to 10.0.0.0/16 \
    peer 198.51.100.50 \
    ikesa enc aes-256-gcm prf sha256 group 14 \
    childsa enc aes-256-gcm \
    psk "change-me"
```

```sh
# rcctl enable iked ; rcctl start iked
# ikectl show sa ; ikectl show flows
```

**POP PF skeleton (both options):**

```pf
## /etc/pf.conf — POP edge (excerpt)
set skip on lo
block all
pass out on egress proto { tcp udp icmp } keep state
# Allow management from NOC
pass in on egress proto tcp from 198.51.100.0/24 to (egress) port 22 keep state
# Permit customer LANs on VRF or tunnel interfaces (adjust)
pass in on enc0 from 10.30.0.0/24 to any keep state
# Or, for MPLS VRF:
# pass in on mpe0 from 10.30.0.0/24 to any keep state
```

---

## Verification

- **HA edges:** `ifconfig carp*`, `pfctl -vvsr` counters for `route-to`/`reply-to`, `tcpdump -ni pfsync0`.
- **Campus services:** `drill`, `unbound-control status`, `nsd-control stats`, `dhcpd` leases, `rad(8)` advertising on correct VLANs.
- **POP:** `ldpctl show neighbors/lib`, `bgpctl show summary` (when applicable), `ikectl show sa/flows`.
- **Telemetry:** `tcpdump -i pflog0`, `pfctl -vvsr | grep log`, `ntpctl -s status`.

## Troubleshooting

- **Asymmetric paths at the edge.** Missing `reply-to` on WAN rules for inbound NAT is the most common cause. Add it per uplink and flush affected states.
- **State desynchronization.** Verify `pfsync0` `syncdev` and PF allowances. Ensure both nodes run the same ruleset and NAT model.
- **IPv6 clients fail SLAAC.** Confirm `rad(8)` advertises exactly one /64 per L2, ICMPv6 is permitted, and that only the intended routers send RAs.
- **Anycast black holes.** Remove a failed next-hop from the ECMP route or automate withdrawals; ensure resolvers actually bind the anycast address.
- **POP label gaps.** For MPLS, confirm core interfaces are `mpls`-enabled and present in `ldpd.conf`. For encrypted spokes, check UDP 500/4500 reachability and PSK alignment.
- **Service port conflicts.** Keep authoritative (NSD) and recursive (Unbound) on distinct IPs or ports; never expose recursion to the Internet.

## See Also

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- Related: [**High Availability and State Replication**](high-availability.md)
- Related: [**Multi-WAN and Policy-Based Routing**](multi-wan-policy-routing.md)
- Related: [**Network Services at Scale**](network-services-at-scale.md)
- Related: [**MPLS and Label Distribution**](mpls.md)
