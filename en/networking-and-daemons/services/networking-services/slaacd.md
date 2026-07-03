# slaacd

## Synopsis

OpenBSD provides native support for **Stateless Address Autoconfiguration (SLAAC)**, as described in [RFC 4862](https://datatracker.ietf.org/doc/html/rfc4862)
. SLAAC allows IPv6 hosts to configure their addresses automatically based on Router Advertisements (RA) sent by local routers. This mechanism avoids the need for a centralized DHCPv6 infrastructure and is the default method for client-side IPv6 configuration in OpenBSD.

The `slaacd(8)` daemon is responsible for receiving these advertisements and configuring IPv6 addresses, routes, and resolver settings accordingly.

In typical OpenBSD networks, `slaacd(8)` is used in conjunction with [`rad(8)`](rad.md)
, which provides Router Advertisements on the server or router side.

## DHCP and Autoconfiguration Components in OpenBSD

The table below summarizes the primary components used for DHCP and autoconfiguration on OpenBSD, including both IPv4 and IPv6 support:

| Component | Role | IP Version | In Base | Purpose |
| --- | --- | --- | --- | --- |
| `dhclient(8)` | DHCP client | IPv4 | ✔ | Assigns IPv4 address and DNS from DHCP server |
| `dhcpd(8)` | DHCP server | IPv4 | ✔ | Assigns IPv4 addresses and configuration to clients |
| `slaacd(8)` | SLAAC client daemon | IPv6 | ✔ | Receives RA messages and configures interfaces |
| [`rad(8)`](rad.md) | Router advertisement daemon | IPv6 | ✔ | Broadcasts IPv6 prefix and routing info |
| `dhcp6s` | DHCPv6 server (from ports) | IPv6 | ✘ | Provides stateful IPv6 configuration (rarely needed) |

For most IPv6 networks, `slaacd(8)` and [`rad(8)`](rad.md)
are sufficient and require no third-party software.

## Configuring SLAAC with slaacd(8)

The `slaacd(8)` daemon listens for Router Advertisements and configures:

- IPv6 addresses (using the advertised prefix)
- Default gateways
- DNS resolver settings (if advertised via RDNSS)

### Enabling SLAAC on an Interface

To enable SLAAC on an interface, create or edit the corresponding `/etc/hostname.if` file to contain:

```
inet6 autoconf
```

For example, to enable SLAAC on `em0`:

```sh
# echo "inet6 autoconf" > /etc/hostname.em0
# sh /etc/netstart em0
```

This causes `slaacd(8)` to start automatically and apply the received IPv6 configuration.

### Managing slaacd at Runtime

Although `slaacd` runs automatically for configured interfaces, it can be explicitly managed via `rcctl(8)`:

```sh
# rcctl enable slaacd
# rcctl start slaacd
# rcctl check slaacd
# rcctl restart slaacd
```

To inspect logs:

```sh
# tail -f /var/log/messages
```

## Viewing Assigned IPv6 Addresses

After successful configuration, IPv6 addresses can be inspected with:

```sh
$ ifconfig em0
```

Example output:

```
inet6 fe80::1%em0 prefixlen 64 scopeid 0x1
inet6 2001:db8:abcd:1234::123 prefixlen 64 autoconf
```

The `autoconf` keyword indicates that the address was assigned via SLAAC.

## DNS Configuration via RDNSS

If the router advertises DNS server addresses using **Recursive DNS Server Advertisement (RDNSS)**, `slaacd(8)` will write them to `/etc/resolv.conf`, provided the file is writable.

To allow this behavior, ensure that `/etc/resolv.conf` is not protected with the system immutable flag:

```sh
# chflags noschg /etc/resolv.conf
```

Otherwise, SLAAC will complete but DNS resolution may not function correctly.

## Advertising IPv6 Prefixes

Router Advertisements must be present on the network for SLAAC to succeed. These are typically provided by `rad(8)`, which must run on the default gateway or router.

See the [rad(8)](rad.md)
chapter for full configuration details.

## Disabling SLAAC on an Interface

To disable SLAAC, remove the `inet6 autoconf` directive from the interface’s configuration file and restart the interface:

```sh
# rm /etc/hostname.em0
# echo "inet6 2001:db8::123 255.255.255.0" > /etc/hostname.em0
# sh /etc/netstart em0
```

Alternatively, leave the file empty or omit the `autoconf` keyword to prevent automatic IPv6 configuration.

## Troubleshooting

### Confirm Interface Addressing

```sh
$ ifconfig em0
```

Check for `inet6` addresses with the `autoconf` tag.

### Monitor Router Advertisements

```sh
# tcpdump -i em0 icmp6
```

This will display incoming RA messages.

### Verify DNS Configuration

```sh
$ cat /etc/resolv.conf
$ host openbsd.org
```

If `resolv.conf` does not reflect expected settings, ensure that RDNSS is being advertised and the file is not protected.
