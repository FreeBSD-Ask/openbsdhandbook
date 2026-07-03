# PF Load Balancing

## Introduction

An address pool is a supply of two or more addresses whose use is shared among a group of users. It can be specified as the target address in ’nat-to’, ‘rdr-to’,‘route-to’, ‘reply-to’ and ‘dup-to’ [filter](filter.md)
options.

There are four methods for using an address pool:

- ‘bitmask’ - grafts the network portion of the pool address over top of the address that is being modified (source address for ’nat-to’ rules, destination address for ‘rdr-to’ rules). Example: if the address pool is 192.0.2.1/24 and the address being modified is 10.0.0.50, then the resulting address will be 192.0.2.50. If the address pool is 192.0.2.1/25 and the address being modified is 10.0.0.130, then the resulting address will be 192.0.2.2.
- ‘random’ - randomly selects an address from the pool.
- ‘source-hash’ - uses a hash of the source address to determine which address to use from the pool. This method ensures that a given source address is always mapped to the same pool address. The key that is fed to the hashing algorithm can optionally be specified after the ‘source-hash’ keyword in hex format or as a string. By default, pfctl will generate a random key every time the ruleset is loaded.
- ‘round-robin’ - loops through the address pool in sequence. This is the default method and also the only method allowed when the address pool is specified using a [table](tables.md)
  .

Except for the ‘round-robin’ method, the address pool must be expressed as a [CIDR](https://web.archive.org/web/20150213012421/http://public.swbell.net/dedicated/cidr.html)
(Classless Inter-Domain Routing) network block. The ‘round-robin’ method will accept multiple individual addresses using a [list](lists-and-macros.md#lists)
or [table](tables.md)
.

The ‘sticky-address’ option can be used with the ‘random’ and ‘round-robin’ pool types to ensure that a particular source address is always mapped to the same redirection address.

## NAT Address Pool

An address pool can be used as the translation address in [’nat-to’](nat.md)
rules. Connections will have their source address translated to an address from the pool based on the method chosen. This can be useful in situations where PF is performing NAT for a very large network. Since the number of NATed connections per translation address is limited, adding additional translation addresses will allow the NAT gateway to scale to serve a larger number of users.

In this example, a pool of two addresses is being used to translate outgoing packets. For each outgoing connection, PF will rotate through the addresses in a round-robin manner.

```pf
match out on egress inet nat-to { 192.0.2.5, 192.0.2.10 }
```

One drawback with this method is that successive connections from the same internal address will not always be translated to the same translation address. This can cause interference, for example, when browsing websites that track user logins based on IP address. An alternate approach is to use the ‘source-hash’ method so that each internal address is always translated to the same translation address. To do this, the address pool must be a [CIDR](https://web.archive.org/web/20150213012421/http://public.swbell.net/dedicated/cidr.html)
network block.

```pf
match out on egress inet nat-to 192.0.2.4/31 source-hash
```

This rule uses the address pool 192.0.2.4/31 (192.0.2.4 - 192.0.2.5) as the translation address for outgoing packets. Each internal address will always be translated to the same translation address because of the ‘source-hash’ keyword.

## Load Balance Incoming Connections

Address pools can also be used to load balance incoming connections. For example, incoming web server connections can be distributed across a web server farm:

```pf
web_servers = "{ 10.0.0.10, 10.0.0.11, 10.0.0.13 }"

match in on egress proto tcp to port 80 rdr-to $web_servers \
    round-robin sticky-address
```

Successive connections will be redirected to the web servers in a round-robin manner with connections from the same source being sent to the same web server. This “sticky connection” will exist as long as there are states that refer to this connection. Once the states expire, so will the sticky connection. Further connections from that host will be redirected to the next web server in the round robin.

## Load Balance Outgoing Traffic

Address pools can be used in combination with the ‘route-to’ filter option to load balance two or more Internet connections when a proper multi-path routing protocol (like [BGP4](http://www.rfc-editor.org/rfc/rfc1771.txt)
) is unavailable. By using ‘route-to’ with a ‘round-robin’ address pool, outbound connections can be evenly distributed among multiple outbound paths.

One additional piece of information that’s needed to do this is the IP address of the adjacent router on each Internet connection. This is fed to the ‘route-to’ option to control the destination of outgoing packets.

The following example balances outgoing traffic across two Internet connections:

```pf
lan_net = "192.168.0.0/24"
int_if  = "dc0"
ext_if1 = "fxp0"
ext_if2 = "fxp1"
ext_gw1 = "198.51.100.100"
ext_gw2 = "203.0.113.200"

pass in on $int_if from $lan_net route-to \
   { ($ext_if1 $ext_gw1), ($ext_if2 $ext_gw2) } round-robin
```

The ‘route-to’ option is used on traffic coming *in* on the *internal* interface to specify the outgoing network interfaces that traffic will be balanced across along with their respective gateways. Note that the ‘route-to’ option must be present on *each* filter rule that traffic is to be balanced for (it cannot be used with ‘match’ rules).

To ensure that packets with a source address belonging to ‘$ext\_if1’ are always routed to ‘$ext\_gw1’ (and similarly for ‘$ext\_if2’ and ‘$ext\_gw2’), the following two lines should be included in the ruleset:

```pf
pass out on $ext_if1 from $ext_if2 route-to ($ext_if2 $ext_gw2)
pass out on $ext_if2 from $ext_if1 route-to ($ext_if1 $ext_gw1)
```

Finally, NAT can also be used on each outgoing interface:

```pf
match out on $ext_if1 from $lan_net nat-to ($ext_if1)
match out on $ext_if2 from $lan_net nat-to ($ext_if2)
```

A complete example that load balances outgoing traffic might look something like this:

```pf
lan_net = "192.168.0.0/24"
int_if  = "dc0"
ext_if1 = "fxp0"
ext_if2 = "fxp1"
ext_gw1 = "198.51.100.100"
ext_gw2 = "203.0.113.200"

#  nat outgoing connections on each internet interface
match out on $ext_if1 from $lan_net nat-to ($ext_if1)
match out on $ext_if2 from $lan_net nat-to ($ext_if2)

#  default deny
block in
block out

#  pass all outgoing packets on internal interface
pass out on $int_if to $lan_net
#  pass in quick any packets destined for the gateway itself
pass in quick on $int_if from $lan_net to $int_if
#  load balance outgoing traffic from internal network.
pass in on $int_if from $lan_net \
    route-to { ($ext_if1 $ext_gw1), ($ext_if2 $ext_gw2) } \
    round-robin
#  keep https traffic on a single connection; some web applications,
#  especially "secure" ones, don't allow it to change mid-session
pass in on $int_if proto tcp from $lan_net to port https \
    route-to ($ext_if1 $ext_gw1)

#  general "pass out" rules for external interfaces
pass out on $ext_if1
pass out on $ext_if2

#  route packets from any IPs on $ext_if1 to $ext_gw1 and the same for
#  $ext_if2 and $ext_gw2
pass out on $ext_if1 from $ext_if2 route-to ($ext_if2 $ext_gw2)
pass out on $ext_if2 from $ext_if1 route-to ($ext_if1 $ext_gw1)
```
