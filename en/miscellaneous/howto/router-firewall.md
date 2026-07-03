# Build a Simple Router and Firewall

## Overview

This chapter describes how to configure OpenBSD as a small router and firewall using two network interfaces: one **WAN** interface that connects to the Internet service provider and one **LAN** interface that connects to the local network. The configuration enables IPv4 forwarding, network address translation (NAT), stateful firewalling, IPv4 address assignment with DHCP, and local DNS caching.

A **router** forwards IP packets between networks. A **firewall** enforces a policy controlling which packets are permitted. **NAT** translates addresses on egress to a public address, allowing multiple local hosts to share one public address. OpenBSD uses **Packet Filter** **pf(4)** for filtering, NAT, and traffic normalization. See [pf(4)](https://man.openbsdhandbook.com/pf.4/)
.

All commands in this chapter require root privileges unless explicitly noted.

This configuration uses two example interfaces, `re0` for the WAN and `re1` for the LAN. Substitute the interface names that exist on the system, as shown by [% ifconfig -A](https://man.openbsdhandbook.com/ifconfig.8/).

## Network Topology and Assumptions

The examples assume:

- WAN interface: `re0`, configured by DHCP from the ISP, or configured with a static address if required by the ISP.
- LAN interface: `re1`, configured with the IPv4 subnet `10.0.0.0/24`. The router uses `10.0.0.1` on the LAN.

Adjust addresses and interface names to match the target environment.

## Enable IP Forwarding

Enable IPv4 and optional IPv6 packet forwarding at runtime with [sysctl(8)](https://man.openbsdhandbook.com/sysctl.8/)
and persist the settings in [sysctl.conf(5)](https://man.openbsdhandbook.com/sysctl.conf.5/)
.

```sh
# sysctl net.inet.ip.forwarding=1
# sysctl net.inet6.ip6.forwarding=1
# printf 'net.inet.ip.forwarding=1\n' >> /etc/sysctl.conf
# printf 'net.inet6.ip6.forwarding=1\n' >> /etc/sysctl.conf
```

IPv6 autoconfiguration on the LAN requires router advertisements. Configure [rad(8)](https://man.openbsdhandbook.com/rad.8/)
or dhcp6d(8) if needed. This chapter focuses on IPv4 services.

## Configure Network Interfaces

Configure the WAN interface. If the ISP assigns addresses by DHCP, write the interface configuration file described in [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
:

```sh
# printf 'inet autoconf\n' > /etc/hostname.re0
```

If the ISP assigns a static address, set address, netmask, and broadcast accordingly:

```sh
# cat > /etc/hostname.re0 <<'EOF'
inet 203.0.113.10 255.255.255.0 203.0.113.255
EOF
```

Configure the LAN interface with a static address:

```sh
# cat > /etc/hostname.re1 <<'EOF'
inet 10.0.0.1 255.255.255.0 10.0.0.255
EOF
```

Apply interface configuration with [netstart(8)](https://man.openbsdhandbook.com/netstart.8/)
:

```sh
# sh /etc/netstart
```

Verify the addresses with [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
:

```sh
$ ifconfig re0
$ ifconfig re1
```

## Configure DHCP Service

The DHCP server [dhcpd(8)](https://man.openbsdhandbook.com/dhcpd.8/)
assigns IPv4 addresses and options to LAN clients. Create `/etc/dhcpd.conf` as described in [dhcpd.conf(5)](https://man.openbsdhandbook.com/dhcpd.conf.5/)
for the `10.0.0.0/24` network. This example sets the default router and DNS server to the router itself and allocates a dynamic pool.

```
subnet 10.0.0.0 netmask 255.255.255.0 {
  option routers 10.0.0.1;
  option domain-name-servers 10.0.0.1;
  range 10.0.0.51 10.0.0.250;
}
```

Optional fixed address assignment by hardware (MAC) address:

```
host server1 {
  fixed-address 10.0.0.30;
  hardware ethernet 00:00:00:00:00:00;
}
```

Enable `dhcpd` at boot, restrict it to the LAN interface, and start it using [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
:

```sh
# rcctl enable dhcpd
# rcctl set dhcpd flags re1
# rcctl start dhcpd
# rcctl check dhcpd
```

Key directives:

- `subnet` and `netmask` define the served IPv4 network.
- `option routers` sets the default gateway for clients.
- `option domain-name-servers` advertises DNS resolvers to clients.
- `range` defines the dynamic address pool available for lease.
- `fixed-address` and `hardware ethernet` bind a static lease to a client MAC address.

## Configure the Firewall and NAT with PF

Use [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
to define the firewall rules and NAT policy for **pf(4)**. Create `/etc/pf.conf` with the following baseline policy. Adjust interface names and networks to match the environment.

```pf
# /etc/pf.conf - minimal router/firewall with NAT for a LAN

# Interfaces and networks
lan_if = "re1"
lan_net = "10.0.0.0/24"

# Non-routable and reserved address space
table <martians> { \
  0.0.0.0/8, 10.0.0.0/8, 127.0.0.0/8, 169.254.0.0/16, \
  172.16.0.0/12, 192.0.0.0/24, 192.0.2.0/24, 198.18.0.0/15, \
  198.51.100.0/24, 203.0.113.0/24, 224.0.0.0/3, 192.168.0.0/16 \
}

# Default behaviors
set block-policy drop
set loginterface egress
set skip on lo

# Normalize traffic
match in all scrub (no-df random-id max-mss 1440)

# NAT for LAN to the current egress address
match out on egress inet from $lan_net to any nat-to (egress:0)

# Anti-spoofing
antispoof quick for { egress, $lan_if }
block in  quick on egress from <martians> to any
block return out quick on egress from any to <martians>

# Default deny
block all

# Allow established and related traffic
pass out on egress inet keep state

# Allow LAN to anywhere
pass in on $lan_if inet from $lan_net to any keep state
```

Validate and load the ruleset, then ensure PF is enabled. Use [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
to check and load rules, and [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
to enable PF at boot:

```sh
# pfctl -nf /etc/pf.conf
# pfctl -f /etc/pf.conf
# rcctl enable pf
# pfctl -e
# pfctl -sr
# pfctl -sn
```

The `egress` keyword matches the interface that carries the default route, as documented in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
. The policy drops unsolicited inbound traffic from the Internet, permits LAN to initiate connections, and performs NAT on outbound IPv4 traffic.

## Configure Local DNS Caching with Unbound

[unbound(8)](https://man.openbsdhandbook.com/unbound.8/)
provides a validating, recursive DNS resolver. Configure it to listen on the router and the LAN address and to forward queries to chosen upstream resolvers. Create `/var/unbound/etc/unbound.conf` as specified in [unbound.conf(5)](https://man.openbsdhandbook.com/unbound.conf.5/)
:

```
server:
  interface: 10.0.0.1
  interface: 127.0.0.1
  interface: ::1

  access-control: 10.0.0.0/24 allow
  access-control: 127.0.0.0/8 allow
  access-control: ::1 allow
  access-control: 0.0.0.0/0 refuse
  access-control: ::0/0 refuse

  hide-identity: yes
  hide-version: yes

forward-zone:
  name: "."
  forward-addr: 64.6.64.6        # Verisign
  forward-addr: 94.75.228.29     # CCC
  forward-first: yes
```

Enable and start the service with [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
:

```sh
# rcctl enable unbound
# rcctl start unbound
# rcctl check unbound
```

Clients on the LAN will receive `10.0.0.1` as their DNS server via DHCP. The router itself can also use the local resolver by pointing [/etc/resolv.conf](https://man.openbsdhandbook.com/resolv.conf.5/)
to `127.0.0.1` if desired.

## Verification

After completing the configuration, connect a client to the LAN and confirm it receives an address in the configured pool, the default router `10.0.0.1`, and the DNS server `10.0.0.1`.

Verify IP connectivity and name resolution from the client. Use [ping(8)](https://man.openbsdhandbook.com/ping.8/)
for a basic reachability test to a public IP and [ftp(1)](https://man.openbsdhandbook.com/ftp.1/)
to fetch a resource by hostname, which exercises DNS:

```sh
$ ping -n 8.8.8.8
$ ftp -o - https://example.com/ >/dev/null
```

On the router, confirm that NAT and states are present with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
:

```sh
# pfctl -ss
# pfctl -s state
```

Refer to the Handbook for a concise `pfctl` cheat sheet at [/pf/cheat\_sheet/](../../networking-and-daemons/pf/cheat_sheet.md)
. For detailed reference, use the Handbook-hosted manual pages: [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
, [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
, [dhcpd.conf(5)](https://man.openbsdhandbook.com/dhcpd.conf.5/)
, [dhcpd(8)](https://man.openbsdhandbook.com/dhcpd.8/)
, [unbound.conf(5)](https://man.openbsdhandbook.com/unbound.conf.5/)
, [unbound(8)](https://man.openbsdhandbook.com/unbound.8/)
, [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
, [sysctl(8)](https://man.openbsdhandbook.com/sysctl.8/)
, and [sysctl.conf(5)](https://man.openbsdhandbook.com/sysctl.conf.5/)
.
