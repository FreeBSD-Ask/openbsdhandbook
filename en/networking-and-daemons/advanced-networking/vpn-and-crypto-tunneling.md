# VPN and Cryptographic Tunneling

## Synopsis

This chapter covers site-to-site virtual private networks (VPNs) and host-to-host cryptographic tunnels on OpenBSD. It focuses on IKEv2 using the base system [iked(8)](https://man.openbsdhandbook.com/iked.8/)
and the kernel IPsec stack [ipsec(4)](https://man.openbsdhandbook.com/ipsec.4/)
, with practical patterns for NAT traversal, selector design, and observability. It also shows a minimal pattern for WireGuard-style tunnels with the in-kernel interface [wg(4)](https://man.openbsdhandbook.com/wg.4/)
. Runtime management uses [ikectl(8)](https://man.openbsdhandbook.com/ikectl.8/)
, packet filtering is handled by [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
and [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
, and link inspection uses [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
. The encrypted interface for IPsec is [enc(4)](https://man.openbsdhandbook.com/enc.4/)
.

Use these patterns to interconnect sites over untrusted networks, to publish internal services across providers securely, or to replace legacy tunnels with modern cryptography.

## Design Considerations

- **Selectors and scope.** Define traffic by networks (for example, `10.0.0.0/24` ↔ `10.20.0.0/24`) rather than `any`. Keep selectors minimal and symmetric on both peers.
- **Authentication.** Pre-shared key (PSK) is simplest for site-to-site. Certificates scale better and are preferred where third-party trust or revocation is needed.
- **NAT traversal.** Permit UDP ports 500 and 4500 and protocol ESP on WAN. IKEv2 NAT-T is automatic when address translation is detected.
- **Routing.** For routed sites, enable IPv4/IPv6 forwarding and add static routes (or dynamic routing inside the tunnel).
- **Performance.** Prefer AES-GCM proposals to offload integrity to the cipher. Keep MTU and MSS in mind where encapsulation crosses constrained links.
- **Operations.** Ensure time sync on all peers (for example, with [ntpd(8)](https://man.openbsdhandbook.com/ntpd.8/)
  ). Log IKE and PF events during rollout.

## Configuration

The examples assume:

- **Site A** WAN: `198.51.100.10`, LAN: `10.0.0.0/24`
- **Site B** WAN: `203.0.113.10`, LAN: `10.20.0.0/24`

Adjust interface names and addresses to match your environment.

### Baseline system settings (both sites)

```sh
# printf '%s\n' \
    'net.inet.ip.forwarding=1' \
    'net.inet6.ip6.forwarding=1' \
    >> /etc/sysctl.conf
  # Enable L3 forwarding for routed traffic through the firewall/router

# sysctl net.inet.ip.forwarding=1 net.inet6.ip6.forwarding=1
  # Apply immediately
```

### PF allowances for IKEv2 and ESP (both sites)

Permit IKE (UDP 500/4500) and ESP on the WAN, and allow traffic between the protected subnets on the inside. Syntax is defined in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
.

```pf
## /etc/pf.conf — minimal allowances for IKEv2/IPsec

set skip on lo

wan     = "em0"           # adjust
lan_net = "{ 10.0.0.0/24, 10.20.0.0/24 }"

block all

# IKE and IPsec on the WAN
pass in on $wan proto udp to ($wan) port { 500, 4500 } keep state
pass in on $wan proto esp keep state
pass out on $wan proto { udp, esp } keep state

# Permit protected subnets after decryption
pass on enc0 from 10.0.0.0/24 to 10.20.0.0/24 keep state
pass on enc0 from 10.20.0.0/24 to 10.0.0.0/24 keep state
```

Reload and confirm with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
.

```sh
# pfctl -f /etc/pf.conf
# pfctl -sr | egrep 'udp .* (500|4500)|proto esp|enc0'
```

### Site-to-site IKEv2 with PSK

Configuration is in [iked.conf(5)](https://man.openbsdhandbook.com/iked.conf.5/)
. One side initiates (`active`), the other listens (`passive`). Proposals below use AES-GCM.

#### Site A (initiator)

```sh
## /etc/iked.conf — Site A

ikev2 "s2s-a2b" active esp from 10.0.0.0/24 to 10.20.0.0/24 \
    peer 203.0.113.10 \
    ikesa enc aes-256-gcm prf sha256 group 14 \
    childsa enc aes-256-gcm \
    psk "change-this-shared-secret"
```

#### Site B (responder)

```sh
## /etc/iked.conf — Site B

ikev2 "s2s-b2a" passive esp from 10.20.0.0/24 to 10.0.0.0/24 \
    ikesa enc aes-256-gcm prf sha256 group 14 \
    childsa enc aes-256-gcm \
    psk "change-this-shared-secret"
```

Enable and start [iked(8)](https://man.openbsdhandbook.com/iked.8/)
on both peers with [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
):

```sh
# rcctl enable iked
  # Start at boot
# rcctl start iked
  # Launch now (initiator will dial the responder)
```

If either peer is behind NAT, IKEv2 will encapsulate ESP in UDP 4500 automatically (NAT-T). Ensure the upstream allows those ports.

### Minimal WireGuard-style tunnel with wg(4)

The [wg(4)](https://man.openbsdhandbook.com/wg.4/)
interface provides a compact, static-key tunnel. Keys are Curve25519; generate them per the wg(4) documentation and place the private key on each host (for example, `/etc/wg/private.key`). Interface attributes are set with [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
.

Example: point-to-point `/30` between hosts with inside addresses `10.99.0.1/30` (Site A) and `10.99.0.2/30` (Site B), UDP port `51820`.

#### Site A

```sh
# ifconfig wg0 create
  # Create the interface
# ifconfig wg0 inet 10.99.0.1/30
  # Assign inside address
# ifconfig wg0 wgport 51820
  # Listen on UDP/51820
# ifconfig wg0 wgkey `cat /etc/wg/private.key`
  # Load this host's private key
# ifconfig wg0 wgpeer <SITE_B_PUBLIC_KEY> wgendpoint 203.0.113.10 51820
  # Configure peer's public key and endpoint
# ifconfig wg0 wgpeer <SITE_B_PUBLIC_KEY> wgaip 10.99.0.2/32
  # Allowed-IPs for the peer (peer's inside address)
# route add -net 10.20.0.0/24 10.99.0.2
  # Route a remote LAN via the tunnel if needed
```

#### Site B

```sh
# ifconfig wg0 create
# ifconfig wg0 inet 10.99.0.2/30
# ifconfig wg0 wgport 51820
# ifconfig wg0 wgkey `cat /etc/wg/private.key`
# ifconfig wg0 wgpeer <SITE_A_PUBLIC_KEY> wgendpoint 198.51.100.10 51820
# ifconfig wg0 wgpeer <SITE_A_PUBLIC_KEY> wgaip 10.99.0.1/32
# route add -net 10.0.0.0/24 10.99.0.1
```

Permit the WireGuard UDP port on each WAN and allow the inside prefixes on `wg0` in PF as appropriate.

## Verification

- **IKEv2 state and flows** with [ikectl(8)](https://man.openbsdhandbook.com/ikectl.8/)
  :

```sh
$ ikectl show sa
  # IKE_SA and CHILD_SA state, SPIs, lifetimes
$ ikectl show flows
  # Traffic selectors currently installed
```

- **Encapsulated traffic** on the wire with [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
  :

```sh
# tcpdump -ni em0 port 500 or port 4500 or proto esp
  # IKE_SA negotiation and ESP/NAT-T on the WAN
# tcpdump -ni enc0 host 10.20.0.50
  # Plaintext (post-decrypt) traffic on enc0 for IPsec
# tcpdump -ni em0 udp port 51820
  # WireGuard UDP transport
```

- **Path testing:**

```sh
$ ping -n -c 3 10.20.0.1
  # From a host behind Site A to a host behind Site B (IKEv2)
$ ping -n -c 3 10.99.0.2
  # Between tunnel endpoints (WireGuard)
```

- **PF and routes**:

```sh
$ pfctl -vvsr | egrep 'udp .* (500|4500)|proto esp|udp .* 51820'
  # Confirm allowances
$ netstat -rn | egrep '10\.99\.0\.|10\.20\.0\.'
  # Confirm inside/tunnel routes
```

## Troubleshooting

- **No IKE negotiation.** Verify UDP 500/4500 reachability and that the responder is in `passive` mode. Inspect logs in `/var/log/daemon` for `iked`.
- **Selector mismatch.** If one side uses `10.0.0.0/24 ↔ 10.20.0.0/24` and the other uses `10.0.0.0/24 ↔ 10.0.0.0/24`, CHILD\_SA will not install. Align `from`/`to` networks. Check `ikectl show flows`.
- **NAT or path asymmetry.** Ensure upstreams carry UDP 4500 unchanged and that return traffic follows the same egress. For IPsec, `enc0` rules should allow the protected subnets.
- **MTU black holes.** If large transfers stall, clamp MSS on the WAN during testing in PF (`scrub in max-mss ...`) and refine after measuring path MTU.
- **Clock skew.** Certificates and IKE lifetimes are time-sensitive. Ensure NTP is working with [ntpd(8)](https://man.openbsdhandbook.com/ntpd.8/)
  .
- **WireGuard handshake fails.** Confirm the correct public keys, matching `wgport`, reachable `wgendpoint`, and that each side has the other’s inside `/32` as an allowed IP. Use `ifconfig wg0` to view current peers and statistics.

## See Also

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- Related: [**Classic and Lightweight Tunnels**](classic-tunnels.md)
- Related: [**Hardening and Operational Safety**](hardening-operations.md)
