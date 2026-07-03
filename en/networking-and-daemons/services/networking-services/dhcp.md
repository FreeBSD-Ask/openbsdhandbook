# DHCP

## Synopsis

Dynamic Host Configuration Protocol (DHCP) allows systems on a network to automatically obtain IPv4 addresses and associated configuration information such as DNS resolvers and default gateways. OpenBSD provides native support for both DHCP clients and servers for IPv4. For IPv6 address configuration, OpenBSD uses SLAAC (Stateless Address Autoconfiguration) with `slaacd(8)` rather than DHCPv6.

This chapter describes how to configure both the DHCP client and server for IPv4 in OpenBSD, including interface configuration, lease management, and optional settings. For IPv6 autoconfiguration and router advertisement, refer to the [SLAAC](slaacd.md)
chapter.

## Overview of IP Configuration Tools

The following table provides a comparison of tools available in the OpenBSD base system and ports for managing IP address configuration for both IPv4 and IPv6:

| Utility | Description | IPv4 | IPv6 |
| --- | --- | --- | --- |
| `dhclient(8)` | DHCP client | ✔ | ✖ (use `slaacd`) |
| `dhcpd(8)` | DHCP server | ✔ | ✖ |
| `dhclient.conf(5)` | DHCP client configuration file | ✔ | ✖ |
| `slaacd(8)` | SLAAC client | ✖ | ✔ |
| `rad(8)` | Router Advertisement daemon | ✖ | ✔ |
| `dhcp6s` (ports) | DHCPv6 server (via ports) | ✖ | ✔ |
| `dhcp6c` (ports) | DHCPv6 client (via ports) | ✖ | ✔ |

For IPv6 support, OpenBSD prefers SLAAC over DHCPv6. The base system includes all necessary tools for SLAAC but does not include DHCPv6. See the [SLAAC and IPv6 Autoconfiguration](slaacd.md)
chapter for detailed guidance.

## Configuring the DHCP Client

The DHCP client is automatically invoked by the system at boot if a network interface is configured with the `dhcp` keyword.

### Interface Setup

To configure a network interface to use DHCP, create an interface configuration file `/etc/hostname.if` with the word `dhcp`. For example, to enable DHCP on interface `em0`:

```sh
# echo "dhcp" > /etc/hostname.em0
# sh /etc/netstart em0
```

This causes `dhclient(8)` to be executed at boot or when the interface is brought up.

### Lease File Management

Leases obtained via DHCP are stored in `/var/db/dhclient.leases.ifname`, where `ifname` is the interface name.

To view the current lease information:

```sh
$ cat /var/db/dhclient.leases.em0
```

To manually release the lease:

```sh
# dhclient -r em0
```

To force `dhclient` to request a new lease:

```sh
# dhclient em0
```

### Hostname Configuration

By default, `dhclient` sends the system hostname to the DHCP server. To disable this behavior, add the following to `/etc/dhclient.conf`:

```
send host-name "";
```

## DHCP Client Customization

Custom DHCP options can be specified in `/etc/dhclient.conf`. This file allows per-interface configuration and override of values received from the DHCP server.

### Example Configuration

To override the DNS servers and domain name:

```
interface "em0" {
    supersede domain-name "example.net";
    supersede domain-name-servers 192.0.2.53, 192.0.2.54;
}
```

Changes will take effect the next time `dhclient` is run on the interface.

## Configuring the DHCP Server

OpenBSD includes the ISC DHCP server as `dhcpd(8)`, which provides dynamic address assignment for clients on a local network.

### Example dhcpd.conf

A minimal DHCP server configuration file might resemble the following:

```
option domain-name "example.net";
option domain-name-servers 192.0.2.53;

subnet 192.0.2.0 netmask 255.255.255.0 {
    range 192.0.2.100 192.0.2.200;
    option routers 192.0.2.1;
}
```

Save this configuration as `/etc/dhcpd.conf`.

### Starting and Enabling the DHCP Server

To enable and start the DHCP server on a specific interface (e.g., `em1`):

```sh
# rcctl enable dhcpd
# rcctl set dhcpd flags em1
# rcctl start dhcpd
```

The interface must be explicitly specified via `rcctl set dhcpd flags ...` for the daemon to operate.

## Logging and Troubleshooting

DHCP client and server logs are written to `/var/log/messages`. To observe real-time DHCP activity:

```sh
# tail -f /var/log/messages
```

Ensure no other DHCP server is running on the same network segment. Packet filtering rules must also allow DHCP traffic (UDP ports 67 and 68).

For server-side diagnostics, start `dhcpd` in the foreground with verbose output:

```sh
# dhcpd -d -f em1
```
