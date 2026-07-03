# MPLS and Label Distribution

## Synopsis

This chapter describes deploying Multiprotocol Label Switching (MPLS) on OpenBSD with the in-base Label Distribution Protocol daemon [ldpd(8)](https://man.openbsdhandbook.com/ldpd.8/)
. It covers enabling MPLS on interfaces with [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
, core label switching using LDP as signaled by [ldpd.conf(5)](https://man.openbsdhandbook.com/ldpd.conf.5/)
, Layer-3 MPLS VPNs using the Provider Edge interface [mpe(4)](https://man.openbsdhandbook.com/mpe.4/)
with [bgpd.conf(5)](https://man.openbsdhandbook.com/bgpd.conf.5/)
, and Layer-2 VPLS pseudowires with [mpw(4)](https://man.openbsdhandbook.com/mpw.4/)
. Use LDP for simple, interoperable label distribution across an IGP domain; apply BGP-based VPNs where you must carry multi-tenant L3 services across the MPLS core. The MPLS enable/disable and per-interface controls are part of [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
. :contentReference[oaicite:0]{index=0}

## Design Considerations

- **Underlay and discovery.** LDP neighbors form by link discovery on interfaces that carry IP connectivity and have MPLS enabled. Ensure the underlay (static/OSPF/IS-IS/BGP-IGP) provides reachability first; LDP does not replace the IGP. :contentReference[oaicite:1]{index=1}
- **Interface enablement.** MPLS is enabled per interface (for example, `ifconfig em0 mpls`). You can disable it with `-mpls`. Adjust MTU to account for the 4-byte shim per label. :contentReference[oaicite:2]{index=2}
- **Label disposition.** By default, directly connected routes use implicit-null; you may advertise explicit-null via `explicit-null yes` when needed (QoS/ECN preservation). :contentReference[oaicite:3]{index=3}
- **VRFs with rdomains.** Use routing domains (VRF-like) to separate customer routing tables; connect a VRF to the MPLS core with an [mpe(4)](https://man.openbsdhandbook.com/mpe.4/)
  interface bound to that rdomain. :contentReference[oaicite:4]{index=4}
- **L2 services.** For VPLS, build full-mesh pseudowires among PEs and bridge them with [bridge(4)](https://man.openbsdhandbook.com/bridge.4/)
  . Use protected domains to avoid loops. :contentReference[oaicite:5]{index=5}
- **Security and interop.** LDP sessions use TCP and hello discovery; consider GTSM and MD5 where appropriate. OpenBSD’s `ldpd` provides knobs for both. :contentReference[oaicite:6]{index=6}

## Configuration

The examples assume a minimal two-PE core:

- **PE1 underlay:** `em0` ↔ Core, `em1` ↔ Core
- **PE2 underlay:** `em0` ↔ Core, `em1` ↔ Core

IP reachability between PEs over the core exists (via your IGP or statics).

### 1) Enable MPLS on transit interfaces and bring up LDP (both PEs)

Enable MPLS per interface and configure `ldpd` to speak LDP on the core links.

```sh
# ifconfig em0 mpls
  # Allow MPLS on core-facing interface
# ifconfig em1 mpls
  # Allow MPLS on the second core-facing interface

# rcctl enable ldpd
  # Start LDP on boot
# rcctl start ldpd
  # Launch now
```

Create a minimal `/etc/ldpd.conf` that runs LDP for IPv4 on the core interfaces:

```
## /etc/ldpd.conf — minimal LDP over IPv4 on two interfaces

router-id 192.0.2.1         # PE1 example; set uniquely per PE

address-family ipv4 {
    interface em0
    interface em1
    # optional: explicit-null yes
}
```

On the second PE, set `router-id 192.0.2.2` and use the same interface list. `ldpd` discovers neighbors on enabled links; no neighbor IPs are required for link-local sessions. :contentReference[oaicite:7]{index=7}

### 2) Verify label distribution across the core

Use [ldpctl(8)](https://man.openbsdhandbook.com/ldpctl.8/)
and [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
:

```sh
$ ldpctl show neighbors
  # Expect each core link’s neighbor in Operational state
$ ldpctl show lib
  # Label Information Base entries for connected and IGP-learned routes
$ ifconfig em0 | egrep -i 'mpls|mtu'
  # Confirm MPLS is enabled and MTU is appropriate
```

### 3) L3 MPLS VPN (PE-CE) with mpe(4) and OpenBGPD

Bind a customer VRF to MPLS with an [mpe(4)](https://man.openbsdhandbook.com/mpe.4/)
interface inside a non-default rdomain, and signal VPN routes using [bgpd(8)](https://man.openbsdhandbook.com/bgpd.8/)
.

**PE1 (example customer VRF rdomain 10):**

```sh
# ifconfig em2 rdomain 10 inet 10.10.10.1/24
  # CE-facing interface in VRF (customer LAN)
# ifconfig mpe0 create
  # Create Provider Edge interface
# ifconfig mpe0 rdomain 10 mplslabel 2000 inet 192.198.0.1/32 up
  # Attach mpe0 to VRF, assign per-VPN label, add a /32 per bgpd.conf(5) example

# rcctl enable bgpd
# rcctl start bgpd
```

Minimal `/etc/bgpd.conf` excerpt on **PE1** to define the VPN and enable VPN AFI/SAFI toward **PE2**:

```
## /etc/bgpd.conf — L3 VPN on mpe0, IBGP to remote PE

AS 65000
router-id 192.0.2.1

vpn "cust10" on mpe0 {
    rd 65000:10
    import-target rt 65000:10
    export-target rt 65000:10
    network 10.10.10.0/24     # from VRF rdomain 10 via mpe0
}

neighbor 192.0.2.2 {
    remote-as 65000
    descr "PE2 core session"
    announce IPv4 vpn
}
```

On **PE2**, mirror the setup: create a VRF (for example, rdomain 20), attach `mpe0` (label 3000), announce its customer network in the `vpn` block, and enable `announce IPv4 vpn` toward PE1. The `vpn` block uses the `mpe` interface’s label and /32, and can be tied to a different rdomain than the one `bgpd` runs in. :contentReference[oaicite:8]{index=8}

**Verification (either PE):**

```sh
$ bgpctl show summary
  # Session to the remote PE should be Established
$ bgpctl show rib vpn
  # VPN RIB should list customer prefixes with RT/RD and mpe0 as the outgoing iface
$ route -T 10 -n show
  # Customer VRF (rdomain 10) shows remote VPN routes via mpe0
```

### 4) L2 VPLS using mpw(4) pseudowires and bridge(4)

For Layer-2 multipoint service, create pseudowires to remote PEs and bridge them with the CE port. `ldpd` can signal VPLS and bind pseudowires to `mpw(4)`.

**PE1:**

```sh
# ifconfig mpw1 create up
  # Create a pseudowire interface (parameters will be set by ldpd)
# ifconfig bridge0 create
# ifconfig bridge0 add em2
  # CE-facing port into the bridge
# ifconfig bridge0 add mpw1
  # Add the pseudowire to the bridge
```

`/etc/ldpd.conf` VPLS block on **PE1**:

```
## /etc/ldpd.conf — VPLS instance with one pseudowire (PE1)

router-id 192.0.2.1

l2vpn "cust-vpls" type vpls {
    bridge bridge0
    interface em2
    pseudowire mpw1 {
        pw-id 100
        neighbor-id 192.0.2.2    # LSR-ID of PE2
    }
}
```

On **PE2**, create `mpw1`, add it to a local `bridge0`, and use the same `pw-id` with `neighbor-id 192.0.2.1`. If needed, adjust MTU and control-word/flow-aware knobs (`pwecw`, `-pwefat`) on `mpw(4)`. :contentReference[oaicite:9]{index=9}

**Verification (either PE):**

```sh
$ ldpctl show neighbors
  # VPLS-targeted neighbor should appear up
$ ifconfig mpw1
  # Local/remote labels (if set) and operational status
$ ifconfig bridge0
  # Member list (CE port + mpw1) and protected status if configured
```

> **Note:** For transport over GRE underlays, [gre(4)](https://man.openbsdhandbook.com/gre.4/)
> can encapsulate MPLS; plan MTU accordingly. :contentReference[oaicite:10]{index=10}

## Verification

- **Core LDP health:** `ldpctl show neighbors`, `ldpctl show lib`.
- **Interface state:** `ifconfig <if> | grep -i mpls` to confirm MPLS enablement and MTU.
- **L3VPN control plane:** `bgpctl show summary` (Established), `bgpctl show rib vpn` (routes, labels, mpe).
- **VRF dataplane:** `route -T <rdomain> -n show` and traceroute between CE subnets to confirm forwarding through the MPLS core. :contentReference[oaicite:11]{index=11}

## Troubleshooting

- **No LDP neighbors.** Ensure MPLS is enabled on both ends of the link, the underlay IP is up, and the interface is listed in `ldpd.conf`. Check discovery/transport preferences and GTSM if using multi-hop. :contentReference[oaicite:12]{index=12}
- **Labels missing for connected routes.** Review `ldpctl show lib`. Consider `explicit-null yes` for specific paths that require EXP/ECN handling. :contentReference[oaicite:13]{index=13}
- **L3VPN routes not present in VRF.** Confirm `mpe` is in the correct rdomain with a /32 address and `mplslabel`, and that neighbors negotiate the VPN AFI/SAFI (`announce IPv4 vpn`). :contentReference[oaicite:14]{index=14}
- **Pseudowire down.** Verify matching `pw-id`, correct `neighbor-id` (LSR-ID), and consistent MTU. Adjust control word (`pwecw`) or flow-aware transport (`-pwefat`) as needed on `mpw(4)`. :contentReference[oaicite:15]{index=15}
- **MTU black holes.** MPLS adds 4 bytes per label (two labels are common). Lower the payload MTU or raise the underlay MTU where possible; re-test with large pings.

## See Also

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- Related: [**Reference Architectures**](reference-architectures.md)
- Related: [**High Availability and State Replication**](high-availability.md)
