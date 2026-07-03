# High Availability and State Replication

## Synopsis

This chapter describes building a two-node OpenBSD firewall or gateway cluster with [carp(4)](https://man.openbsdhandbook.com/carp.4/)
virtual IPs, [pfsync(4)](https://man.openbsdhandbook.com/pfsync.4/)
state replication, and optional service failover using [relayd(8)](https://man.openbsdhandbook.com/relayd.8/)
. It provides canonical configurations for inside and outside virtual IPs (VIPs), a dedicated state-sync network, and safe failover behavior. Use this pattern whenever an outage of a single gateway or firewall would impact production traffic. Packet filtering is configured in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
and managed at runtime with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
.

## Design Considerations

- **Topology.** Use three physical interfaces per node: **WAN**, **LAN**, and **SYNC**. Place [pfsync(4)](https://man.openbsdhandbook.com/pfsync.4/)
  on a private L2 segment with no other traffic.
- **Addressing.** Assign each node a fixed unicast address on WAN and LAN, and create per-segment CARP VIPs with unique VHIDs. Example:
  - WAN: node A `203.0.113.2/29`, node B `203.0.113.3/29`, VIP `203.0.113.1/29`, VHID 10
  - LAN: node A `10.10.10.2/24`, node B `10.10.10.3/24`, VIP `10.10.10.1/24`, VHID 20
- **Priority and preemption.** Lower `advskew` wins. Enable `net.inet.carp.preempt=1` so the preferred node retakes MASTER after repair.
- **Isolation and filtering.** Allow `proto carp` on member interfaces and `proto pfsync` only on the SYNC interface. Skip filtering on the SYNC interface.
- **Failure domains.** A SYNC link failure must trigger CARP demotion so a healthy peer becomes MASTER. Monitor upstream reachability if asymmetric failures are possible.
- **Configuration parity.** Keep identical PF and service configuration on both nodes. States replicate; rules do not.
- **Operational guardrails.** Test failover during maintenance windows. Log CARP events at an elevated level during rollout.

## Configuration

The examples assume interface names `em0` (WAN), `em1` (LAN), and `em2` (SYNC). Adjust to your hardware. Interface files are described in [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
. Interface configuration and CARP parameters are managed with [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
. System controls are set with [sysctl(8)](https://man.openbsdhandbook.com/sysctl.8/)
and persisted via [sysctl.conf(5)](https://man.openbsdhandbook.com/sysctl.conf.5/)
. Services are managed with [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
.

### System settings (both nodes)

```sh
# printf '%s\n' \
    'net.inet.carp.preempt=1' \
    'net.inet.carp.log=2' \
    >> /etc/sysctl.conf
  # Enable CARP preemption and raise log verbosity; persists across reboots

# sysctl net.inet.carp.preempt=1 net.inet.carp.log=2
  # Apply immediately at runtime
```

### Node A (preferred MASTER)

```sh
# cat > /etc/hostname.em0 <<'EOF'
inet 203.0.113.2 255.255.255.248
EOF
  # WAN unicast on em0

# cat > /etc/hostname.em1 <<'EOF'
inet 10.10.10.2 255.255.255.0
EOF
  # LAN unicast on em1

# cat > /etc/hostname.em2 <<'EOF'
inet 10.255.255.2 255.255.255.252
EOF
  # SYNC network on em2 (private /30)

# cat > /etc/hostname.carp0 <<'EOF'
inet 203.0.113.1 255.255.255.248 vhid 10 carpdev em0 pass sharedsecret advskew 0
EOF
  # WAN VIP: VHID 10, lowest advskew to prefer MASTER

# cat > /etc/hostname.carp1 <<'EOF'
inet 10.10.10.1 255.255.255.0 vhid 20 carpdev em1 pass sharedsecret advskew 0
EOF
  # LAN VIP: VHID 20

# cat > /etc/hostname.pfsync0 <<'EOF'
up syncdev em2
EOF
  # pfsync over em2; multicast by default on the SYNC segment

# sh /etc/netstart em0 em1 em2 carp0 carp1 pfsync0
  # Bring interfaces up now without reboot
```

### Node B (BACKUP)

```sh
# cat > /etc/hostname.em0 <<'EOF'
inet 203.0.113.3 255.255.255.248
EOF

# cat > /etc/hostname.em1 <<'EOF'
inet 10.10.10.3 255.255.255.0
EOF

# cat > /etc/hostname.em2 <<'EOF'
inet 10.255.255.3 255.255.255.252
EOF

# cat > /etc/hostname.carp0 <<'EOF'
inet 203.0.113.1 255.255.255.248 vhid 10 carpdev em0 pass sharedsecret advskew 100
EOF
  # Higher advskew so this node remains BACKUP while A is healthy

# cat > /etc/hostname.carp1 <<'EOF'
inet 10.10.10.1 255.255.255.0 vhid 20 carpdev em1 pass sharedsecret advskew 100
EOF

# cat > /etc/hostname.pfsync0 <<'EOF'
up syncdev em2
EOF

# sh /etc/netstart em0 em1 em2 carp0 carp1 pfsync0
```

### PF policy (both nodes)

The following minimal policy enables pfsync and CARP traffic, skips filtering on the SYNC interface, and allows all on LAN while blocking by default on WAN. Edit to fit your security model. Syntax is defined in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
. Load and introspect with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
.

```sh
## /etc/pf.conf (minimal HA skeleton)

set skip on { lo, pfsync0 }

# Interfaces
wan = "em0"
lan = "em1"
sync = "em2"

# Allow control-plane for HA
pass on $wan proto carp
pass on $lan proto carp
pass on $sync proto pfsync

# Example policy (tighten for production)
block all
pass on $lan
pass out on $wan proto { tcp udp icmp } from ($wan) to any keep state
```

```sh
# pfctl -f /etc/pf.conf
  # Atomic reload of rules
# pfctl -sr
  # Verify rules include proto carp and proto pfsync
```

### Optional: L4 VIP for a service using relayd (both nodes)

Use [relayd.conf(5)](https://man.openbsdhandbook.com/relayd.conf.5/)
to front-end a service VIP on the LAN side that fails over with CARP. Manage the daemon with [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
and runtime with `relayctl(8)`.

```pf
## /etc/relayd.conf (simple TCP health-checked relay)
table <web_backends> { 10.10.10.11, 10.10.10.12 }

relay "www" {
    listen on 10.10.10.1 port 80
    forward to <web_backends> check tcp port 80
}
```

```sh
# rcctl enable relayd
  # Start on boot
# rcctl start relayd
  # Launch now
```

### Optional: event-driven demotion on SYNC or upstream loss

In addition to automatic kernel demotion on certain failures, operators often demote CARP when SYNC breaks or when upstream reachability fails. The simplest operational control is to adjust `advskew` on all CARP VIPs of the affected node. The following example forces a node to BACKUP or returns it to preferred MASTER:

```sh
# ifconfig carp0 advskew 240
  # Heavily demote WAN VIP on this node

# ifconfig carp1 advskew 240
  # Heavily demote LAN VIP on this node

# ifconfig carp0 advskew 0
  # Restore preferred priority for WAN VIP

# ifconfig carp1 advskew 0
  # Restore preferred priority for LAN VIP
```

Automating this decision can be done with policy tooling such as `ifstated(8)` (not shown here; refer to its manual for condition syntax) that triggers demotion on link or reachability failure.

## Verification

- **Interface roles.** Confirm MASTER/BACKUP on each CARP VIP:

```sh
$ ifconfig carp0
  # Shows "MASTER" or "BACKUP" and VHID/advskew
$ ifconfig carp1
$ ifconfig pfsync0
  # Confirms syncdev em2 and up status
```

- **State replication.** Start a connection through the cluster, then verify states exist on both nodes with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
  :

```sh
$ pfctl -vvss | head
  # Show replicated states with creators and age/counters
```

- **Wire-level sync.** Observe pfsync traffic on the SYNC interface with [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
  :

```sh
# tcpdump -n -e -ttt -i pfsync0
  # Expect steady state updates; bursts during new flows and failover
```

- **CARP health.** Inspect CARP sysctls with [sysctl(8)](https://man.openbsdhandbook.com/sysctl.8/)
  :

```sh
$ sysctl net.inet.carp
  # Shows allow, preempt, log level, and current demotion value
```

- **Runtime dashboards.** Use [systat(1)](https://man.openbsdhandbook.com/systat.1/)
  `pf` view for live counters, and check `/var/log/messages` for CARP role changes.

## Troubleshooting

- **MASTER/MASTER condition.** Two nodes both claim MASTER on a VLAN. Check for VHID collisions, mismatched `pass`, or isolated broadcast domains. Inspect `ifconfig carpX` and correct VHID/password.
- **No state replication.** Verify `pfsync0` is up with the correct `syncdev`, PF is enabled, and `proto pfsync` is permitted only on the SYNC interface. Confirm with `tcpdump -i pfsync0`.
- **Flapping roles.** Upstream instability or asymmetric paths can cause preemption loops. Temporarily raise `advskew` on the unstable node, investigate physical links, and consider disabling preemption until repaired.
- **Failover works, but connections drop.** Ensure rulesets are identical on both nodes and NAT settings match. Confirm states replicate (`pfctl -vvss`) and that hosts on LAN receive gratuitous ARP from CARP during role change.
- **Relayd not serving after failover.** Ensure `relayd` runs on both nodes, listens on the CARP VIP, and that the health checks match the service. Review `/var/log/relayd` and `relayctl show relays`.
- **Demotion stuck high.** Inspect `sysctl net.inet.carp.demotion`. Find and resolve the underlying cause (for example, SYNC down). Reset by lowering `advskew` back to normal and clearing the fault.

## See Also

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- Related: [**Multi-WAN and Policy-Based Routing**](multi-wan-policy-routing.md)
- Related: [**Troubleshooting Playbooks**](troubleshooting-playbooks.md)
