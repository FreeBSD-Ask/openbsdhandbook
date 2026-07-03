# Telemetry, Logging, and Flow Export

## Synopsis

This chapter provides operational observability patterns for OpenBSD firewalls and routers: packet-filter logging via [pflog(4)](https://man.openbsdhandbook.com/pflog.4/)
and [pflogd(8)](https://man.openbsdhandbook.com/pflogd.8/)
, flow export with [pflow(4)](https://man.openbsdhandbook.com/pflow.4/)
to external collectors (NetFlow v5/v9 and, where supported by your release, IPFIX), system and service telemetry with [snmpd(8)](https://man.openbsdhandbook.com/snmpd.8/)
, and central log shipping with [syslogd(8)](https://man.openbsdhandbook.com/syslogd.8/)
configured through [syslog.conf(5)](https://man.openbsdhandbook.com/syslog.conf.5/)
. Verification relies on [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
, [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
, and [systat(1)](https://man.openbsdhandbook.com/systat.1/)
. Use these patterns to produce actionable signals for incident response, capacity planning, and security monitoring.

## Design Considerations

- **Clocks and identity.** Accurate time is mandatory for correlation. Run [ntpd(8)](https://man.openbsdhandbook.com/ntpd.8/)
  and ensure unique hostnames and stable IP sources for telemetry.
- **Scope and privacy.** Only log and export what you can store and protect. Avoid full packet payloads unless explicitly justified. Prefer sampled or flow-level telemetry for aggregate visibility.
- **Separation of concerns.** Treat the firewall/router as a producer. Perform heavy analytics, long-term storage, and dashboards on external systems.
- **Network paths.** Place collectors on dedicated management networks or via isolated VPNs. Protect UDP-based feeds (for example, NetFlow) from loss and spoofing.
- **Version caveats.** [pflow(4)](https://man.openbsdhandbook.com/pflow.4/)
  supports NetFlow v5/v9. IPFIX availability and options are release-dependent. Confirm capabilities on OpenBSD 7.8 before enabling specific templates.
- **Rotation and retention.** Size `/var` and set appropriate rotation in newsyslog.conf(5). Ship logs off-box; do not rely only on local disks.

## Configuration

### 1) Packet filter logging (pflog)

Enable targeted logging in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
. The `log` keyword on a rule emits metadata to `pflog0`, which [pflogd(8)](https://man.openbsdhandbook.com/pflogd.8/)
stores as a rotating pcap at `/var/log/pflog`.

```pf
## /etc/pf.conf — excerpt with rule logging

set skip on lo

wan = "em0"
lan = "em1"
lan_net = "10.10.10.0/24"

block all

# Log blocked unsolicited WAN traffic
block log on $wan

# Log SSH admin access attempts to the firewall
pass  log on $wan proto tcp to ($wan) port 22 keep state

# Allow and log HTTPS from LAN to anywhere (for demo; tighten for production)
pass  log on $lan proto tcp from $lan_net to any port 443 keep state
```

Apply the policy and ensure pflogd is enabled:

```sh
# pfctl -f /etc/pf.conf
  # Atomic reload of PF rules
# rcctl enable pflogd
  # Start pflog capture on boot
# rcctl start pflogd
  # Launch now; writes /var/log/pflog
# tcpdump -n -e -ttt -i pflog0
  # Live view of PF log stream (Ctrl-C to stop)
```

> Use `pfctl -vvsr` to see which rules carry `log` and to inspect per-rule counters.

### 2) Flow export with pflow(4) (NetFlow v5/v9, optionally IPFIX)

[pflow(4)](https://man.openbsdhandbook.com/pflow.4/)
exports summaries of PF state traffic to a remote collector over UDP. Configure a `pflow0` interface with a stable **source address**, collector **destination** (IP:port), and **protocol** version.

Assumptions:

- Export source: `192.0.2.10` (loopback or a management address)
- Collector: `192.0.2.50:2055` (NetFlow)
- Protocol: v9

```sh
# ifconfig pflow0 create
  # Create the exporter interface
# ifconfig pflow0 flowsrc 192.0.2.10 flowdst 192.0.2.50 2055 flowproto 9
  # Set source, destination, and NetFlow version
# ifconfig pflow0 up
  # Activate exporter
# ifconfig pflow0
  # Verify parameters
```

Permit the export path in PF (replace `em0`/addresses as needed):

```sh
## /etc/pf.conf — allow pflow export to collector

set skip on lo
wan = "em0"

pass out on $wan proto udp from 192.0.2.10 to 192.0.2.50 port 2055 keep state
```

Reload and confirm with a packet capture toward the collector:

```sh
# pfctl -f /etc/pf.conf
# tcpdump -ni $wan host 192.0.2.50 and udp port 2055
  # Observe flow export packets; verify at the collector side as well
```

> If your collector expects IPFIX, check [pflow(4)](https://man.openbsdhandbook.com/pflow.4/)
> on OpenBSD 7.8 for IPFIX support and template options, then switch `flowproto` accordingly.

### 3) System telemetry with snmpd(8)

[snmpd(8)](https://man.openbsdhandbook.com/snmpd.8/)
exposes standard MIBs (interfaces, system, sensors). Bind it to a management interface and restrict access. The exact model (community vs. SNMPv3) depends on your security policy; SNMPv3 is recommended where supported by your tooling.

Minimal example: listen on LAN and allow read-only queries from `10.10.10.0/24`.

```
## /etc/snmpd.conf — minimal read-only service

listen on 10.10.10.1

system contact "netops@example.com"
system location "DC1-R1"

read-only community "monitoring" source 10.10.10.0/24
```

```sh
# rcctl enable snmpd
# rcctl start snmpd
# pkill -0 snmpd && echo "snmpd running"
  # Basic liveness check
```

> Consult [snmpd.conf(5)](https://man.openbsdhandbook.com/snmpd.conf.5/)
> for SNMPv3 user configuration and trap receivers. Ensure PF permits UDP 161 from your NMS and any trap egress (UDP 162).

### 4) Central log shipping with syslogd(8)

Forward local logs to a remote collector using [syslog.conf(5)](https://man.openbsdhandbook.com/syslog.conf.5/)
. UDP forwarding uses a single `@` before the host.

```
## /etc/syslog.conf — forward everything to a collector and keep local files

*.*     @192.0.2.60
# Keep existing local file rules below (trim to your needs)
auth.info       /var/log/authlog
daemon.info     /var/log/daemon
kern.info       /var/log/kernlog
pf.info         /var/log/pflogdump
```

```sh
# rcctl reload syslogd
  # Re-read configuration; syslogd sends to 192.0.2.60:514/udp
# logger -t test "syslog forwarding test $(date)"
  # Send a test message; verify it arrives at the collector
```

> If your collector requires TCP or TLS, place a log relay in front of it or consult the collector’s ingestion options. Keep `/etc/newsyslog.conf` tuned for local retention.

## Verification

- **PF rule counters and logs** with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
  and live pflog:

```sh
$ pfctl -vvsr | egrep 'log\b'
  # Confirm which rules log and observe counters increasing
$ tcpdump -n -e -ttt -i pflog0
  # Live packet logs for validation during tests
$ tcpdump -nr /var/log/pflog | head
  # Inspect the rotated capture file
```

- **pflow export on the wire**:

```sh
# tcpdump -ni em0 host 192.0.2.50 and udp port 2055
  # Confirm NetFlow/flow export packets leaving the router
```

- **SNMP reachability** (from NMS or a test host):

```sh
$ snmpwalk -v2c -c monitoring 10.10.10.1 sysUpTime.0
  # Basic liveness; substitute v3 commands if applicable
```

- **syslog forwarding path**:

```sh
$ logger -t probe "forward-test $(date)"
  # Emit a message and confirm it arrives at 192.0.2.60
$ tail -n 5 /var/log/messages
  # Local copy remains available per syslog.conf
```

- **Dashboards**: use [systat(1)](https://man.openbsdhandbook.com/systat.1/)
  `pf` and `ifstat` modes during rollout to correlate queueing, rule hits, and interface rates with exported telemetry.

## Troubleshooting

- **No pflog records.** Ensure rules contain `log`, PF is loaded, and [pflogd(8)](https://man.openbsdhandbook.com/pflogd.8/)
  is running. Use `tcpdump -i pflog0` to bypass files and validate the stream.
- **Collector does not see pflow.** Verify `flowsrc/flowdst/flowproto` on `pflow0`, that PF permits UDP to the collector, and that intermediate ACLs allow it. Confirm time sync; some collectors drop excessively skewed timestamps.
- **High pflow loss.** Flow export is best-effort over UDP. Reduce sampling window (where supported), dedicate a management path, or deploy a closer collector. Ensure CPU is not saturated during bursts.
- **SNMP timeouts.** Confirm `listen on` address and `source` restriction match the NMS. Verify PF allows UDP 161 from the NMS and that community or v3 credentials are correct.
- **Syslog not forwarding.** Check syntax in `/etc/syslog.conf`, reload `syslogd`, and confirm reachability to the collector’s UDP 514. If the collector requires TCP/TLS, use its supported ingestion method.
- **Disk pressure in /var.** Increase rotation frequency in newsyslog.conf(5) and export logs/flows off-box. Consider excluding noisy facilities or raising thresholds.

## See Also

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- Related: [**QoS and Traffic Shaping**](qos-traffic-shaping.md)
- Related: [**Network Services at Scale**](network-services-at-scale.md)
- Related: [**Troubleshooting Playbooks**](troubleshooting-playbooks.md)
