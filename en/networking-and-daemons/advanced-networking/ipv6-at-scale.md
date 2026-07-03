# IPv6 at Scale

## Synopsis

This chapter describes operating IPv6 at scale on OpenBSD: interface addressing, Router Advertisements (RA), Neighbor Discovery (ND), upstream integration, and production guardrails. It uses the in-base router advertisement daemon [rad(8)](https://man.openbsdhandbook.com/rad.8/)
with configuration in [rad.conf(5)](https://man.openbsdhandbook.com/rad.conf.5/)
, client autoconfiguration with [slaacd(8)](https://man.openbsdhandbook.com/slaacd.8/)
, ND inspection via [ndp(8)](https://man.openbsdhandbook.com/ndp.8/)
, interface management with [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
, and packet filtering in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
applied by [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
. Use these patterns for campus segments, data-center pods, and ISP edges where you operate multiple /64s or larger aggregates.

## Design Considerations

- **Prefix plan.** Allocate one /64 per L2 segment. For campuses and pods, obtain at least a /56 (256 × /64) or /48 (65,536 × /64) from your provider. Use hierarchical addressing that mirrors your failure domains and routing boundaries.
- **Forwarding model.** Route between VLANs; avoid L2 extension across failure domains. IPv6 first-hop redundancy can reuse the CARP pattern from the HA chapter.
- **Router Advertisements.** RAs convey on-link prefixes, default gateway, DNS (RDNSS), and search domains (DNSSL). Keep RA timers consistent and advertise exactly one /64 per L2.
- **ND and ICMPv6.** Do not block ICMPv6. Neighbor Discovery, Path MTU Discovery, and error handling depend on it.
- **Prefix Delegation (PD).** Upstreams may delegate a /56 or /60 dynamically. Plan how you subdivide and advertise the delegated space to downstream L2s. Review the release notes for OpenBSD 7.8 and [slaacd(8)](https://man.openbsdhandbook.com/slaacd.8/)
  for current PD client capabilities.
- **NPTv6.** Prefer proper routing. Only consider Network Prefix Translation (NPTv6) for constrained renumbering scenarios, and test thoroughly. See [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
  for current support and caveats.
- **Operations at scale.** Standardize RA defaults, centralize logging, and instrument ND tables and error counters per segment. Use change windows to test DAD, RA reception, and PMTU.

## Configuration

The examples assume:

- Router with interfaces `em1` (LAN-A VLAN 10), `em2` (LAN-B VLAN 20), `em0` (WAN).
- Assigned aggregate `2001:db8:10::/56`, where `2001:db8:10:0010::/64` is LAN-A and `2001:db8:10:0020::/64` is LAN-B.

### 1) System forwarding and interface addressing (router)

Enable IPv4/IPv6 forwarding and assign IPv6 addresses to the router’s LAN interfaces.

```sh
# printf '%s\n' \
    'net.inet.ip.forwarding=1' \
    'net.inet6.ip6.forwarding=1' \
    >> /etc/sysctl.conf
  # Persist L3 forwarding

# sysctl net.inet.ip.forwarding=1 net.inet6.ip6.forwarding=1
  # Apply immediately

# cat > /etc/hostname.em1 <<'EOF'
inet6 2001:db8:10:10::1 64
up
EOF
  # LAN-A: router address and prefix length

# cat > /etc/hostname.em2 <<'EOF'
inet6 2001:db8:10:20::1 64
up
EOF
  # LAN-B: router address and prefix length

# sh /etc/netstart em1 em2
  # Bring the interfaces up now
```

### 2) Router Advertisements with rad(8)

Advertise one /64 per L2. Include RDNSS and DNSSL so hosts learn DNS settings via RA.

```
## /etc/rad.conf — advertise /64s and DNS on two interfaces

# LAN-A (VLAN 10)
interface em1
    prefix 2001:db8:10:10::/64
    rdnss 2001:db8:10:10::53
    dnssl example.internal

# LAN-B (VLAN 20)
interface em2
    prefix 2001:db8:10:20::/64
    rdnss 2001:db8:10:20::53
    dnssl example.internal
```

```sh
# rcctl enable rad
  # Start at boot
# rcctl start rad
  # Launch now; begin sending RAs
```

> Keep one advertising router per segment during initial rollout. If you deploy redundant routers with CARP, ensure both run `rad(8)` and that the virtual default gateway address appears in DNS and documentation.

### 3) Client autoconfiguration (hosts)

OpenBSD hosts use [slaacd(8)](https://man.openbsdhandbook.com/slaacd.8/)
) to form addresses from RA. Ensure the interface requests autoconfiguration.

```
## /etc/hostname.em1 on a host
inet6 autoconf
up
```

```sh
# rcctl enable slaacd
# rcctl start slaacd
  # The host will configure a global address and default route from RA
```

### 4) PF allowances for IPv6 control-plane and data

Permit essential ICMPv6, allow LAN traffic, and apply a conservative scrub for PMTU stability. Adjust to your security model as needed.

```pf
## /etc/pf.conf — minimal IPv6 allowances

set skip on lo

lan = "{ em1, em2 }"
wan = "em0"

block all

# ICMPv6 is mandatory for ND and PMTU; allow it on all interfaces
pass inet6 proto icmp6 keep state

# Permit LAN IPv6; constrain further per policy
pass in on $lan inet6 from { 2001:db8:10:10::/64, 2001:db8:10:20::/64 } to any keep state

# Outbound to WAN
pass out on $wan inet6 from any to any keep state

# Conservative scrub to avoid PMTU black holes during rollout
scrub in on $wan inet6
```

```sh
# pfctl -f /etc/pf.conf
# pfctl -sr | grep icmp6
  # Verify ICMPv6 allowance is active
```

### 5) Upstream integration and Prefix Delegation (overview)

When an ISP delegates a prefix (for example, a /56), subdivide it deterministically (for example, VLAN-ID ↔ 0xNN in `2001:db8:10:NN::/64`) and advertise per-L2 via `rad(8)`. Depending on release and provider requirements, prefix delegation may be delivered dynamically. Review OpenBSD 7.8 documentation for [slaacd(8)](https://man.openbsdhandbook.com/slaacd.8/)
and plan operational procedures (lease monitoring, re-advertisement on change). Where PD is not available, use static assignment from your aggregate.

### 6) Notes on NPTv6

Where renumbering pressure is high and routing is not an option, NPTv6 can translate one global /64 to another while preserving host bits. Prefer native routing; reserve NPTv6 for edge-cases and lab validation. For syntax and constraints, consult [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
for your OpenBSD 7.8 release and test in an isolated segment before production.

## Verification

- **Addresses and routes**: confirm per-L2 /64s and the default route.

```sh
$ ifconfig em1 em2 | grep inet6
  # Verify router and host global addresses
$ route -n show -inet6
  # Confirm default route via the on-link router on hosts; confirm connected /64s on the router
```

- **Neighbor Discovery**: inspect neighbor caches and router entries with [ndp(8)](https://man.openbsdhandbook.com/ndp.8/)
  .

```sh
$ ndp -an
  # Neighbor and router cache; look for stale or incomplete entries
```

- **RAs and ND on the wire**: capture ICMPv6 on a LAN with [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
  .

```sh
# tcpdump -ni em1 icmp6
  # Expect Router Advertisements, Neighbor Solicitations/Advertisements
```

- **DNS via RDNSS**: verify that hosts learned DNS servers via RA.

```sh
$ getent hosts www.example.internal
  # Name resolution using learned RDNSS
```

## Troubleshooting

- **Hosts do not get IPv6 addresses.** Confirm `rad(8)` is running and advertising on the correct interface, `net.inet6.ip6.forwarding=1` on routers, and that PF is not blocking ICMPv6. Use `tcpdump -ni emX icmp6` to observe RAs.
- **Duplicate Address Detection (DAD) failures.** If an address remains tentative or is marked duplicate, check for stray RA senders or misconfigured anycast. Inspect `ndp -an` for conflicting MACs.
- **Intermittent reachability or stalls.** Packet Too Big messages may be blocked. Ensure ICMPv6 is permitted end-to-end and verify PMTU with large pings (`ping -s`).
- **Unexpected router selection.** Ensure only the intended routers advertise on each L2. During migrations, disable RA on the retiring router first, then withdraw prefixes.
- **Prefix changes with PD.** When the delegated prefix changes, update interface addresses and `rad.conf(5)` promptly. Automate downstream updates or use deterministic mapping from the delegated aggregate.
- **Firewall denies IPv6.** Verify that your policy contains explicit `inet6` rules. Mixed `inet`/`inet6` environments often mistakenly permit only IPv4.

## See Also

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- Related: [**QoS and Traffic Shaping**](qos-traffic-shaping.md)
- Related: [**Troubleshooting Playbooks**](troubleshooting-playbooks.md)
