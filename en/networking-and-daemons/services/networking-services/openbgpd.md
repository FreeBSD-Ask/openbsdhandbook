# OpenBGPD

## Synopsis

This chapter provides canonical, production-focused patterns for running BGP on OpenBSD using the base system daemon [bgpd(8)](https://man.openbsdhandbook.com/bgpd.8/)
. It covers configuration in [bgpd.conf(5)](https://man.openbsdhandbook.com/bgpd.conf.5/)
, runtime operations with [bgpctl(8)](https://man.openbsdhandbook.com/bgpctl.8/)
, service control via [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
, and optional origin validation with [rpki-client(8)](https://man.openbsdhandbook.com/rpki-client.8/)
. These patterns apply to external peering, multi-provider edges, iBGP within a site or POP, and policy expression with communities and local preference.

## Design Considerations

- **Policy-first.** Default to “deny” and explicitly allow intended imports and announcements. Keep filters close to neighbors and keep global defaults conservative.
- **Deterministic preference.** Set `localpref` for inbound path selection inside the AS; use communities for structured policy signaling.
- **Scale boundaries.** In small fabrics, a full-mesh iBGP suffices. For larger topologies, introduce route reflectors and keep reflector state minimal.
- **RPKI.** Validate origins for received routes; drop `roa invalid`. Schedule [rpki-client(8)](https://man.openbsdhandbook.com/rpki-client.8/)
  to refresh ROAs.
- **Observability.** Treat `bgpctl show summary`, `show neighbor`, and `show rib {in,out}` as part of every change and incident procedure.
- **Change control.** Stage edits, syntax-check, and keep a roll-back path (console or timed revert) when touching upstream sessions.

## Configuration

### 1) Service lifecycle and skeleton

Create a minimal skeleton with the local AS and router ID. Enable the daemon.

```
## /etc/bgpd.conf — minimal skeleton
AS 64512
router-id 198.51.100.2
```

```sh
# rcctl enable bgpd
  # Start on boot
# bgpd -n -f /etc/bgpd.conf
  # Syntax check (no start)
# rcctl start bgpd
  # Launch the daemon
```

### 2) Single-homed eBGP to an upstream (announce specific prefixes)

Assumptions:

- Public prefixes: `198.51.100.0/24`, `2001:db8:100::/48`
- Upstream neighbor: `203.0.113.1`, neighbor AS `64500`
- eBGP source address: `198.51.100.2`

```
## /etc/bgpd.conf — eBGP to one ISP with explicit outbound and guarded inbound
AS 64512
router-id 198.51.100.2

# Prefixes intended for announcement
prefix-set "our-net" {
    198.51.100.0/24,
    2001:db8:100::/48
}

neighbor 203.0.113.1 {
    remote-as 64500
    descr "ISP-A"
    local-address 198.51.100.2
    enforce neighbor-as yes
    max-prefix 500000 restart        # Tune per provider guidance
    announce none                     # Outbound controlled by match rules below
}

# Inbound: accept from the upstream (subject to max-prefix and future filters)
allow from neighbor 203.0.113.1

# Outbound: announce only approved prefixes to this neighbor
match to neighbor 203.0.113.1 prefix-set "our-net" announce
```

Reload and verify:

```sh
# bgpctl reload
  # Graceful reconfiguration
$ bgpctl show summary
  # Session state, prefix counts, per-neighbor stats
$ bgpctl show neighbor 203.0.113.1
  # Capabilities and last error (if any)
$ bgpctl show rib out neighbor 203.0.113.1
  # Only approved prefixes should be advertised
```

### 3) Dual-homed eBGP (prefer one upstream with localpref; communities on egress)

Assumptions:

- Second upstream: `198.51.100.1`, AS `64496`
- Preference: routes from `ISP-A` should win over `ISP-B` by higher `localpref`
- Egress tagging: set a reference community on announced routes (example only)

```
## /etc/bgpd.conf — two providers; deterministic preference and egress tagging
AS 64512
router-id 198.51.100.2

prefix-set "our-net" { 198.51.100.0/24, 2001:db8:100::/48 }

neighbor 203.0.113.1 {
    remote-as 64500
    descr "ISP-A"
    local-address 198.51.100.2
    enforce neighbor-as yes
    max-prefix 500000 restart
    announce none
}

neighbor 198.51.100.1 {
    remote-as 64496
    descr "ISP-B"
    local-address 198.51.100.2
    enforce neighbor-as yes
    max-prefix 500000 restart
    announce none
}

# Inbound acceptance from both
allow from neighbor 203.0.113.1
allow from neighbor 198.51.100.1

# Prefer ISP-A inside the AS
match from neighbor 203.0.113.1 set localpref 200
match from neighbor 198.51.100.1 set localpref 150

# Outbound: announce the same set to both, with a reference community
match to neighbor 203.0.113.1 prefix-set "our-net" set community 64512:100 announce
match to neighbor 198.51.100.1 prefix-set "our-net" set community 64512:100 announce
```

Verification focus:

```sh
$ bgpctl show rib as 64500
  # Routes learned from ISP-A (AS path view)
$ bgpctl show rib | egrep '198\.51\.100\.0/24|2001:db8:100::/48'
  # Originated prefixes present in the RIB before export
$ bgpctl show neighbor | egrep 'LocalPref|Community'
  # Policy markers visible as expected
```

> Announcements require that the route exist in the routing table (for example, via static routes or loopback assignments). Install deterministic originating routes (host /32 or /128 for service IPs, aggregates for blocks) before `bgpd` can export them.

### 4) iBGP within the AS (small fabric, full mesh)

Assumptions:

- Two routers, `R1` (this host) and `R2` at `10.0.0.2`, both in AS `64512`
- Full-mesh iBGP; R1 preferred as exit to `ISP-A` via higher `localpref` applied to routes learned from that ISP

```
## /etc/bgpd.conf — add a simple iBGP peer
AS 64512
router-id 10.0.0.1

neighbor 10.0.0.2 {
    remote-as 64512
    descr "iBGP-R2"
    announce none
}

# iBGP acceptance
allow from neighbor 10.0.0.2

# Optional: restrict exchange to internal routes
prefix-set "internal" { 10.10.0.0/16, 10.20.0.0/16 }
match to neighbor 10.0.0.2 prefix-set "internal" announce
```

Verification focus:

```sh
$ bgpctl show summary
  # iBGP session should be Established
$ bgpctl show rib neighbor 10.0.0.2
  # Routes accepted from the peer
$ bgpctl show rib out neighbor 10.0.0.2
  # Routes advertised to the peer (filtered by "internal")
```

> For larger topologies, introduce route reflectors to avoid a full mesh. Keep reflectors simple and limit re-advertisement to required address families and scopes.

### 5) RPKI origin validation (using rpki-client)

Run [rpki-client(8)](https://man.openbsdhandbook.com/rpki-client.8/)
periodically to fetch and validate ROAs, then have `bgpd` load the generated set. Reject `roa invalid` by policy.

```sh
# rcctl enable rpki_client
# rcctl start rpki_client
  # Starts periodic ROA fetch/validation; artifacts under /var/db/rpki-client/
# rpki-client -vv
  # On-demand run for initial population and diagnostics
```

bgpd configuration:

```pf
## /etc/bgpd.conf — load ROAs and drop invalids
AS 64512
router-id 198.51.100.2

roa-set {
    /var/db/rpki-client/openbgpd
}

# Policy: do not accept routes with invalid origin per ROA
deny from any roa invalid

# Allow remaining imports per neighbor rules
```

Observe RPKI state:

```sh
$ bgpctl show rib roa invalid
  # Routes rejected due to invalid origin
$ bgpctl show rib roa unknown | head
  # Routes without covering ROAs (informational)
```

### 6) Anycast service advertisement inside the AS

For an internal anycast service (for example, resolver `10.10.255.53/32`), originate the address on each serving node (lo0 alias or static route), and announce using iBGP. ECMP or policy then steers clients.

```
## /etc/bgpd.conf — originate a host route for anycast via iBGP
AS 64512
router-id 10.0.0.1

network 10.10.255.53/32
  # Route must exist in the RIB (lo0 alias or static) before export
```

## Verification

Core commands with [bgpctl(8)](https://man.openbsdhandbook.com/bgpctl.8/)
:

```sh
$ bgpctl show summary
  # Session state and counters
$ bgpctl show neighbor <addr>
  # Capabilities, timers, last error
$ bgpctl show rib
  # Loc-RIB; append 'in' or 'out' with 'neighbor <addr>' for per-peer views
$ bgpctl show rib community 64512:100
  # Filter by community; useful for policy checks
$ bgpctl log verbose
  # Increase bgpd(8) log verbosity temporarily during troubleshooting
```

Wire-level and system checks:

```sh
# tcpdump -ni egress port 179
  # Observe BGP OPEN/KEEPALIVE/UPDATE on the wire
$ route -n show -inet ; route -n show -inet6
  # Originating/static routes present for announced prefixes
```

## Troubleshooting

- **Session stuck Idle/Active.** Confirm IP reachability, `local-address`, and that PF allows TCP/179. Validate that `remote-as` matches the peer’s configured local AS and that `enforce neighbor-as` is not blocking expected paths.
- **No announcements leaving.** The route must exist in the RIB (`network` statement or static route). Confirm that an explicit `match ... announce` rule targets the neighbor. Inspect `bgpctl show rib out neighbor ...`.
- **Excessive routes from a peer.** Raise or add `max-prefix` with `restart` and a `warning` threshold. Constrain scope with prefix-lists or `prefix-set` filters.
- **Inbound path not preferred.** Review `localpref` on received routes. Check whether more-specifics or MEDs override intent.
- **RPKI drops unexpected routes.** Inspect `bgpctl show rib roa invalid` and cross-check current ROAs on the parent ROA repository. Investigate ROA state before relaxing policy.
- **Flapping after reloads.** Use `bgpctl reload` for graceful policy updates. For structural changes (peer add/remove), apply during maintenance windows or off-peak hours.

## See Also

- [Networking](../../../install-and-configure/networking.md)
- [Advanced Networking](../../advanced-networking/README.md)
- Man pages: [bgpd(8)](https://man.openbsdhandbook.com/bgpd.8/)
  , [bgpd.conf(5)](https://man.openbsdhandbook.com/bgpd.conf.5/)
  , [bgpctl(8)](https://man.openbsdhandbook.com/bgpctl.8/)
  , [rpki-client(8)](https://man.openbsdhandbook.com/rpki-client.8/)
- Related: [MPLS and Label Distribution](../../advanced-networking/mpls.md)
- Related: [Reference Architectures](../../advanced-networking/reference-architectures.md)
