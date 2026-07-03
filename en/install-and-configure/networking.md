# Networking

## Synopsis

OpenBSD provides a flexible and secure framework for configuring and managing network interfaces. This chapter explains how to configure both wired and wireless networking, IPv4 and IPv6 addressing, DNS resolution, routing, network bridges and trunks, and includes diagnostic tools and secure practices. All configuration is done through OpenBSD-native files and utilities such as `ifconfig(8)`, `hostname.if(5)`, `dhclient(8)`, and `pf(4)`.

## Network Interface Basics

Each network interface is named according to the driver and unit number, such as `em0` for the first Intel gigabit Ethernet interface or `iwn0` for an Intel wireless device. To display all interfaces, including inactive ones:

```sh
# ifconfig -a
```

To check the status of a specific interface:

```sh
# ifconfig em0
```

To list available network interfaces based on detected hardware, examine the system boot messages:

```sh
# dmesg | grep -E '^[a-z]+[0-9]+.*(Ethernet|wireless|network)'
```

Firmware for some wireless devices is not included in the base system. Use `fw_update(8)` to install required firmware:

```
# fw_update
```

## Configuring Wired Interfaces

Network interface configuration is persistent across reboots via files named `/etc/hostname.if`, where `if` corresponds to the interface name (e.g., `em0`).

To use DHCP:

```sh
# echo dhcp > /etc/hostname.em0
```

To configure a static IP address:

```sh
# cat /etc/hostname.em0
inet 192.168.1.100 255.255.255.0 192.168.1.1
```

The gateway address must be reachable within the configured subnet.

To bring up the interface manually:

```sh
# ifconfig em0 up
```

To apply configuration changes:

```
# sh /etc/netstart em0
```

Root privileges are required for these commands.

### Configuring Interface Aliases

An **interface alias** allows multiple IP addresses to be assigned to a single network interface. This is useful in many scenarios, including:

- Running multiple services bound to different IPs
- Operating virtual hosts
- Separating traffic for organizational or policy purposes

Aliases can be assigned to the same subnet as the primary address, or to entirely different subnets. When the alias resides in the **same subnet**, the netmask must be set to `255.255.255.255` to avoid unintended routing behavior. If the alias belongs to a **different subnet**, the correct netmask for that subnet must be specified, and in many cases, an explicit route will be needed to make the address reachable.

Interface aliases are configured in the interface’s startup file, located in `/etc/hostname.if`, where *if* is the interface name (such as `em0`, `re0`, or `iwn0`). These configurations are applied automatically at boot or manually using `sh /etc/netstart`.

#### Example: Multiple Aliases in the Same Subnet

In the following example, the system is assigned a primary IPv4 address of `10.0.0.2/24`, along with two aliases in the same subnet:

File: `/etc/hostname.re0`

```
inet 10.0.0.2 255.255.255.0
inet alias 10.0.0.3 255.255.255.255
inet alias 10.0.0.4 255.255.255.255
```

To activate the changes immediately:

```sh
# sh /etc/netstart re0
```

#### Example: Alias in a Different Subnet

If an alias must be added from a different network range, the netmask must match that network. For example, to add an alias in the `192.168.100.0/24` subnet:

```
inet 10.0.0.2 255.255.255.0
inet alias 192.168.100.10 255.255.255.0
```

In this case, OpenBSD does not automatically add a route for the new subnet. If required, a static route must be defined in `/etc/mygate` or set using `route(8)`.

#### Viewing Aliases

By default, `ifconfig(8)` shows only the primary address of each interface. To view all configured addresses, including aliases, use the `-A` option:

```sh
# ifconfig -A
```

This command will display all IPv4 and IPv6 addresses associated with each interface, including any aliases.

## Wireless Networking

To connect to an open network:

```sh
# echo "nwid mynetwork" > /etc/hostname.iwn0
# sh /etc/netstart iwn0
```

To connect to a WPA2-secured network:

```sh
# echo 'nwid mynetwork wpakey "mypassword"' > /etc/hostname.iwn0
```

Quote the `wpakey` if it contains special characters.

To connect to multiple access points or configure other parameters:

```
join mynetwork wpakey mypassword
dhcp
```

To scan for available networks:

```sh
# ifconfig iwn0 up
# ifconfig iwn0 scan
```

WPA3 support is limited to drivers such as `iwm(4)` and `athn(4)` on compatible hardware. Refer to the relevant driver manual page for details.

## Hostname and DNS Resolution

Set the system hostname in `/etc/myname`:

