# Multi-WAN and Policy-Based Routing

## Synopsis

This chapter shows how to operate an OpenBSD edge with two or more upstream providers. It covers outbound policy using `route-to`, symmetric return paths with `reply-to`, per-interface NAT, and inbound publishing of services on multiple providers. Configuration lives in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
and is managed with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
. Interface attributes are managed with [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
. Where automated failover is required, integrate link and reachability checks with [ifstated(8)](https://man.openbsdhandbook.com/ifstated.8/)
. Use this pattern when you must split or steer traffic across providers, or need deterministic return paths for inbound NAT without using BGP.

## Design Considerations

- **Topology.** Use distinct physical interfaces per WAN. Example: `em0` for ISP-A and `em3` for ISP-B. Keep a clear **LAN** interface for policy classification.
- **Addressing and gateways.** Each WAN has its own gateway. Example: ISP-A `198.51.100.1`, ISP-B `203.0.113.5`.
- **Asymmetry.** Stateful filtering and NAT tie flows to the egress that created the state. Changing egress requires new states. Plan to clear affected states during failover.
- **NAT alignment.** Per-interface NAT is mandatory for multi-WAN. Avoid a single `egress` NAT when you intend to steer flows across multiple egresses.
- **Policy granularity.** Classify by source networks, application ports, or destinations. Simpler is better. Start with per-subnet policies.
- **Inbound services.** Without BGP or external DNS steering, inbound presence is per-provider. Use `reply-to` to force responses back out the interface that received the connection.
- **Monitoring and automation.** Detect failure of a specific provider before changing policy. Use [ifstated(8)](https://man.openbsdhandbook.com/ifstated.8/)
  to trigger clean policy reloads and optional state resets.
- **MTU.** Mixed-access technologies (for example, PPPoE vs. Ethernet) can introduce fragmentation. Consider a conservative scrub with MSS clamping if you observe PMTUD problems.

## Configuration

The example below implements two uplinks with deterministic outbound and inbound behavior. Adjust interface names and addresses to match your environment.

### Interface assumptions

- `em0` — ISP-A, address `198.51.100.10/29`, gateway `198.51.100.1`
- `em3` — ISP-B, address `203.0.113.10/29`, gateway `203.0.113.5`
- `em1` — LAN, `10.10.10.0/24` users and servers
- Optional server to publish inbound: `10.10.10.50` (HTTPS)

### PF policy with per-WAN NAT, route-to, and reply-to

Place the following in `/etc/pf.conf`. Load and inspect with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
. Syntax is defined in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
.

```sh
## /etc/pf.conf — Multi-WAN and policy-based routing (example)

set skip on lo

# Interfaces and networks
lan      = "em1"
wan_a    = "em0"
wan_b    = "em3"
lan_net  = "10.10.10.0/24"

# Upstream gateways
gw_a = "198.51.100.1"
gw_b = "203.0.113.5"

# Optional convenience macros
web_svc = "10.10.10.50"
tcp_services = "{ 22, 80, 443 }"

# Conservative scrub; adjust as required
scrub in all max-mss 1452

# Per-interface NAT for each uplink
match out on $wan_a inet from $lan_net to any nat-to ($wan_a)
match out on $wan_b inet from $lan_net to any nat-to ($wan_b)

# Base policy
block all

# Inbound control-plane to the firewall itself (per-WAN reply-to for symmetry)
pass in on $wan_a proto { tcp, udp, icmp } to ($wan_a) \
    reply-to ($wan_a $gw_a) keep state
pass in on $wan_b proto { tcp, udp, icmp } to ($wan_b) \
    reply-to ($wan_b $gw_b) keep state

# Outbound policy from LAN:
# - Default via ISP-A
# - Specific subnet or application class via ISP-B
# Order matters: specific policies first, then the default.

# Example: send a source subnet (or specific hosts) via ISP-B
# Replace 10.10.20.0/24 with a real subset if applicable.
# pass in on $lan from 10.10.20.0/24 to any \
#     route-to ($wan_b $gw_b) keep state

# Example: send bulk/backup traffic via ISP-B (by destination ports)
pass in on $lan proto { tcp, udp } from $lan_net to any port { 6881:6999, 873 } \
    route-to ($wan_b $gw_b) keep state

# Default: everything else goes via ISP-A
pass in on $lan from $lan_net to any \
    route-to ($wan_a $gw_a) keep state

# Inbound publishing of a HTTPS service on both providers
# rdr occurs before filtering; provide a pass rule that includes reply-to
# so return traffic follows the same ISP it arrived on.

# ISP-A publication
rdr on $wan_a proto tcp from any to ($wan_a) port 443 -> $web_svc
pass in on $wan_a proto tcp from any to $web_svc port 443 \
    reply-to ($wan_a $gw_a) keep state

# ISP-B publication
rdr on $wan_b proto tcp from any to ($wan_b) port 443 -> $web_svc
pass in on $wan_b proto tcp from any to $web_svc port 443 \
    reply-to ($wan_b $gw_b) keep state

# Allow LAN to firewall services as needed (SSH/HTTPS example)
pass in on $lan proto tcp from $lan_net to ($lan) port $tcp_services keep state

# Outbound from the firewall itself (administration, package fetches)
pass out on $wan_a inet to any keep state
pass out on $wan_b inet to any keep state
```

#### Notes

- `route-to` applies where the packet **enters** PF. For traffic originating on the LAN, match it with `pass in on $lan ... route-to (...)`.
- `reply-to` on the WAN-side pass rules pins return traffic for inbound sessions to the same interface and gateway that received the connection.
- NAT rules are written per uplink so that translation follows the egress selected by policy.

### Operational toggles and state hygiene

During planned failover or while testing, it is useful to adjust policy and reset only the flows that must move.

```sh
# pfctl -f /etc/pf.conf
  # Atomic reload after any change

# ifconfig em0 down
  # Simulate ISP-A failure (bring the interface back up after testing with: ifconfig em0 up)

# pfctl -k 0.0.0.0/0 -k 0.0.0.0/0
  # Flush all IPv4 states so new paths can be chosen
# pfctl -K "10.10.10.0/24"
  # Or selectively kill states for the LAN prefix only

# tcpdump -ni em3 icmp or port 443
  # Observe traffic moving to ISP-B during a test
```

If you automate uplink health detection with [ifstated(8)](https://man.openbsdhandbook.com/ifstated.8/)
, ensure that the action sequence includes a clean `pfctl -f` of a prepared policy and a targeted state reset for affected prefixes or applications.

## Verification

Use PF counters, live captures, and system routes to verify that selection and symmetry work as intended.

- **Rule counters and states** with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
  :

```sh
$ pfctl -vvsr
  # Verify that specific "route-to" rules are matching and increasing counters
$ pfctl -vvss | egrep 'em0|em3'
  # Inspect states; egress interface and gateway selection appear in state details
```

- **Path validation** with [traceroute(8)](https://man.openbsdhandbook.com/traceroute.8/)
  and [ping(8)](https://man.openbsdhandbook.com/ping.8/)
  ):

```sh
$ traceroute -n 1.1.1.1
  # From a LAN host through the firewall; expect ISP-A by default
$ traceroute -n 9.9.9.9
  # For a destination matched by a policy to ISP-B, expect the ISP-B path
$ ping -n -c 3 8.8.8.8
  # Basic reachability; repeat while toggling policies
```

- **Interface-level observation** with [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
  :

```sh
# tcpdump -ni em0 host 8.8.8.8 or port 443
  # Confirm flows selected for ISP-A egress
# tcpdump -ni em3 host 9.9.9.9 or port 443
  # Confirm flows selected for ISP-B egress
```

- **System routes** with [route(8)](https://man.openbsdhandbook.com/route.8/)
  :

```sh
$ route -n show -inet
  # The system default route is not authoritative for PF "route-to" decisions,
  # but it matters for traffic sourced by the firewall itself
```

## Troubleshooting

- **Asymmetric return paths.** Missing or incorrect `reply-to` on WAN pass rules for inbound services causes replies to leave via the wrong ISP. Add `reply-to` on each WAN.
- **NAT mismatch.** If clients on LAN lose connectivity when a policy switches egress, ensure per-interface `match out on $wan_* nat-to ($wan_*)` rules exist for each uplink.
- **Flows stick to old egress.** Existing states pin to the original path. Kill only the affected states using `pfctl -K <subnet>` or perform a targeted application restart.
- **Performance issues after path change.** MTU or MSS differences between providers can break PMTUD. Keep a conservative `scrub ... max-mss` during testing and refine later.
- **Policy does not match.** Rule ordering matters. Place specific selectors before the default LAN routing rule. Confirm with `pfctl -vvsr`.
- **Firewall-originated traffic uses the wrong ISP.** `route-to` does not affect traffic sourced by the firewall. Adjust the system default route with [route(8)](https://man.openbsdhandbook.com/route.8/)
  or add application-specific source addresses.

## See Also

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- Related: [**High Availability and State Replication**](high-availability.md)
- Related: [**Troubleshooting Playbooks**](troubleshooting-playbooks.md)
