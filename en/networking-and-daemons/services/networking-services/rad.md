# rad

## Synopsis

IPv6 addressing on OpenBSD is typically handled using **Stateless Address Autoconfiguration (SLAAC)** rather than DHCPv6. The `rad(8)` daemon provides **Router Advertisements (RA)** that clients such as [`slaacd(8)`](slaacd.md)
use to self-configure IPv6 addresses, routes, and optional DNS resolver information without requiring a central DHCPv6 server.

Older OpenBSD releases used `rtadvd(8)` for this role. `rtadvd(8)` was replaced by `rad(8)` in OpenBSD 6.4, so new OpenBSD systems should use `rad(8)` and `/etc/rad.conf`.

In contrast with IPv4, where DHCP is commonly used for address configuration, OpenBSD’s base system intentionally omits DHCPv6 server functionality. A lightweight DHCPv6 server is available via ports (`dhcp6s`) but is seldom needed in practice.

## DHCP and Autoconfiguration Components in OpenBSD

The following table summarizes the major DHCP and IPv6 autoconfiguration components available on OpenBSD, covering both IPv4 and IPv6.

| Component | Role | IP Version | In Base | Purpose |
| --- | --- | --- | --- | --- |
| `dhclient(8)` | DHCP client | IPv4 | ✔ | Assigns IPv4 address and DNS from DHCP server |
| `dhcpd(8)` | DHCP server | IPv4 | ✔ | Assigns IPv4 addresses and configuration to clients |
| [`slaacd(8)`](slaacd.md) | SLAAC client daemon | IPv6 | ✔ | Receives RA messages and configures interfaces |
| `rad(8)` | Router advertisement daemon | IPv6 | ✔ | Broadcasts IPv6 prefix, route, and DNS information |
| `dhcp6s` | DHCPv6 server (from ports) | IPv6 | ✘ | Provides stateful IPv6 configuration when required |

For most IPv6 environments on OpenBSD, `rad(8)` and [`slaacd(8)`](slaacd.md)
are sufficient to provide robust autoconfiguration without requiring additional software.

## Configuration

The configuration file for `rad(8)` is `/etc/rad.conf`. A minimal configuration advertises on the selected interface:

```
interface em0
```

To advertise IPv6 prefixes on interface `em0`, enable and start the daemon as follows:

```sh
# rcctl enable rad
# rcctl start rad
```

This instructs `rad(8)` to send Router Advertisements on `em0`, which are then processed by client machines, typically running `slaacd(8)`.

### Customizing Advertisement Behavior

A more explicit `/etc/rad.conf` can set the advertised prefix and DNS information:

```
interface em0 {
    prefix 2001:db8:1::/64
    dns {
        nameserver 2001:db8:1::1
        search example.org
    }
}
```

This announces the LAN prefix and provides Recursive DNS Server (RDNSS) information to clients that support it.

### Example: LAN Router with Static IPv6 Prefix

Given a statically assigned IPv6 prefix (for example, `2001:db8:1::/64`), the router must:

1. Assign an address to its LAN interface:

   ```sh
   # ifconfig em0 inet6 2001:db8:1::1 prefixlen 64 alias
   ```
2. Create `/etc/rad.conf` with an `interface em0` block.
3. Ensure that packet forwarding is enabled:

   ```sh
   # sysctl net.inet6.ip6.forwarding=1
   ```
4. Persist forwarding in `/etc/sysctl.conf`:

   ```conf
   net.inet6.ip6.forwarding=1
   ```
5. Enable and start `rad(8)`:

   ```sh
   # rcctl enable rad
   # rcctl start rad
   ```

## Notes on DHCPv6 and Mixed Environments

OpenBSD’s base system does not include a DHCPv6 server. The `dhcp6s` daemon is available via the ports collection, but it is generally used only when **stateful** IPv6 configuration is required, such as central IP management or lease tracking.

Most OpenBSD systems will operate using:

- `rad(8)` to announce prefixes, routes, and optional DNS resolver information
- [`slaacd(8)`](slaacd.md)
  on clients to auto-configure

This configuration provides full IPv6 connectivity without requiring DHCPv6 and aligns with common IPv6 deployment practice.

## Debugging and Verification

Check that `rad(8)` is enabled and running:

```sh
# rcctl check rad
```

To verify that router advertisements are being sent:

```sh
# tcpdump -n -i em0 icmp6
```

Look for `Router Advertisement` ICMPv6 messages.

To verify that a client interface accepted the RA and configured an address:

```sh
# ifconfig em0
```

Expected output should include an `inet6` autoconfigured address, typically starting with the advertised prefix.

## Additional Considerations

Ensure that the system firewall (`pf(4)`) allows IPv6 traffic, including ICMPv6 Router Advertisements:

Example `/etc/pf.conf` fragment:

```pf
pass inet6 proto ipv6-icmp from :: to ff02::1
```

This permits outgoing multicast advertisements necessary for SLAAC.