```
myhostname.example.net
```

Configure DNS servers in `/etc/resolv.conf`:

```
search example.net
nameserver 9.9.9.9
nameserver 2620:fe::fe
```

Ensure IPv6 connectivity when specifying IPv6 nameservers.

To resolve hostnames locally:

```sh
# cat /etc/hosts
127.0.0.1    localhost
192.168.1.1  gw.example.net gw
```

By default, `dhclient(8)` overwrites `/etc/resolv.conf`. To prevent this:

```
# chflags schg /etc/resolv.conf
```

**Warning:** Using `schg` will block dynamic updates, which is unsuitable in mobile or dynamic environments. Alternatively, customize `dhclient.conf(5)`.

## Routing and Default Gateway

To view the routing table:

```
# netstat -rn
# netstat -rn -f inet6
```

To add a default route:

```
# route add default 192.168.1.1
```

To remove it:

```
# route delete default
```

To persist the default gateway for a static configuration, include it in `/etc/hostname.if`. For more complex routing:

```
!/sbin/route add -inet6 default 2001:db8::1
```

The `!` prefix executes the command after the interface is configured.

## IPv6 Configuration

To use automatic IPv6 addressing via SLAAC:

```sh
# echo 'inet6 autoconf' > /etc/hostname.em0
```

To enable privacy extensions (temporary addresses):

```
inet6 autoconfprivacy
```

To disable SLAAC:

```
inet6 -autoconf
```

Static IPv6 configuration:

```
inet6 2001:db8::100 64
!/sbin/route add -inet6 default 2001:db8::1
```

Test IPv6 connectivity:

```
# ping6 -c 3 openbsd.org
# traceroute6 openbsd.org
```

To discover local hosts via link-local addresses:

```
# ping6 ff02::1%iwn0
```

## Bridging and Interface Aggregation

To create a bridge:

```sh
# cat /etc/hostname.bridge0
add em0
add em1
up
```

Configure member interfaces:

```sh
# cat /etc/hostname.em0
up media 1000baseT mediaopt full-duplex

# cat /etc/hostname.em1
up media 1000baseT mediaopt full-duplex
```

To create a link aggregation (LACP) trunk:

```sh
# cat /etc/hostname.trunk0
trunkproto lacp trunkport em0 trunkport em1
inet 192.168.1.2 255.255.255.0 192.168.1.1
```

**Note:** Switches must support LACP or spanning tree protocol. To check supported media types:

```sh
# ifconfig em0 media
```

## Network Diagnostics

To test basic connectivity:

```
# ping 1.1.1.1
# ping openbsd.org
```

To test DNS:

```
# host openbsd.org
```

To view ARP table:

```
# arp -a
```

To trace network paths:

```
# traceroute openbsd.org
```

To inspect wireless signal strength:

```sh
# ifconfig iwn0
```

To view available wireless networks:

```sh
# ifconfig iwn0 scan
```

To send an IPv6 multicast ping:

```
# ping6 ff02::1%iwn0
```

To capture packets:

```
# tcpdump -i em0
```

Root privileges are required for `tcpdump`.

## Secure Networking Practices

Keep firmware up to date:

```
# fw_update
```

Avoid connecting to unsecured networks. Use strong WPA2 or WPA3 passphrases.

To enable a basic firewall:

```sh
# cat /etc/pf.conf
set block-policy drop
block all
pass out all keep state
pass in on em0 proto tcp to port 22 keep state
```

Enable and load the firewall:

```sh
# rcctl enable pf
# pfctl -f /etc/pf.conf
# pfctl -e
```

To monitor blocked traffic:

```
# tcpdump -n -e -ttt -i pflog0
```

## VLAN Configuration

To configure a VLAN interface:

```sh
# cat /etc/hostname.vlan0
vnetid 10 parent em0
inet 192.168.10.100 255.255.255.0 192.168.10.1
```

Ensure the switch port connected to `em0` is set as a trunk and supports VLAN ID 10.

## Local DNS Caching

To enable local DNS caching with `unbound(8)`:

```sh
# rcctl enable unbound
# rcctl start unbound
```

Configure `/etc/resolv.conf` to use `127.0.0.1` as a nameserver.

## Performance Tuning

To set the Maximum Transmission Unit (MTU):

```sh
# ifconfig em0 mtu 9000
```

To disable features like TCP segmentation offload (TSO):

```sh
# ifconfig em0 -tso
```

Refer to `ifconfig(8)` for other tunable options.
