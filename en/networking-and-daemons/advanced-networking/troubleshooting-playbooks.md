# Troubleshooting Playbooks

## Synopsis

This chapter provides deterministic troubleshooting playbooks for common production faults: High Availability (HA) role flaps, asymmetric paths at multi-WAN edges, tunnel MTU black holes, IPv6 Neighbor Discovery (ND) and Router Advertisement (RA) problems, and practical Quality-of-Service (QoS) verification. Each playbook uses base tools only: packet filter policy in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
inspected with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
, interfaces via [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
, live captures with [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
, system controls with [sysctl(8)](https://man.openbsdhandbook.com/sysctl.8/)
, reachability tests with [ping(8)](https://man.openbsdhandbook.com/ping.8/)
and ping6(8), path inspection with [traceroute(8)](https://man.openbsdhandbook.com/traceroute.8/)
, IPv6 neighbor tools in [ndp(8)](https://man.openbsdhandbook.com/ndp.8/)
, and service-specific helpers such as [ikectl(8)](https://man.openbsdhandbook.com/ikectl.8/)
for IPsec or `relayctl(8)` for relayd when relevant. Device-level features include [carp(4)](https://man.openbsdhandbook.com/carp.4/)
, [pfsync(4)](https://man.openbsdhandbook.com/pfsync.4/)
, [wg(4)](https://man.openbsdhandbook.com/wg.4/)
, [gif(4)](https://man.openbsdhandbook.com/gif.4/)
, and [gre(4)](https://man.openbsdhandbook.com/gre.4/)
.

## Design Considerations

- **Reproduce, then narrow.** Reproduce the failure, capture once near the fault, and use counters and states to avoid guessing.
- **Prefer egress perspective.** At policy edges, states and NAT pin flows to egress. Inspect states before changing routes.
- **One change at a time.** Introduce a reversible change, observe, revert if inconclusive.
- **Clocks and logs.** Ensure accurate time (see [ntpd(8)](https://man.openbsdhandbook.com/ntpd.8/)
  ). Use `/var/log/messages` and PF rule counters to align observations.
- **Safety.** During production incidents, demote a node or isolate a path rather than wholesale restarts. Keep a failsafe PF anchor for management access (see the Hardening chapter).

## Configuration (Operator Toolbox)

Prepare a minimal, reusable toolbox on each device.

```sh
# rcctl enable pflogd ; rcctl start pflogd
  # Persist and start PF log capture to /var/log/pflog

# pfctl -sr | sed -n '1,40p'
  # Snapshot of the active rule order for later correlation

# sysctl net.inet.carp
  # Baseline CARP preempt/log/demotion values (on HA nodes)

# install -d -m 700 /root/ts ; date > /root/ts/incident.start
  # Create a working scratch area with a start timestamp
```

---

## Playbook 1 — HA Role Flaps (CARP/pfsync)

**Symptoms.** Frequent MASTER↔BACKUP flips, brief outages, or state desynchronization.

**Likely causes.** Unstable SYNC link, VHID/password mismatch, asymmetric upstream reachability, or aggressive preemption.

### Steps

1. **Observe CARP status and demotion.**

```sh
$ ifconfig carp0 carp1
  # Expect one MASTER and one BACKUP per VIP
$ sysctl net.inet.carp.demotion
  # Non-zero indicates a fault (e.g., interface down, pfsync issues)
$ tail -n 100 /var/log/messages | egrep -i 'carp|pfsync'
  # Timeline of role changes and reasons
```

2. **Verify the SYNC domain and state replication.**

```sh
$ ifconfig pfsync0
  # Confirm syncdev and UP status
# tcpdump -ni pfsync0
  # Observe steady updates under load; bursts during new flows/failover
$ pfctl -vvss | head
  # States present with creators and counters on both nodes
```

3. **Check HA control-plane allowances and VHID/secret.**

```sh
$ pfctl -sr | egrep 'proto (carp|pfsync)'
  # Rules allowing carp and pfsync
$ grep -H carpdev /etc/hostname.carp*
  # Same VHIDs and sharedsecret on both nodes?
```

4. **Stabilize now, investigate next.**

```sh
# ifconfig carp0 advskew 240 ; ifconfig carp1 advskew 240
  # Demote this node (prefer peer as MASTER)
# sysctl net.inet.carp.preempt=0
  # Temporarily disable preemption to avoid ping-pong
```

5. **Common fixes.** Replace or isolate the SYNC cable/VLAN, ensure `set skip on pfsync0` in PF, align rulesets and NAT, and only then re-enable preemption and restore `advskew`.

---

## Playbook 2 — Asymmetric Paths and Broken Inbound NAT

**Symptoms.** Some destinations work; others time out. Inbound services connect but replies exit via the wrong WAN. Outbound policies appear ignored.

**Likely causes.** Missing `reply-to` on WAN pass rules for NAT’d services, stale states, or rule ordering issues for `route-to`.

### Steps

1. **Confirm rule matches and state egress.**

```sh
$ pfctl -vvsr | egrep 'route-to|reply-to'
  # Ensure per-WAN pinning rules exist and are ordered correctly
$ pfctl -vvss | egrep 'em0|em3|Gateway'
  # Inspect states for selected egress interface/gateway
```

2. **Capture on both WANs while reproducing.**

```sh
# tcpdump -ni em0 host <client-or-server> and port 443
# tcpdump -ni em3 host <client-or-server> and port 443
  # Verify which interface carries SYN/ACK or replies
```

3. **Apply corrective actions.**

```sh
# pfctl -K 10.10.10.50
  # Kill states for a published server so new states pick up corrected policy
# pfctl -f /etc/pf.conf
  # Reload after adding missing reply-to/route-to
$ route -n show -inet | head
  # Remember: system default route affects firewall-originated traffic only
```

---

## Playbook 3 — Tunnel MTU / Black-Hole Detection (IPsec, WireGuard, gif/gre)

**Symptoms.** Small packets succeed; large transfers stall. TLS handshakes complete but large responses hang. ICMP “Too Big” not seen.

**Likely causes.** Encapsulation overhead exceeding underlay MTU; PMTUD blocked; excessive MSS.

### Steps

1. **Probe with increasing payload sizes.**

```sh
$ ping -n -c 3 -s 1400 <remote-ipv4>
  # Try large payload; decrease (e.g., 1380, 1360) until it works
$ ping6 -n -c 3 -s 1400 <remote-ipv6>
  # IPv6 is more sensitive; adjust until success
```

2. **Inspect the tunnel and adjust MTU/MSS.**

- **IPsec (IKEv2):**

```sh
$ ikectl show sa ; ikectl show flows
  # Confirm SAs/flows up
# tcpdump -ni enc0 host <peer-inside>
  # Plaintext post-decrypt traffic
```

Add a conservative scrub during mitigation, then refine:

```
## pf.conf excerpt — temporary MSS clamp
scrub in on egress max-mss 1452
```

- **WireGuard:**

```sh
$ ifconfig wg0
  # Review wgpeer stats; note current MTU
# ifconfig wg0 mtu 1380
  # Lower MTU to accommodate overhead
```

- **gif/gre:**

```sh
$ ifconfig gif0 ; ifconfig gre0
  # Check configured MTU
# ifconfig gif0 mtu 1400
  # Lower and re-test; adjust per path measurements
```

3. **Confirm ICMP visibility.**

```sh
# tcpdump -ni egress icmp or icmp6
  # Look for “Fragmentation needed” (v4) or “Packet Too Big” (v6)
```

---

## Playbook 4 — IPv6 ND and RA Failures (Hosts Not Getting IPv6)

**Symptoms.** Clients lack global IPv6, show only link-local, or lose default route. Duplicate Address Detection (DAD) failures appear.

**Likely causes.** RA not sent on the right interface, ICMPv6 blocked, multiple routers advertising, or stale ND cache.

### Steps

1. **Check the router side (RA sender).**

```sh
# rcctl check rad
  # RA daemon running?
$ ifconfig em1 | grep inet6
  # Router has an on-link address for the /64
# tcpdump -ni em1 icmp6
  # Observe Router Advertisements on the segment
```

2. **Allow mandatory ICMPv6 in PF and reload.**

```pf
## pf.conf excerpt — ensure ICMPv6 is permitted
pass inet6 proto icmp6 keep state
```

```sh
# pfctl -f /etc/pf.conf
```

3. **Inspect from a host (client side).**

```sh
$ ndp -an | head
  # Router entries should appear with reachable MAC
# rcctl check slaacd
  # Autoconf client running?
$ route -n show -inet6 | egrep '::/0|2001:'
  # Default route via router; global address present
```

4. **Resolve duplicates or rogue RAs.**

```sh
$ ndp -an | egrep -i 'incomplete|failed'
  # Identify anomalies
# ndp -d <IPv6-or-mac>
  # Clear a problematic cache entry (repopulates via ND)
```

If two routers advertise the same /64 unintentionally, disable RA on the retiring router first, then withdraw addresses.

---

## Playbook 5 — QoS Verification (PF Queues)

**Symptoms.** Interactive traffic suffers under load; queue counters do not move as expected.

**Likely causes.** Classification direction mismatch, incorrect queue names, or parents allowing unlimited borrowing.

### Steps

1. **Confirm the queue tree and counters.**

```sh
$ pfctl -vvsq
  # Tree, bandwidth/min/max, packets/bytes per queue
```

2. **Validate classification rules and hits.**

```sh
$ pfctl -vvsr | egrep 'set queue|set prio'
  # Verify intended rules exist and order is correct
# tcpdump -ni egress port 22 or port 443
  # Generate traffic and correlate with queue counters
```

3. **Exercise under load and observe.**

```sh
$ scp largefile user@remote:/dev/null &
  # Create bulk load
$ ssh user@remote
  # Verify keystroke latency while watching 'pfctl -vvsq'
```

4. **Adjust and re-test.**

- Lower `bandwidth` or set `max` on default/bulk queues.
- Use two-queue assignment, for example `set queue (web_bulk, web_prio)`, to prioritize ACKs.

---

## Verification

- **Repeatable tests.** Keep a short script that pings, curls a known endpoint, and emits a syslog marker before and after changes.
- **Counters before/after.** Record `pfctl -vvsr`, `pfctl -vvsq`, and `pfctl -s state | wc -l` before and after each step.
- **Packet captures.** Save `tcpdump` traces with timestamps during failures; retain them alongside the PF snapshot for later analysis.

```sh
$ date && pfctl -vvsr | wc -l && pfctl -vvsq | wc -l
  # Quick sanity of rule/queue counts with a timestamp
# tcpdump -ni egress -w /root/ts/egress.pcap host <peer> and port <svc>
  # Capture for offline review
```

## Troubleshooting

- **Changes did not take effect.** Ensure you reloaded PF (`pfctl -f`) and killed stale states where path selection must change.
- **Data path differs from captures.** Verify you are capturing on the correct interface (physical vs. VLAN vs. tunnel) and that `set skip on` is not hiding traffic from PF.
- **HA “works” but drops occur.** Confirm identical rulesets and NAT on both nodes; states replicate, rules do not.
- **MTU still inconsistent.** Check intermediate links (PPPoE, VPNs, or VLAN stacks). If ICMP control is filtered upstream, keep MSS clamping in place.

## See Also

- [Networking](../../install-and-configure/networking.md)
- [OpenBGBD](../services/networking-services/openbgpd.md)
- Related: [**High Availability and State Replication**](high-availability.md)
- Related: [**Multi-WAN and Policy-Based Routing**](multi-wan-policy-routing.md)
- Related: [**VPN and Cryptographic Tunneling**](vpn-and-crypto-tunneling.md)
- Related: [**IPv6 at Scale**](ipv6-at-scale.md)
- Related: [**QoS and Traffic Shaping**](qos-traffic-shaping.md)
