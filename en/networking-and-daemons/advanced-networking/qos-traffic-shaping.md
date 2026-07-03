# QoS and Traffic Shaping

## Synopsis

This chapter shows how to classify and shape traffic with PF queues to protect latency-sensitive flows and to manage bulk transfers. Queue configuration and assignment are defined in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
and managed at runtime with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
. Live inspection uses [systat(1)](https://man.openbsdhandbook.com/systat.1/)
and targeted captures with [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
. Use these patterns to provide predictable latency for interactive protocols while keeping link utilization high.

## Design Considerations

- **Shape egress, not ingress.** Shaping controls the traffic you transmit. You cannot directly slow down packets already arriving from the Internet; prefer egress shaping and ACK prioritization to influence remote senders.
- **Accuracy of link rates.** Specify realistic bandwidths for the WAN interface (provider rate minus overhead). If the root queue allows borrowing up to the physical line rate, the default queue may saturate the link.
- **Simple classes win.** Start with three classes: *interactive/voice*, *default*, and *bulk*. Expand only when you have measurements showing contention.
- **Queue assignment strategy.** Use `set queue` on rules (often via `match`) to assign traffic to queues. Consider the two-queue form to prioritize TCP ACKs and `lowdelay` traffic.
- **Priority vs. queues.** `set prio` selects interface priority queues (0–7) and also maps to VLAN PCP on [vlan(4)](https://man.openbsdhandbook.com/vlan.4/)
  . It does not shape by itself. Use it sparingly in addition to queueing when you need L2 markings.
- **Verification first.** Enable counters, test with reproducible flows, and watch queue utilization under load. Keep a roll-back of the previous `pf.conf`.

## Configuration

The example below shapes outbound traffic on a single uplink. It prioritizes SSH interactivity and VoIP, keeps web browsing responsive, and deprioritizes bulk transfers. Adjust interface names, addresses, and rates to match your environment.

### Assumptions

- `em0` — WAN (100 Mb/s contracted)
- `em1` — LAN (`10.10.10.0/24`)

### Queues and Classification (single-file policy)

Place this in `/etc/pf.conf`. The *QUEUEING* and `set queue` syntax are defined in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
.

```sh
## /etc/pf.conf — QoS with PF queues (example)

set skip on lo

# Interfaces
wan     = "em0"
lan     = "em1"
lan_net = "10.10.10.0/24"

# -------------------------
# Queue tree on the WAN
# -------------------------
# Root queue at or slightly below line rate; avoid over-commit until measured.
queue rootq on $wan bandwidth 95M max 100M

# Realistic classes. Borrowing is implicit unless max bounds them.
queue voice       parent rootq bandwidth 2M    min 1M
queue web         parent rootq bandwidth 40M
  queue web_bulk  parent web   bandwidth 35M
  queue web_prio  parent web   bandwidth 5M    min 2M
queue ssh         parent rootq bandwidth 5M
  queue ssh_bulk  parent ssh   bandwidth 3M
  queue ssh_prio  parent ssh   bandwidth 2M    min 1M
queue default     parent rootq bandwidth 48M   default
queue bulk        parent rootq bandwidth 10M   max 20M

# -------------------------
# Base policy (tighten to your model)
# -------------------------
block all

# Permit LAN to anywhere; classification follows in match rules below
pass in on $lan from $lan_net to any keep state

# Permit outbound from the firewall itself
pass out on $wan proto { tcp udp icmp } from ($wan) to any keep state

# -------------------------
# Classification (sticky via match rules)
# -------------------------

# VoIP (SIP + RTP range example)
match out on $wan proto udp from $lan_net to any port { 5060, 10000:20000 } \
    set queue voice

# SSH: favor interactivity by assigning TCP ACKs and lowdelay to the second queue
match out on $wan proto tcp from $lan_net to any port 22 \
    set queue (ssh_bulk, ssh_prio)

# HTTPS: keep browsing responsive by prioritizing ACKs/lowdelay
match out on $wan proto tcp from $lan_net to any port 443 \
    set queue (web_bulk, web_prio)

# Bulk examples: backup, torrents, rsync (adjust as needed)
match out on $wan proto { tcp udp } from $lan_net to any port { 873, 6881:6999 } \
    set queue bulk

# Default: everything else lands in the default queue
# (No explicit rule needed because 'default' is marked on the queue tree)
```

Load and confirm:

```sh
# pfctl -f /etc/pf.conf
  # Atomic reload
# pfctl -vvsq
  # Inspect the queue tree, rates, and counters
```

### Optional: VLAN PCP marking with `set prio`

If your upstream honors 802.1Q PCP, assign a higher interface priority for voice while retaining queueing control. This marks L2 frames and may aid upstream scheduling on enterprise links.

```pf
## Excerpt — add to classification if using VLAN PCP
match out on $wan proto udp from $lan_net to any port { 5060, 10000:20000 } \
    set prio 6 set queue voice
```

### Optional: Shaping replies from a published service

When you publish a LAN service to the Internet, classify **egress** replies from the server so that returning traffic honors your QoS.

```pf
## Excerpt — shape egress from a LAN web server
web_svc = "10.10.10.50"

# If you also use rdr on the WAN elsewhere, keep that configuration unchanged.
# Prioritize ACKs for HTTPS responses leaving via the WAN.
match out on $wan proto tcp from $web_svc to any port 443 \
    set queue (web_bulk, web_prio)
```

## Verification

- **Queue usage and limits** with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
  :

```sh
$ pfctl -vvsq
  # Verify the tree, min/max/burst, current/peak rates, and packet counters
$ pfctl -vvsr | egrep 'set queue|set prio'
  # Confirm rule-to-queue mappings are loaded as intended
```

- **Live view** with [systat(1)](https://man.openbsdhandbook.com/systat.1/)
  :

```sh
$ systat pf
  # Observe states and rule counters under load; keep an eye on the queues' counters via pfctl
```

- **On-the-wire sanity** with [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
  :

```sh
# tcpdump -ni em0 port 22 or port 443 or udp and portrange 10000-20000
  # Confirm flows exist during your test and correlate with queue counters
```

- **Load generation.** Use controlled transfers (for example, `scp`/`rsync` for bulk and an SSH session for interactivity) to demonstrate that keystrokes remain responsive while bulk traffic is active.

## Troubleshooting

- **No effect on latency or throughput.** Ensure you created a queue tree on the **egress** interface actually carrying the traffic and that `set queue` rules match in the correct direction (often `match out on $wan`). Verify with `pfctl -vvsr` and `pfctl -vvsq`.
- **“queue does not exist.”** A `set queue` refers to a leaf queue name not present on the outgoing interface. Correct the queue name or ensure the queue is a **leaf** under the proper parent.
- **Default queue saturates.** If latency-sensitive traffic still suffers, reduce `bandwidth` or `max` on `default`, or raise `min` for priority queues. Validate with repeatable tests.
- **Unintended borrowing.** If a child queue grows beyond expectations, cap it with `max` or lower the parent’s bandwidth.
- **ACK starvation.** Without the two-queue form, TCP ACKs may sit behind large packets. Use `set queue (bulk, prio)` for classes that benefit from responsive ACKs.
- **Layer-2 markings ignored.** `set prio` maps to VLAN PCP, but upstream equipment may ignore it. Treat it as a hint, not enforcement.
- **Shaping the wrong traffic.** Use `tcpdump` to validate selectors. A common error is matching on `in on $wan` instead of `out on $wan` for client egress.

## See Also

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- Related: [**IPv6 at Scale**](ipv6-at-scale.md)
- Related: [**Telemetry, Logging, and Flow Export**](telemetry-logging-flow.md)
