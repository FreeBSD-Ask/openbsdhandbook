# Network Services at Scale

## Synopsis

This chapter describes building reliable, scalable network services on OpenBSD: authoritative DNS with [nsd(8)](https://man.openbsdhandbook.com/nsd.8/)
configured via [nsd.conf(5)](https://man.openbsdhandbook.com/nsd.conf.5/)
, validating recursive DNS with [unbound(8)](https://man.openbsdhandbook.com/unbound.8/)
configured via [unbound.conf(5)](https://man.openbsdhandbook.com/unbound.conf.5/)
, IPv4 address allocation with [dhcpd(8)](https://man.openbsdhandbook.com/dhcpd.8/)
configured via [dhcpd.conf(5)](https://man.openbsdhandbook.com/dhcpd.conf.5/)
, and time services with [ntpd(8)](https://man.openbsdhandbook.com/ntpd.8/)
configured via [ntpd.conf(5)](https://man.openbsdhandbook.com/ntpd.conf.5/)
. Service lifecycle management uses [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
. Packet filtering allowances live in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
and are applied with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
. Use these patterns for branch offices, campus networks, and PoPs where you operate DNS, DHCP, and NTP as first-class, redundant services.

## Design Considerations

- **Separation of roles.** Bind authoritative and recursive DNS to different IP addresses. A common pattern is NSD listening on a public/WAN address (or a dedicated service VIP) and Unbound listening on LAN addresses.
- **Redundancy.** Use at least two servers per critical service. Combine with CARP and pfsync from the HA chapter for failover of service VIPs.
- **Security posture.** Permit UDP/TCP 53 from the Internet **only** to authoritative servers. Restrict recursion to inside prefixes. Enable DNSSEC validation in Unbound.
- **Operational boundaries.** NSD does not support dynamic updates; keep zone management explicit and version-controlled. Unbound is not authoritative; use stub zones to prefer local authoritative answers when both run on the same host.
- **IPv6.** Provide AAAA records in authoritative zones and ensure recursive service answers both A and AAAA. Address assignment for IPv6 prefers RA (see the IPv6 chapter); DHCPv6 is out of scope here.
- **Time service.** DNSSEC validation requires correct time. Ensure `ntpd(8)` is running on resolvers and that clients have a local NTP source.

## Configuration

Assumptions for examples:

- Interfaces: `em0` (WAN), `em1` (LAN `10.10.10.0/24`).
- Service IPs: `203.0.113.53` (authoritative DNS on WAN), `10.10.10.1` (recursive DNS and NTP on LAN).
- Authoritative zone: `example.com.` served by NSD.
- Local recursion for LAN clients via Unbound.

### 1) Authoritative DNS with nsd(8)

Bind NSD to the public address and serve `example.com.` from a local zone file. Control socket is enabled for safe reloads.

```
## /etc/nsd.conf — NSD authoritative service on WAN address

server:
    ip-address: 203.0.113.53
    hide-version: yes
    do-ip4: yes
    do-ip6: no
    server-count: 1
    zonesdir: "/var/nsd/zones"
    # control socket for nsd-control(8)
    control-enable: yes

zone:
    name: "example.com"
    zonefile: "example.com.zone"
```

Create the zone file (minimal example; expand as required). The SOA and NS records must be correct for production.

```
## /var/nsd/zones/example.com.zone — minimal zone

$ORIGIN example.com.
$TTL 300
@   IN SOA ns1.example.com. hostmaster.example.com. (
        2025010101 ; serial (YYYYMMDDNN)
        3600       ; refresh
        600        ; retry
        1209600    ; expire
        300        ; minimum
)
    IN  NS  ns1.example.com.
ns1 IN  A   203.0.113.53
www IN  A   203.0.113.80
api IN  A   203.0.113.81
```

First-time control key setup and service start:

```sh
# nsd-control-setup
  # Generate control keys/certs under /var/nsd/etc; one-time
# rcctl enable nsd
  # Start on boot
# rcctl start nsd
  # Launch now
# nsd-control reload
  # Validate and load the zone
```

### 2) Validating recursive DNS with unbound(8)

Unbound will listen on LAN, serve only inside clients, and validate DNSSEC. When NSD and Unbound share a host, forward a local zone to NSD via a **stub** to avoid conflicts on port 53 by running NSD on WAN and Unbound on LAN.

```
## /var/unbound/etc/unbound.conf — LAN recursion with validation

server:
    interface: 127.0.0.1
    interface: 10.10.10.1
    access-control: 10.10.10.0/24 allow
    access-control: 127.0.0.0/8 allow

    # Hardening and privacy
    hide-identity: yes
    hide-version: yes
    qname-minimisation: yes
    harden-referral-path: yes
    harden-glue: yes
    prefetch: yes

    # DNSSEC
    auto-trust-anchor-file: "/var/unbound/db/root.key"

    # Root priming; Unbound can fetch root NS automatically if needed
    root-hints: "/var/unbound/db/root.hints"

# Prefer local authoritative answers (optional internal zone served by NSD)
stub-zone:
    name: "corp.example."
    stub-addr: 127.0.0.1@5353
```

If you need Unbound to prefer a local zone while NSD keeps port 53 on WAN, bind NSD additionally to `127.0.0.1 port 5353` and serve `corp.example.` there (example below). Otherwise omit the `stub-zone`.

```
## Excerpt — additional NSD listener for an internal zone (loopback)

server:
    # ... existing server block ...
    ip-address: 127.0.0.1@5353

zone:
    name: "corp.example"
    zonefile: "corp.example.zone"
```

Initialize Unbound control and start the resolver:

```sh
# unbound-control-setup
  # Generate control keys/certs under /var/unbound/etc; one-time
# rcctl enable unbound
# rcctl start unbound
# unbound-control status
  # Confirm validator is ready and loaded
```

### 3) DHCP for IPv4 with dhcpd(8)

Serve `10.10.10.0/24`, advertise the local resolver and default gateway, and provide a static mapping example.

```
## /etc/dhcpd.conf — IPv4 DHCP on LAN

option domain-name "corp.example";
option domain-name-servers 10.10.10.1;
option routers 10.10.10.1;
option subnet-mask 255.255.255.0;
default-lease-time 3600;
max-lease-time 7200;
authoritative;

subnet 10.10.10.0 netmask 255.255.255.0 {
    range 10.10.10.100 10.10.10.200;

    # Static reservation example
    host printer-01 {
        hardware ethernet 00:11:22:33:44:55;
        fixed-address 10.10.10.50;
    }
}
```

Enable the daemon and bind it to the LAN interface:

```sh
# rcctl set dhcpd flags em1
  # Specify served interfaces (required)
# rcctl enable dhcpd
# rcctl start dhcpd
# tail -n 20 /var/db/dhcpd.leases
  # Inspect active leases
```

> For IPv6 address assignment, prefer Router Advertisements with `rad(8)` as shown in the IPv6 chapter.

### 4) Time service with ntpd(8)

Provide an internal NTP source on the LAN and maintain correct time on the server itself.

```sh
## /etc/ntpd.conf — serve LAN and sync from upstream pool

listen on 10.10.10.1
servers pool.ntp.org
```

```sh
# rcctl enable ntpd
# rcctl start ntpd
# ntpctl -s status
  # Report synchronization state
```

### 5) PF allowances for DNS, DHCP, and NTP

Permit external queries to NSD on WAN, recursive service on LAN, DHCP on LAN, and NTP for clients.

```pf
## /etc/pf.conf — service allowances (merge into your policy)

set skip on lo

wan = "em0"
lan = "em1"

# Base policy
block all

# Authoritative DNS on WAN (NSD)
pass in on $wan proto { udp tcp } to 203.0.113.53 port 53 keep state

# Recursive DNS on LAN (Unbound)
pass in on $lan proto { udp tcp } to 10.10.10.1 port 53 keep state

# DHCP on LAN (server replies from 67 to client port 68)
pass in on $lan proto udp from port 68 to port 67 keep state
pass out on $lan proto udp from port 67 to port 68 keep state

# NTP on LAN
pass in on $lan proto udp to 10.10.10.1 port 123 keep state

# Resolver and NSD outbound as needed
pass out on $wan proto { udp tcp } to any port { 53, 123 } keep state
```

Reload after editing:

```sh
# pfctl -f /etc/pf.conf
  # Load rules atomically
```

## Verification

- **Authoritative zone served by NSD** (from a remote host or the firewall itself), using drill(1):

```sh
$ drill @203.0.113.53 example.com SOA
  # Expect the SOA you configured
$ drill @203.0.113.53 www.example.com A
  # Expect 203.0.113.80
```

- **Recursive resolution with DNSSEC validation** (from a LAN client):

```sh
$ drill @10.10.10.1 cloudflare.com A
  # General recursion works
$ drill -S @10.10.10.1 dnssec-failed.org
  # Should return a validation failure (bogus), proving DNSSEC is active
```

- **Unbound and NSD control**:

```sh
# unbound-control list_stubs
  # Confirm any stub-zones are loaded
# nsd-control stats
  # Basic qps/counters for authoritative service
```

- **DHCP leases and client config**:

```sh
$ grep dhcp /var/log/messages | tail -n 20
  # Server activity
$ cat /var/db/dhcpd.leases | tail -n 10
  # Lease database updates
```

- **NTP status**:

```sh
$ ntpctl -s peers
  # Upstream peers and offsets
$ ntpctl -s status
  # Overall sync status
```

## Troubleshooting

- **Port conflict on 53.** NSD and Unbound cannot both bind the same IP:port. Bind NSD to the WAN address and Unbound to LAN addresses, or move one to `127.0.0.1@5353` and use Unbound `stub-zone` for the internal domain.
- **Open resolver exposed.** If outside hosts can recurse, restrict Unbound with `access-control` to inside prefixes only and confirm PF denies WAN access to the recursive listener.
- **DNSSEC failures.** Validation requires correct system time and a current trust anchor. Ensure `ntpd(8)` is synchronized and that Unbound has a valid `auto-trust-anchor-file`.
- **Authoritative zone not loading.** Check `nsd-control reload` output and syslog for syntax errors. Validate SOA/NS correctness and serial increments.
- **DHCP not serving.** Confirm `rcctl set dhcpd flags <lan-ifaces>` and that PF allows UDP 67/68 on the LAN. Inspect `/var/db/dhcpd.leases` and logs for pool exhaustion or MAC mismatches.
- **Clients ignore NTP.** Ensure LAN clients point to `10.10.10.1` for NTP and that PF allows UDP 123 from LAN to the server.

## See Also

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- Related: [**High Availability and State Replication**](high-availability.md)
- Related: [**Telemetry, Logging, and Flow Export**](telemetry-logging-flow.md)
- Related: [**IPv6 at Scale**](ipv6-at-scale.md)
