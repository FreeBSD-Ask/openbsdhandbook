# Virtualization and Host Networking

## Synopsis

This chapter presents canonical networking patterns for OpenBSD virtualization with [vmd(8)](https://man.openbsdhandbook.com/vmd.8/)
and [vmctl(8)](https://man.openbsdhandbook.com/vmctl.8/)
: Layer-2 bridging with [bridge(4)](https://man.openbsdhandbook.com/bridge.4/)
, routed segments using [vether(4)](https://man.openbsdhandbook.com/vether.4/)
, and per-VM segmentation with [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
anchors. Virtual NICs use [tap(4)](https://man.openbsdhandbook.com/tap.4/)
. Configuration for the hypervisor lives in [vm.conf(5)](https://man.openbsdhandbook.com/vm.conf.5/)
. Use these patterns to attach guests to existing VLANs, to provide isolated routed networks with NAT, and to enforce least-privilege network policies per VM.

## Design Considerations

- **Attachment model.**
  - **Bridged:** VMs appear on an existing L2 segment. Use when DHCP or broadcast discovery must reach guests.
  - **Routed (vether):** VMs reside behind a virtual L3 interface; the host routes/NATs. Use for isolation and tight control.
- **Failure domains.** Avoid extending large L2 domains unnecessarily. Prefer routed segments with [vether(4)](https://man.openbsdhandbook.com/vether.4/)
  at scale.
- **VLANs.** When bridging into a trunk, add the **parent** interface that carries VLAN tags to the bridge so tags pass transparently; if you need a single VLAN, bridge its [vlan(4)](https://man.openbsdhandbook.com/vlan.4/)
  interface instead.
- **MAC accounting.** Upstream switches may limit learned MACs. Bridging many VMs through one physical port can exceed limits. Coordinate with switching policy or prefer routed attachment.
- **Security posture.** Treat each VM as untrusted. Use PF anchors to apply explicit per-VM policies, and default-deny inter-VM traffic even on the same host.
- **Operations.** Keep VM images and network definitions under version control. Enable verbose logging during initial rollout. Apply a consistent naming scheme (for example, `switch "lan"`, `switch "dmz"`).

## Configuration

The host has `em0` (WAN/uplink) and `em1` (LAN or trunk). The examples show:

1. Bridging VMs into an existing LAN (or trunk).
2. Building an isolated routed network with `vether0` and optional NAT.
3. Enforcing per-VM policy with PF anchors.

### 1) Base hypervisor setup

Create a bridge for guest attachment, define a vmd switch, and enable the hypervisor.

```sh
# cat > /etc/hostname.bridge0 <<'EOF'
add em1
up
EOF
  # Bridge carries em1 toward the LAN/trunk; VMs will join via tap(4)

# sh /etc/netstart bridge0
  # Activate the bridge

# rcctl enable vmd
  # Start vmd(8) at boot
# rcctl start vmd
  # Launch now

# install -d -o root -g wheel /var/vm
  # Ensure a place for VM disks
```

Define a vmd switch and a simple VM in [vm.conf(5)](https://man.openbsdhandbook.com/vm.conf.5/)
. The switch ties VMs to `bridge0`.

```
## /etc/vm.conf — example switch and guest
switch "lan" {
    interface bridge0
}

vm "web01" {
    memory 1024M
    disk "/var/vm/web01.img"
    interface { switch "lan" }
}
```

Create the disk and start the VM (install media omitted here; adjust to your workflow).

```sh
# vmctl create /var/vm/web01.img -s 20G
  # Create a sparse disk
# vmctl start web01
  # Start the VM; vmd(8) attaches a tap(4) to bridge0
# vmctl status
  # List VMs and assigned tap interfaces
```

### 2) Bridged guests on an existing LAN (or VLAN trunk)

With `bridge0` carrying `em1` and the vmd switch targeting `bridge0`, guests appear on the physical LAN (or all VLANs traversing `em1` if it is a trunk). If you must expose a **single** VLAN to guests, bridge its VLAN interface instead:

```
## /etc/hostname.vlan10 — example single-VLAN bridge member
vlandev em1
vlan 10
up
```

```sh
# ifconfig bridge0 delete em1
  # Remove the parent from the bridge when you only want VLAN 10
# ifconfig bridge0 add vlan10
  # Add just the VLAN subinterface; guests see only VLAN 10
# ifconfig bridge0
  # Verify members
```

Provide DHCP or static addressing from your existing infrastructure. To serve DHCP locally to bridged guests, configure [dhcpd(8)](https://man.openbsdhandbook.com/dhcpd.8/)
on `bridge0` or `em1` per your policy (see the Services chapter).

### 3) Routed guests with vether(4) and optional NAT

Create an isolated L3 segment for guests behind `vether0`. The host routes between `vether0` and the uplink.

```
## /etc/hostname.vether0 — routed guest segment
inet 10.50.0.1 255.255.255.0
up
```

Create a dedicated bridge for the routed segment and join guest taps to it dynamically (vmd will add taps when guests start).

```
## /etc/hostname.bridge1 — bridge for the routed segment
add vether0
up
```

Point a second vmd switch at `bridge1` and attach VMs to it.

```
## /etc/vm.conf — excerpt adding a routed switch and VM
switch "isolated" {
    interface bridge1
}

vm "db01" {
    memory 2048M
    disk "/var/vm/db01.img"
    interface { switch "isolated" }
}
```

Enable forwarding and, if desired, NAT to the uplink using [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
. Manage PF at runtime with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
.

```pf
## /etc/pf.conf — minimal routed segment with NAT

set skip on lo

wan      = "em0"
isol_net = "10.50.0.0/24"

# NAT the isolated guests out the uplink
match out on $wan from $isol_net to any nat-to ($wan)

block all
pass in on vether0 from $isol_net to any keep state
pass out on $wan proto { tcp udp icmp } from ($wan) to any keep state
```

```sh
# printf '%s\n' 'net.inet.ip.forwarding=1' >> /etc/sysctl.conf
  # Persist IPv4 forwarding
# sysctl net.inet.ip.forwarding=1
  # Apply immediately
# pfctl -f /etc/pf.conf
  # Load PF policy
```

Serve addresses to routed guests from the host with [dhcpd(8)](https://man.openbsdhandbook.com/dhcpd.8/)
) bound to `vether0` (see the Services chapter), or configure static addresses in `10.50.0.0/24`.

### 4) Micro-segmentation with PF anchors (per-VM policy)

Use PF anchors to isolate guests and to keep per-VM rules small and auditable.

Main policy:

```pf
## /etc/pf.conf — anchor framework (excerpt)

set skip on lo

# Load all per-VM policies from a dedicated directory
anchor "vm/*"
load anchor "vm/*" from "/etc/pf.vm/*.conf"

# Default posture
block all
pass out on egress proto { tcp udp icmp } keep state
```

Per-VM policy files:

```pf
## /etc/pf.vm/web01.conf — allow HTTP/HTTPS from anywhere to web01, SSH from admin subnet
# Attach rules to the VM's tap interface (substitute the actual tapN)
intf = "tap3"

# Inbound public service
pass in on $intf proto tcp to ($intf) port { 80, 443 } keep state

# Operations from admin LAN only
pass in on $intf proto tcp from 10.10.0.0/24 to ($intf) port 22 keep state
```
```sh
## /etc/pf.vm/db01.conf — allow MySQL only from web01; block everything else inbound
intf = "tap4"

# Only the web tier may reach the database
pass in on $intf proto tcp from 10.50.0.20 to ($intf) port 3306 keep state
```

Load and verify:

```sh
# pfctl -f /etc/pf.conf
  # Reload main policy and anchors
# pfctl -a 'vm/*' -sr
  # Show rules within the vm/* anchors
```

> Maintain a mapping of VM names to tap interfaces (from `vmctl status`) and reflect it in anchor files, or template anchors during VM deployment.

## Verification

- **Hypervisor state** with [vmctl(8)](https://man.openbsdhandbook.com/vmctl.8/)
  :

```sh
$ vmctl status
  # VM names, IDs, and tap interfaces
```

- **Bridging and membership** with [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
  :

```sh
$ ifconfig bridge0
  # Expect em1 and one or more tap(4) members for bridged guests
$ ifconfig bridge1
  # Expect vether0 and guest taps for the routed segment
```

- **Routing and NAT**:

```sh
$ netstat -rn
  # Verify a connected route for 10.50.0.0/24 via vether0
$ pfctl -vvsr | egrep 'match out .* nat-to|\banchor vm/'
  # Confirm NAT rule and anchor loads
```

- **On-the-wire checks** with [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
  :

```sh
# tcpdump -ni vether0
  # Guest traffic inside the routed segment
# tcpdump -ni em1 vlan 10
  # VLAN-tagged frames if bridging a single VLAN interface
```

## Troubleshooting

- **Guests do not get DHCP on bridged networks.** Confirm the correct interface is in `bridge0` (`em1` for the trunk or the specific `vlan(4)` for a single VLAN). Ensure the upstream DHCP server can reach the segment and that PF is not blocking BOOTP.
- **No connectivity from routed guests.** Verify `net.inet.ip.forwarding=1`, that `vether0` has the expected address, and that PF loaded the NAT `match` rule. Use `tcpdump` on `$wan` and `vether0` to trace packets.
- **Anchors not applied.** Ensure the `anchor` and `load anchor` lines exist in the main policy and that per-VM files compile. Inspect with `pfctl -a 'vm/*' -sr`.
- **VM attached to the wrong network.** Check the `switch` stanza in `vm.conf` and the `interface { switch "…" }` block in the VM definition. Reload vmd or restart the VM if you changed the attachment.
- **MAC limit exceeded on upstream switch.** Reduce bridged guests on that port or migrate the workload to the routed `vether0` pattern.
- **VLAN tagging not preserved.** When passing multiple VLANs, bridge the **parent** physical interface (for example, `em1`) instead of a single `vlan(4)` child. To expose only one VLAN, bridge just that `vlan(4)` interface.

## See Also

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- Related: [**High Availability and State Replication**](high-availability.md)
- Related: [**Network Services at Scale**](network-services-at-scale.md)
