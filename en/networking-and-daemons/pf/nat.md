# PF NAT

## Introduction

Network Address Translation (NAT) is a way to map an entire network (or networks) to a single IP address. NAT is necessary when the number of IP addresses assigned to you by your Internet Service Provider is less than the total number of computers that you wish to provide internet access for. NAT is described in [RFC 1631](https://tools.ietf.org/html/rfc1631)
.

NAT allows you to take advantage of the reserved address blocks described in [RFC 1918](http://tools.ietf.org/html/rfc1918)
. Typically, your internal network will be setup to use one or more of these network blocks. They are:

```
10.0.0.0/8       (10.0.0.0 - 10.255.255.255)
172.16.0.0/12    (172.16.0.0 - 172.31.255.255)
192.168.0.0/16   (192.168.0.0 - 192.168.255.255)
```

An OpenBSD system doing NAT will have at least two network interfaces, one to the internet, the other to your internal network. NAT will be translating requests from the internal network so they appear to all be coming from your OpenBSD NAT system.

## How NAT Works

When a client on the internal network contacts a machine on the internet, it sends out IP packets destined for that machine. These packets contain all the addressing information necessary to get them to their destination. NAT is concerned with these pieces of information:

- Source IP address (for example, 192.168.1.35)
- Source TCP or UDP port (for example, 2132)

When the packets pass through the NAT gateway, they will be modified so that they appear to be coming from the NAT gateway itself. The NAT gateway will record the changes it makes in its state table so that it can a) reverse the changes on return packets and b) ensure that return packets are passed through the firewall and are not blocked. For example, the following changes might be made:

- Source IP: replaced with the external address of the gateway (for example, 198.51.100.1)
- Source port: replaced with a randomly chosen, unused port on the gateway (for example, 53136)

Neither the internal machine nor the Internet host is aware of these translation steps. To the internal machine, the NAT system is simply an internet gateway. To the internet host, the packets appear to come directly from the NAT system; it is completely unaware that the internal workstation even exists.

When the internet host replies to the internal machine’s packets, they will be addressed to the NAT gateway’s external IP (198.51.100.1) at the translation port (53136). The NAT gateway will then search the state table to determine if the reply packets match an already established connection. A unique match will be found based on the IP/port combination which tells PF the packets belong to a connection initiated by the internal machine 192.168.1.35. PF will then make the opposite changes it made to the outgoing packets and forward the reply packets on to the internal machine.

Translation of ICMP packets happens in a similar fashion but without the source port modification.

## IP Forwarding

Since NAT is almost always used on routers and network gateways, it will probably be necessary to enable IP forwarding so that packets can travel between network interfaces on the OpenBSD machine. IP forwarding is enabled using the sysctl mechanism:

```sh
# sysctl net.inet.ip.forwarding=1
# echo  'net.inet.ip.forwarding=1' >> /etc/sysctl.conf
```

Or, for IPv6:

```sh
# sysctl net.inet6.ip6.forwarding=1
# echo  'net.inet6.ip6.forwarding=1' >> /etc/sysctl.conf
```

## Configuring NAT

NAT is specified as an optional ’nat-to’ parameter to an outbound ‘pass’ rule. Often, rather than being set directly on the ‘pass’ rule, a ‘match’ rule is used. When a packet is selected by a ‘match’ rule, parameters (e.g. ’nat-to’) in that rule are remembered and are applied to the packet when a ‘pass’ rule matching the packet is reached. This permits a whole class of packets to be handled by a single ‘match’ rule and then specific decisions on whether to allow the traffic can be made with ‘block’ and ‘pass’ rules.

The general format in ‘pf.conf’ looks something like this:

```ini
match out on interface [af] \
   from src_addr to dst_addr \
   nat-to ext_addr [pool_type] [static-port]
[...]
pass out [log] on interface [af] [proto protocol] \
   from ext_addr [port src_port] \
   to dst_addr [port dst_port]
```

***match***

When a packet traverses the ruleset and matches a ‘match’ rule, any optional parameters specified in that rule are remembered for future use (made “sticky”).

***pass***

This rule allows the packet to be transmitted. If the packet was previously matched by a ‘match’ rule where parameters were specified, they will be applied to this packet. ‘pass’ rules may have their own parameters; these take priority over parameters specified in a ‘match’ rule.

