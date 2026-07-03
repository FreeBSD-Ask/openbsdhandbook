# Large-Scale L2 and L3 Design

## Synopsis

This chapter presents practical patterns for scaling Layer-2 and Layer-3 design on OpenBSD: VLAN planning with [vlan(4)](https://man.openbsdhandbook.com/vlan.4/)
, routed access (SVI-style interfaces) using [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
and [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
, first-hop redundancy with [carp(4)](https://man.openbsdhandbook.com/carp.4/)
, and anycast service placement (static ECMP or BGP-based; see OpenBGPD). Use these patterns to bound failure domains, simplify operations, and provide deterministic paths for clients and services.

## Design Considerations

- **VLAN strategy.** Allocate one VLAN per failure domain or broadcast scope. Keep names and IDs hierarchical (for example, `SITE-FLOOR-ROLE`). Reserve ranges for infrastructure (routing, management, storage).
- **Routed access over bridged access.** Prefer Layer-3 at access (SVIs on routers) rather than extending Layer-2 across buildings or floors. Use [bridge(4)](https://man.openbsdhandbook.com/bridge.4/)
  only where strictly necessary and enable STP per port when bridging.
- **First-hop redundancy.** Provide a virtual default gateway per VLAN using [carp(4)](https://man.openbsdhandbook.com/carp.4/)
  . Replicate states for firewalls with [pfsync(4)](https://man.openbsdhandbook.com/pfsync.4/)
  .
- **Inter-VLAN policy.** Treat inter-VLAN routing as a policy boundary and enforce it in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
  . Default deny east-west traffic and allow only required flows.
- **Anycast placement.** For resilient local services (for example, recursive DNS) place identical instances close to users. Steer traffic by static ECMP on distribution routers or by iBGP in larger networks (see [bgpd(8)](https://man.openbsdhandbook.com/bgpd.8/)
  ).
- **MTU and fragmentation.** If carrying VLANs over encapsulations, validate effective MTU and clamp MSS during rollout where needed.
- **Documentation and parity.** Keep `hostname.if` files and PF policies under version control across redundant nodes.

## Configuration

The examples build a routed-access campus with two redundant routers (**R1** and **R2**). Each has a trunk uplink `em1` carrying VLANs 10 (Users) and 20 (Servers). Gateways are provided with CARP VIPs; inter-VLAN policy is enforced with PF. Finally, an anycast resolver service is placed using static multi-path routes.

### 1) System forwarding (both routers)

Enable Layer-3 forwarding and persistent settings with [sysctl(8)](https://man.openbsdhandbook.com/sysctl.8/)
and [sysctl.conf(5)](https://man.openbsdhandbook.com/sysctl.conf.5/)
.

```sh
# printf '%s\n' \
    'net.inet.ip.forwarding=1' \
    'net.inet6.ip6.forwarding=1' >> /etc/sysctl.conf
  # Persist IPv4/IPv6 forwarding

# sysctl net.inet.ip.forwarding=1 net.inet6.ip6.forwarding=1
  # Apply immediately
```

### 2) VLAN SVIs and CARP gateways

Assume addressing:

- VLAN 10 (Users): `10.10.10.0/24`, gateway VIP `10.10.10.1`
- VLAN 20 (Servers): `10.20.20.0/24`, gateway VIP `10.20.20.1`

**R1** uses `.2` and preferred `advskew 0`; **R2** uses `.3` and `advskew 100`. CARP runs on the VLAN interfaces as the `carpdev`.

#### R1

```
# /etc/hostname.vlan10
vlandev em1
vlan 10
inet 10.10.10.2 255.255.255.0
up
# /etc/hostname.vlan20
vlandev em1
vlan 20
inet 10.20.20.2 255.255.255.0
up
# /etc/hostname.carp0
inet 10.10.10.1 255.255.255.0 vhid 10 carpdev vlan10 pass sharedsecret advskew 0
# /etc/hostname.carp1
inet 10.20.20.1 255.255.255.0 vhid 20 carpdev vlan20 pass sharedsecret advskew 0
```

```sh
# sh /etc/netstart vlan10 vlan20 carp0 carp1
  # Bring SVIs and VIPs online
```

#### R2

```
# /etc/hostname.vlan10
vlandev em1
vlan 10
inet 10.10.10.3 255.255.255.0
up
# /etc/hostname.vlan20
vlandev em1
vlan 20
inet 10.20.20.3 255.255.255.0
up
# /etc/hostname.carp0
inet 10.10.10.1 255.255.255.0 vhid 10 carpdev vlan10 pass sharedsecret advskew 100
# /etc/hostname.carp1
inet 10.20.20.1 255.255.255.0 vhid 20 carpdev vlan20 pass sharedsecret advskew 100
```

```sh
# sh /etc/netstart vlan10 vlan20 carp0 carp1
```

### 3) Inter-VLAN policy in PF (both routers)

Permit user and server VLANs to their gateways, but restrict east-west traffic by default. Adjust to your policy. Manage with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
.

```pf
## /etc/pf.conf — routed access with default-deny east-west

set skip on lo

users_if  = "vlan10"
servers_if= "vlan20"
users_net = "10.10.10.0/24"
servers_net="10.20.20.0/24"

block all

# ICMP and essential control
pass inet proto icmp keep state
pass inet6 proto icmp6 keep state

# Allow hosts to reach their default gateway (ARP/ND handled by kernel)
pass in on $users_if from $users_net to ($users_if) keep state
pass in on $servers_if from $servers_net to ($servers_if) keep state

# North-south (to WAN/upstream) — example; refine per environment
pass out on egress proto { tcp udp icmp } from { $users_net, $servers_net } to any keep state

# East-west exceptions (allow only required flows)
# Example: users to servers HTTPS
pass in on $users_if proto tcp from $users_net to $servers_net port 443 keep state
```

```sh
# pfctl -f /etc/pf.conf
  # Load policy atomically
# pfctl -sr | egrep 'vlan10|vlan20'
  # Sanity-check rule placement
```

### 4) Anycast resolver placement (static ECMP)

Publish a campus-local anycast resolver IP `10.10.255.53/32` served by two hosts: `10.20.20.11` (in VLAN 20) and `10.20.20.12`. Each resolver also holds `10.10.255.53/32` on its loopback so replies source the anycast address.

#### On each resolver host

```sh
# ifconfig lo0 alias 10.10.255.53/32
  # Bind the anycast address locally
```

#### On each router (R1 and R2): add equal-cost routes

Use [route(8)](https://man.openbsdhandbook.com/route.8/)
with `-mpath` to install multiple next-hops. Traffic will load-share; withdrawals remove a path.

```sh
# route add -mpath -host 10.10.255.53 10.20.20.11
  # First next-hop via Server-A
# route add -mpath -host 10.10.255.53 10.20.20.12
  # Second next-hop via Server-B
# route -n show | grep 10.10.255.53
  # Confirm two active paths
```

> For large networks, prefer iBGP anycast: each resolver originates `10.10.255.53/32` from a loopback, and distribution routers learn it via BGP. See [bgpd.conf(5)](https://man.openbsdhandbook.com/bgpd.conf.5/)
> and the OpenBGPD chapter.

### 5) Optional: bridging at the edge (constrained scope)

Where bridging is unavoidable (for example, a small lab), constrain the broadcast domain and enable STP on member ports. Manage with [bridge(4)](https://man.openbsdhandbook.com/bridge.4/)
.

```
## /etc/hostname.bridge0 — minimal, with STP on access port
add em2
add em3
stp em2
stp em3
up
```

```sh
# sh /etc/netstart bridge0
# ifconfig bridge0
  # Verify members and STP state
```

## Verification

- **SVIs and VIPs up:**

```sh
$ ifconfig vlan10 vlan20 | egrep 'inet |status'
  # Check router addresses per VLAN
$ ifconfig carp0 carp1
  # Expect MASTER on R1 and BACKUP on R2 (or vice versa after tests)
```

- **Inter-VLAN policy:**

```sh
$ pfctl -vvsr | egrep 'vlan10|vlan20|pass in on'
  # Confirm expected allow rules and counters increase under test
```

- **Anycast reachability and path diversity:**

```sh
$ ping -n -c 3 10.10.255.53
  # Resolver anycast reachable
$ route -n show -host 10.10.255.53
  # Two next-hops installed (mpath)
$ traceroute -n 10.10.255.53
  # Observe path; withdraw one next-hop and repeat to see failover
```

- **On-wire:**

```sh
# tcpdump -ni vlan10 arp or icmp
  # Observe gateway ARP/ICMP during tests
# tcpdump -ni vlan20 host 10.10.255.53
  # Anycast flows toward resolvers
```

## Troubleshooting

- **Clients cannot reach the gateway.** Verify that VLAN tags match on the switch and `vlandev` on the router. Confirm CARP is MASTER on one node and that the VIP is present on the correct VLAN.
- **Inter-VLAN traffic bypasses policy.** Ensure routing occurs on the OpenBSD routers (SVIs) and not on a downstream device. Review PF rule order with `pfctl -vvsr`.
- **CARP flaps.** Check `net.inet.carp.demotion` and link stability. Mismatched `advskew` or missing `pass sharedsecret` across nodes can cause role churn.
- **Anycast black hole.** If one resolver is down but the route remains, remove the failed next-hop (`route delete -host 10.10.255.53 10.20.20.11`) or automate health checks (see `ifstated(8)`). For BGP anycast, rely on session withdrawal.
- **STP not preventing loops.** Ensure STP is enabled per bridge member and that there is only one bridged path between segments. For larger networks, avoid bridging and move the boundary to Layer-3.
- **Asymmetric paths and MTU issues.** Validate that all trunks carry the same VLAN set and MTU. Use conservative MSS clamping temporarily if needed while testing.

## See Also

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- Related: [**High Availability and State Replication**](high-availability.md)
- Related: [**MPLS and Label Distribution**](mpls.md)
- Related: [**Reference Architectures**](reference-architectures.md)