***out***

Specifies the direction of packet flow where this rule applies. ’nat-to’ may only be specified for outbound packets.

***log***

Log matching packets via pflogd. Normally, only the first packet that matches will be logged. To log all matching packets, use ’log (all)’.

***interface***

The name or group of the network interface to transmit packets on.

***af***

The address family, either ‘inet’ for IPv4 or ‘inet6’ for IPv6. PF is usually able to determine this parameter based on the source/destination address(es).

***protocol***

The protocol (e.g. tcp, udp, icmp) of packets to allow. If *src\_port* or *dst\_port* is specified, the protocol *must* also be given:

***src\_addr***

The source (internal) address of packets that will be translated. The source address can be specified as:

- A single IPv4 or IPv6 address.
- A [CIDR](https://web.archive.org/web/20150213012421/http://public.swbell.net/dedicated/cidr.html)
  network block.
- A fully qualified domain name that will be resolved via DNS when the ruleset is loaded. All resulting IP addresses will be substituted into the rule.
- The name or group of a network interface. Any IPv4 and IPv6 addresses assigned to the interface will be substituted into the rule at load time.
- The name of a network interface followed by ‘/*netmask*’ (e.g. ‘/24’). Each IP address on the interface is combined with the netmask to form a CIDR network block which is substituted into the rule.
- The name or group of a network interface followed by any one of these modifiers:

  - ‘:network’ - substitutes the CIDR network block (e.g., 192.168.0.0/24)
  - ‘:broadcast’ - substitutes the network broadcast address (e.g., 192.168.0.255)
  - ‘:peer’ - substitutes the peer’s IP address on a point-to-point link

    In addition, the ‘:0’ modifier can be appended to either an interface name/group or to any of the above modifiers to indicate that PF should not include aliased IP addresses in the substitution. These modifiers can also be used when the interface is contained in parentheses. Example: ‘fxp0:network:0’
- A [table](tables.md)
  .
- Any of the above but negated using the ‘!’ (“not”) modifier.
- A set of addresses using a [list](lists-and-macros.md#lists)
  .
- The keyword ‘any’ meaning all addresses

***src\_port***

The source port in the Layer 4 packet header. Ports can be specified as:

- A number between 1 and 65535
- A valid service name from [services(5)](https://man.openbsdhandbook.com/services.5/)
- A set of ports using a [list](lists-and-macros.md#lists)
- A range:
  - ‘!=’ (not equal)
  - ‘<’ (less than)
  - ‘>’ (greater than)
  - ‘<=’ (less than or equal)
  - ‘>=’ (greater than or equal)
  - ‘><’ (range)
  - ‘<>’ (inverse range)

    The last two are binary operators (they take two arguments) and do not include the arguments in the range.
  - ‘:’ (inclusive range)

    The inclusive range operator is also a binary operator and does include the arguments in the range.

The ‘port’ option is not usually used in ’nat’ rules because the goal is usually to NAT all traffic regardless of the port(s) being used.

***dst\_addr***

The destination address of packets to be translated. The destination address is specified in the same way as the source address.

***dst\_port***

The destination port in the Layer 4 packet header. This port is specified in the same way as the source port.

***ext\_addr***

The external (translation) address on the NAT gateway that packets will be translated to. The external address can be specified as:

- A single IPv4 or IPv6 address.
- A [CIDR](https://web.archive.org/web/20150611113606/http://public.swbell.net/dedicated/cidr.html)
  network block.
- A fully qualified domain name that will be resolved via DNS when the ruleset is loaded.
- The name or group of the external network interface. Any IP addresses assigned to the interface will be substituted into the rule at load time.
- The name or group of the external network interface in parentheses ‘( )’. This tells PF to update the rule if the IP address(es) on the named interface changes. This is highly useful when the external interface gets its IP address via DHCP or dial-up, as the ruleset doesn’t have to be reloaded each time the address changes.
- The name or group of a network interface followed by either one of these modifiers:

  - ‘:network’ - substitutes the CIDR network block (e.g., 192.168.0.0/24)
  - ‘:peer’ - substitutes the peer’s IP address on a point-to-point link

    In addition, the ‘:0’ modifier can be appended to either an interface name/group or to any of the above modifiers to indicate that PF should not include aliased IP addresses in the substitution. These modifiers can also be used when the interface is contained in parentheses. Example: ‘fxp0:network:0’
- A set of addresses using a [list](lists-and-macros.md#lists)
  .

***pool\_type***

Specifies the type of [address pool](https://www.openbsdhandbook.com/pf/pools)
to use for translation.

***static-port***

Tells PF not to translate the source port in TCP and UDP packets.

This would lead to a most basic form of these lines similar to this:

```pf
match out on tl0 from 192.168.1.0/24 to any nat-to 198.51.100.1
pass on tl0 from 192.168.1.0/24 to any
```

Or you may simply use:

```pf
pass out on tl0 from 192.168.1.0/24 to any nat-to 198.51.100.1
```

This rule says to perform NAT on the ’tl0’ interface for any packets coming from 192.168.1.0/24 and to replace the source IP address with 198.51.100.1.

While the above rule is correct, it is not recommended form. Maintenance could be difficult as any change of the external or internal network numbers would require the line be changed. Compare instead with this easier to maintain line (’tl0’ is external, ‘dc0’ internal):

```pf
pass out on tl0 inet from dc0:network to any nat-to tl0
```

The advantage should be fairly clear: you can change the IP addresses of either interface without changing this rule. Note that ‘inet’ should be specified in this case to ensure that only IPv4 addresses are used, avoiding unexpected surprises.

When specifying an interface name for the translation address as above, the IP address is determined at pf.conf *load* time, not on the fly. If you are using DHCP to configure your external interface, this can be a problem. If your assigned IP address changes, NAT will continue translating outgoing packets using the old IP address. This will cause outgoing connections to stop functioning. To get around this, you can tell PF to automatically update the translation address by putting parentheses around the interface name:

```pf
pass out on tl0 inet from dc0:network to any nat-to (tl0)
```

This method works for translation to both IPv4 and IPv6 addresses.

## Bidirectional Mapping (1:1 Mapping)

A bidirectional mapping can be established by using the ‘binat-to’ parameter. A ‘binat-to’ rule establishes a one to one mapping between an internal IP address and an external address. This can be useful, for example, to provide a web server on the internal network with its own external IP address. Connections from the Internet to the external address will be translated to the internal address and connections from the web server (such as DNS requests) will be translated to the external address. TCP and UDP ports are never modified with ‘binat-to’ rules as they are with ’nat’ rules.

Example:

```conf
web_serv_int = "192.168.1.100"
web_serv_ext = "198.51.100.6"

pass on tl0 from $web_serv_int to any binat-to $web_serv_ext
```

## Translation Rule Exceptions

If you need to translate most traffic, but provide exceptions in some cases, make sure that the exceptions are handled by a filter rule which does not include the nat-to parameter. For example, if the NAT example above was modified to look like this:

```
pass  out on tl0 from 192.168.1.0/24 to any nat-to 198.51.100.79
pass  out on tl0 from 192.168.1.208  to any
```

Then the entire 192.168.1.0/24 network would have its packets translated to the external address 198.51.100.79 except for 192.168.1.208.

## Checking NAT Status

To view the active NAT translations, pfctl is used with the ‘-s state’ option. This option will list all the current NAT sessions:

```sh
# pfctl -s state
fxp0 tcp 192.168.1.35:2132 (198.51.100.1:53136) -> 198.51.100.10:22 TIME_WAIT:TIME_WAIT
fxp0 udp 192.168.1.35:2491 (198.51.100.1:60527) -> 198.51.100.33:53   MULTIPLE:SINGLE
```

Explanations (first line only):

*fxp0*

Indicates the interface that the state is bound to. The word ‘self’ will appear if the state is [floating](options.html#state-policy)
.

*tcp*

The protocol being used by the connection.

*192.168.1.35:2132*

The IP address (192.168.1.35) of the machine on the internal network. The source port (2132) is shown after the address. This is also the address that is replaced in the IP header.

*198.51.100.1:53136*

The IP address (198.51.100.1) and port (53136) on the gateway that packets are being translated to.

*198.51.100.10:22*

The IP address (198.51.100.10) and the port (22) that the internal machine is connecting to.

*TIME\_WAIT:TIME\_WAIT*

This indicates what state PF believes the TCP connection to be in.
