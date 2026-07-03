# PF Filter Rules

## Introduction

Packet filtering is the selective passing or blocking of data packets as they pass through a network interface. The criteria that pf uses when inspecting packets are based on the Layer 3 IPv4 and IPv6 and Layer 4 TCP, UDP, ICMP, and ICMPv6 headers. The most often used criteria are source and destination address, source and destination port, and protocol.

Filter rules specify the criteria that a packet must match and the resulting action, either block or pass, that is taken when a match is found. Filter rules are evaluated in sequential order, first to last. Unless the packet matches a rule containing the ‘quick’ keyword, the packet will be evaluated against *all* filter rules before the final action is taken. The last rule to match is the “winner” and will dictate what action to take on the packet. There is an implicit ‘pass all’ at the beginning of a filtering ruleset, meaning that if a packet does not match any filter rule the resulting action will be ‘pass’.

## Rule Syntax

The general, *highly simplified*, syntax for filter rules is:

```ini
action [direction] [log] [quick] [on interface] [af] [proto protocol]
       [from src_addr [port src_port]] [to dst_addr [port dst_port]]
       [flags tcp_flags] [state]
```

***action***

The action to be taken for matching packets, either ‘pass’ or ‘block’. The ‘pass’ action will pass the packet back to the kernel for further processing while the ‘block’ action will react based on the setting of the [‘block-policy’](options.md#set-block-policy-option)
option. The default reaction may be overridden by specifying either ‘block drop’ or ‘block return’.

***direction***

The direction the packet is moving on an interface, either ‘in’ or ‘out’.

***log***

Specifies that the packet should be logged via pflogd. If the rule creates state then only the packet which establishes the state is logged. To log all packets regardless, use ’log (all)’.

***quick***

If a packet matches a rule specifying ‘quick’, then that rule is considered the last matching rule and the specified ‘*action*’ is taken.

***interface***

The name or group of the network interface the packet is moving through. Interfaces can be added to arbitrary groups using the ifconfig command. Several groups are also automatically created by the kernel:

- The ’egress’ group, which contains the interface(s) that holds the default route(s).
- Interface family group for cloned interfaces. For example: ‘ppp’ or ‘carp’.

This would cause the rule to match for any packet traversing any ‘ppp’ or ‘carp’ interface, respectively.

***af***

The address family of the packet, either ‘inet’ for IPv4 or ‘inet6’ for IPv6. PF is usually able to determine this parameter based on the source and/or destination address(es).

***protocol***

The Layer 4 protocol of the packet:

- ’tcp’
- ‘udp’
- ‘icmp’
- ‘icmp6’
- A valid protocol name from ‘/etc/protocols’
- A protocol number between 0 and 255
- A set of protocols using a [list](lists-and-macros.md#lists)
  .

***src\_addr***, ***dst\_addr***

The source/destination address in the IP header. Addresses can be specified as:

- A single IPv4 or IPv6 address.
- A [CIDR](https://web.archive.org/web/20150213012421/http://public.swbell.net/dedicated/cidr.html)
  network block.
- A fully qualified domain name that will be resolved via DNS when the ruleset is loaded. All resulting IP addresses will be substituted into the rule.
- The name of a network interface or group. Any IP addresses assigned to the interface will be substituted into the rule.
- The name of a network interface followed by ‘/*netmask*’ (i.e., ‘/24’). Each IP address on the interface is combined with the netmask to form a CIDR network block which is substituted into the rule.
- The name of a network interface or group in parentheses ‘( )’. This tells PF to update the rule if the IP address(es) on the named interface change. This is useful on an interface that gets its IP address via DHCP or dial-up as the ruleset doesn’t have to be reloaded each time the address changes.
- The name of a network interface followed by any one of these modifiers:

  - ‘:network’ - substitutes the CIDR network block (e.g., 192.168.0.0/24)
  - ‘:broadcast’ - substitutes the network broadcast address (e.g., 192.168.0.255)
  - ‘:peer’ - substitutes the peer’s IP address on a point-to-point link

    In addition, the ‘:0’ modifier can be appended to either an interface name or to any of the above modifiers to indicate that PF should not include aliased IP addresses in the substitution. These modifiers can also be used when the interface is contained in parentheses. Example: ‘fxp0:network:0’
- A [table](tables.md)
  .
- The keyword ‘urpf-failed’ can be used for the source address to indicate that it should be run through the [uRPF check](#urpf)
  .
- Any of the above but negated using the ‘!’ (“not”) modifier.
- A set of addresses using a [list](lists-and-macros.md#lists)
  .
- The keyword ‘any’ meaning all addresses
- The keyword ‘all’ which is short for ‘from any to any’.

***src\_port***, ***dst\_port***

The source/destination port in the Layer 4 packet header. Ports can be specified as:

- A number between 1 and 65535
- A valid service name from [’/etc/services’](https://man.openbsdhandbook.com/services.5/)
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

***tcp\_flags***

Specifies the flags that must be set in the TCP header when using ‘proto tcp’. Flags are specified as ‘flags *check*/*mask*’. For example: ‘flags S/SA’ - this instructs PF to only look at the S and A (SYN and ACK) flags and to match if only the SYN flag is “on” (and is applied to all TCP rules by default). ‘flags any’ instructs PF not to check flags.

***state***

Specifies whether state information is kept on packets matching this rule.

- ’no state’ - works with TCP, UDP, and ICMP. PF will not track this connection statefully. For TCP connections, ‘flags any’ is usually also required.
- ‘keep state’ - works with TCP, UDP, and ICMP. This option is the default for all filter rules.
- ‘modulate state’ - works only with TCP. PF will generate strong Initial Sequence Numbers (ISNs) for packets matching this rule.
- ‘synproxy state’ - proxies incoming TCP connections to help protect servers from spoofed TCP SYN floods. This option includes the functionality of ‘keep state’ and ‘modulate state’.

## Default Deny

The recommended practice when setting up a firewall is to take a “default deny” approach. That is to deny *everything*, and then selectively allow certain traffic through the firewall. This approach is recommended because it errs on the side of caution and also makes writing a ruleset easier.

To create a default deny filter policy, the first filter rule should be:

```
block all
```

This will block all traffic on all interfaces in either direction from anywhere to anywhere.

## Passing Traffic

Traffic must now be explicitly passed through the firewall or it will be dropped by the default deny policy. This is where packet criteria such as source/destination port, source/destination address and protocol come into play. Whenever traffic is permitted to pass through the firewall, the rule(s) should be written to be as restrictive as possible. This is to ensure that the intended traffic, and only the intended traffic, is permitted to pass.

Some examples:

```pf
# Pass traffic in on dc0 from the local network, 192.168.0.0/24, to the OpenBSD
# machine's IP address 192.168.0.1\. Also, pass the return traffic out on dc0.
pass in  on dc0 from 192.168.0.0/24 to 192.168.0.1
pass out on dc0 from 192.168.0.1    to 192.168.0.0/24

# Pass TCP traffic in to the web server running on the OpenBSD machine.
pass in on egress proto tcp from any to egress port www
```

## The ‘quick’ Keyword

As indicated earlier, each packet is evaluated against the filter ruleset from top to bottom. By default, the packet is marked for passage, which can be changed by any rule, and could be changed back and forth several times before the end of the filter rules. **The last matching rule wins.** There is an exception to this: The ‘quick’ option on a filtering rule has the effect of canceling any further rule processing and causes the specified action to be taken. Let’s look at a couple examples:

Wrong:

```pf
block in on egress proto tcp to port ssh
pass  in all
```

In this case, the ‘block’ line may be evaluated, but will never have any effect, as it is then followed by a line which will pass everything.

Better:

```pf
block in quick on egress proto tcp to port ssh
pass  in all
```

These rules are evaluated a little differently. If the ‘block’ line is matched, due to the ‘quick’ option, the packet will be blocked, and the rest of the ruleset will be ignored.

## Keeping State

One of PF’s important abilities is “keeping state” or “stateful inspection.” Stateful inspection refers to PF’s ability to track the state, or progress, of a network connection. By storing information about each connection in a state table, PF is able to quickly determine if a packet passing through the firewall belongs to an already-established connection. If it does, it is passed through the firewall without going through ruleset evaluation.

Keeping state has many advantages, including simpler rulesets and better packet filtering performance. PF is able to match packets moving in *either* direction to state table entries, meaning that filter rules which pass returning traffic don’t need to be written. Since packets matching stateful connections don’t go through ruleset evaluation, the time PF spends processing those packets can be greatly lessened.

When a rule creates state, the first packet matching the rule creates a “state” between the sender and receiver. Now, not only do packe firewall belongs to an already-established connection. If it does, it is passed through the firewall without going through ruleset evaluation.

Keeping state has many advantages, including simpler rulesets and better packet filtering performance. PF is able to match packets moving in *either* direction to state table entriegress proto tcp from any to any

This rule allows any outbound TCP traffic on the egress interface and also permits the reply traffic to pass back through the firewall. Keeping state significantly improves the performance of your firewall as state lookups are dramatically faster than running a packet through the filter rules.

The ‘modulate state’ option works just like ‘keep state’, except that it only applies to TCP packets. With ‘modulate state’, the Initial Sequence Number (ISN) of outgoing connections is randomized. This is useful for protecting connections initiated by certain operating systems that do a poor job of choosing ISNs. To allow simpler rulesets, the ‘modulate state’ option can be used in rules that specify protocols other than TCP. In those cases, it is treated as ‘keep state’.

Keep state on outgoing TCP, UDP and ICMP packets and modulate TCP ISNs:

```pf
pass out on egress proto { tcp, udp, icmp } from any to any modulate state
```

Another advantage of keeping state is that corresponding ICMP traffic will be passed through the firewall. For example, if a TCP connection passing through the firewall is being tracked statefully and an ICMP source-quench message referring to this TCP connection arrives, it will be matched to the appropriate state entry and passed through the firewall.

The scope of a state entry is controlled globally by the [state-policy](options.md#set-state-policy-option)
runtime option, and on a per-rule basis by the ‘if-bound’ and ‘floating’ state option keywords. These per-rule keywords have the same meaning as when used with the ‘state-policy’ option. For example:

```pf
pass out on egress proto { tcp, udp, icmp } from any to any modulate state (if-bound)
```

This rule would dictate that in order for packets to match the state entry, they must be transiting the egress interface.

## Keeping State for UDP

One will sometimes hear it said that “one cannot create state with UDP, as UDP is a stateless protocol!” While it is true that a UDP communication session does not have any concept of state (an explicit start and stop of communications), this does not have any impact on PF’s ability to create state for a UDP session. In the case of protocols without “start” and “end” packets, PF simply keeps track of how long it has been since a matching packet has gone through. If the timeout is reached, the state is cleared. The timeout values can be set in the [options](options.md#set-timeout-option-value)
section of the ‘pf.conf’ file.

## Stateful Tracking Options

Filter rules that create state entries can specify various options to control the behavior of the resulting state entry. The following options are available:

***max \_number***

Limit the maximum number of state entries the rule can create to *number*. If the maximum is reached, packets that would normally create state fail to match this rule until the number of existing states decreases below the limit.

***no state***

Prevents the rule from automatically creating a state entry.

***source-track***

This option enables the tracking of number of states created per source IP address. This option has two formats:

- ‘source-track rule’ - The maximum number of states created by this rule is limited by the rule’s ‘max-src-nodes’ and ‘max-src-states’ options. Only state entries created by this particular rule count toward the rule’s limits.
- ‘source-track global’ - The number of states created by all rules that use this option is limited. Each rule can specify different ‘max-src-nodes’ and ‘max-src-states’ options, however state entries created by any participating rule count towards each individual rule’s limits.

The total number of source IP addresses tracked globally can be controlled via the [‘src-nodes’ runtime option](options.md#set-limit-option-value)
.

***max-src-nodes \_number***

When the ‘source-track’ option is used, ‘max-src-nodes’ will limit the number of source IP addresses that can simultaneously create state. This option can only be used with ‘source-track rule’.

***max-src-states \_number***

When the ‘source-track’ option is used, ‘max-src-states’ will limit the number of simultaneous state entries that can be created per source IP address. The scope of this limit (i.e., states created by this rule only or states created by all rules that use ‘source-track’) is dependent on the ‘source-track’ option specified.

Options are specified inside parenthesis and immediately after one of the state keywords (‘keep state’, ‘modulate state’, or ‘synproxy state’). Multiple options are separated by commas. The ‘keep state’ option is the implicit default for all filter rules. Despite this, when specifying stateful options, one of the state keywords must still be used in front of the options.

An example rule:

```pf
pass in on egress proto tcp to $web_server port www keep state   \
                  (max 200, source-track rule, max-src-nodes 100, \
                   max-src-states 3)
```

The rule above defines the following behavior:

- Limit the absolute maximum number of states that this rule can create to 200.
- Enable source tracking: limit state creation based on states created by this rule only.
- Limit the maximum number of nodes that can simultaneously create state to 100.
- Limit the maximum number of simultaneous states per source IP to 3.

A separate set of restrictions can be placed on stateful TCP connections that have completed the 3-way handshake.

**max-src-conn** ***number***

Limit the maximum number of simultaneous TCP connections which have completed the 3-way handshake that a single host can make.

**max-src-conn-rate** ***number*** / ***interval***

Limit the rate of new connections to a certain amount per time interval.

Both of these options automatically invoke the ‘source-track rule’ option and are incompatible with ‘source-track global’.

Since these limits are only being placed on TCP connections that have completed the 3-way handshake, more aggressive actions can be taken on offending IP addresses.

**overload** ***<*table*>***

Put an offending host’s IP address into the named table.

**flush** ***global***

Kill any other states that match this rule and that were created by this source IP. When ‘global’ is specified, kill all states matching this source IP, regardless of which rule created the state.

An example:

```pf
table <abusive_hosts> persist
block in quick from <abusive_hosts>

pass in on egress proto tcp to $web_server port www flags S/SA keep state \
                                (max-src-conn 100, max-src-conn-rate 15/5, \
                                 overload <abusive_hosts> flush)
```

This does the following:

- Limits the maximum number of connections per source to 100.
- Rate limits the number of connections to 15 in a 5 second span.
- Puts the IP address of any host that breaks these limits into the ‘<abusive\_hosts>’ table.
- For any offending IP addresses, flush any states created by this rule.

## TCP Flags

Matching TCP packets based on flags is most often used to filter TCP packets that are attempting to open a new connection. The TCP flags and their meanings are listed here:

- **F** : FIN - Finish; end of session
- **S** : SYN - Synchronize; indicates request to start session
- **R** : RST - Reset; drop a connection
- **P** : PUSH - Push; packet is sent immediately
- **A** : ACK - Acknowledgement
- **U** : URG - Urgent
- **E** : ECE - Explicit Congestion Notification Echo
- **W** : CWR - Congestion Window Reduced

To have PF inspect the TCP flags during evaluation of a rule, the ‘flags’ keyword is used with the following syntax:

```
flags _check_/_mask_
flags any
```

The ‘*mask*’ part tells PF to only inspect the specified flags and the ‘*check*’ part specifies which flag(s) must be “on” in the header for a match to occur. Using the ‘any’ keyword allows any combination of flags to be set in the header.

```pf
pass in on egress proto tcp from any to any port ssh flags S/SA
pass in on egress proto tcp from any to any port ssh
```

As ‘flags S/SA’ is set by default, the above rules are equivalent, Each of these rules passes TCP traffic with the SYN flag set while only looking at the SYN and ACK flags. A packet with the SYN and ECE flags would match the above rules, while a packet with SYN and ACK or just ACK would not.

The default flags can be overridden by using the ‘flags’ option as outlined above.

One should be careful with using flags – understand what you are doing and why, and be careful with the advice people give as a lot of it is bad. Some people have suggested creating state “only if the SYN flag is set and no others.” Such a rule would end with:

```ini
[...] flags S/FSRPAUEW  _bad idea!!_
```

The theory is to create state only on the start of the TCP session, and the session should start with a SYN flag, and no others. The problem is some sites are starting to use the ECN flag and any site using ECN that tries to connect to you would be rejected by such a rule. A much better guideline is to not specify any flags at all and let PF apply the default flags to your rules. If you truly need to specify flags yourself, then this combination should be safe:

```ini
[...] flags S/SAFR
```

While this is practical and safe, it is also unnecessary to check the FIN and RST flags if traffic is also being scrubbed. The scrubbing process will cause PF to drop any incoming packets with illegal TCP flag combinations (such as SYN and RST) and to normalize potentially ambiguous combinations (such as SYN and FIN).

## TCP SYN Proxy

Normally when a client initiates a TCP connection to a server, PF will pass the [handshake](http://www.inetdaemon.com/tutorials/internet/tcp/3-way_handshake.shtml)
packets between the two endpoints as they arrive. PF has the ability, however, to proxy the handshake. With the handshake proxied, PF itself will complete the handshake with the client, initiate a handshake with the server, and then pass packets between the two. In the case of a TCP SYN flood attack, the attacker never completes the three-way handshake, so the attacker’s packets never reach the protected server, but legitimate clients will complete the handshake and get passed. This minimizes the impact of spoofed TCP SYN floods on the protected service, handling it in PF instead. Routine use of this option is not recommended, however, as it breaks expected TCP protocol behavior when the server can’t process the request and when load balancers are involved.

The TCP SYN proxy is enabled using the ‘synproxy state’ keywords in filter rules. For example:

```pf
pass in on egress proto tcp to $web_server port www synproxy state
```

Here, connections to the web server will be TCP proxied by PF.

Because of the way ‘synproxy state’ works, it also includes the same functionality as ‘keep state’ and ‘modulate state’.

The SYN proxy will not work if PF is running on a bridge.

## Blocking Spoofed Packets

Address “spoofing” is when a malicious user fakes the source IP address in packets they transmit in order to either hide their real address or to impersonate another node on the network. Once the user has spoofed their address, they can launch a network attack without revealing the true source of the attack or attempt to gain access to network services that are restricted to certain IP addresses.

PF offers some protection against address spoofing through the ‘antispoof’ keyword:

```pf
antispoof [log] [quick] for interface [af]
```

***log***

Specifies that matching packets should be logged via pflogd.

***quick***

If a packet matches this rule then it will be considered the “winning” rule and ruleset evaluation will stop.

***interface***

The network interface to activate spoofing protection on. This can also be a [list](lists-and-macros.md#lists)
of interfaces.

***af***

The address family to activate spoofing protection for, either ‘inet’ for IPv4 or ‘inet6’ for IPv6.

Example:

```pf
antispoof for fxp0 inet
```

When a ruleset is loaded, any occurrences of the ‘antispoof’ keyword are expanded into two filter rules. Assuming that the egress interface has IP address 10.0.0.1 and a subnet mask of 255.255.255.0 (i.e., a /24), the above ‘antispoof’ rule would expand to:

```pf
block in on ! fxp0 inet from 10.0.0.0/24 to any
block in inet from 10.0.0.1 to any
```

These rules accomplish two things:

- Blocks all traffic coming from the 10.0.0.0/24 network that does *not* pass in through the ‘fxp0’ interface. Since the 10.0.0.0/24 network is on the ‘fxp0’ interface, packets with a source address in that network block should never be seen coming in on any other interface.
- Blocks all incoming traffic from 10.0.0.1, the IP address on ‘fxp0’. The host machine should never send packets to itself through an external interface, so any incoming packets with a source address belonging to the machine can be considered malicious.

**NOTE**: The filter rules that the ‘antispoof’ rule expands to will also block packets sent over the loopback interface to local addresses. It’s best practice to skip filtering on loopback interfaces anyways, but this becomes a necessity when using antispoof rules:

```pf
set skip on lo0
antispoof for fxp0 inet
```

Usage of ‘antispoof’ should be restricted to interfaces that have been assigned an IP address. Using ‘antispoof’ on an interface without an IP address will result in filter rules such as:

```
block drop in on ! fxp0 inet all
block drop in inet all
```

With these rules, there is a risk of blocking *all* inbound traffic on *all* interfaces.

## Unicast Reverse Path Forwarding

PF offers a Unicast Reverse Path Forwarding (uRPF) feature. When a packet is run through the uRPF check, the source IP address of the packet is looked up in the routing table. If the outbound interface found in the routing table entry is the same as the interface that the packet just came in on, then the uRPF check passes. If the interfaces don’t match, then it’s possible the packet has had its source address spoofed.

The uRPF check can be performed on packets by using the ‘urpf-failed’ keyword in filter rules:

```pf
block in quick from urpf-failed label uRPF
```

Note that the uRPF check only makes sense in an environment where routing is symmetric.

uRPF provides the same functionality as [antispoof](filter.md#blocking-spoofed-packets)
rules.

## Passive Operating System Fingerprinting

Passive OS fingerprinting (OSFP) is a method for passively detecting the operating system of a remote host based on certain characteristics within that host’s TCP SYN packets. This information can then be used as criteria within filter rules.

PF determines the remote operating system by comparing characteristics of a TCP SYN packet against the [fingerprints file](options.md#set-fingerprints-file)
, which by default is [pf.os](https://man.openbsdhandbook.com/pf.os.5/)
. Once PF is enabled, the current fingerprint list can be viewed with this command:

```sh
# pfctl -s osfp
```

Within a filter rule, a fingerprint may be specified by OS class, version, or subtype/patch level. Each of these items is listed in the output of the ‘pfctl’ command shown above. To specify a fingerprint in a filter rule, the ‘os’ keyword is used:

```pf
pass  in on egress proto tcp from any os OpenBSD
block in on egress proto tcp from any os "Windows 2000"
block in on egress proto tcp from any os "Linux 2.4 ts"
block in on egress proto tcp from any os unknown
```

The special operating system class ‘unknown’ allows for matching packets when the OS fingerprint is not known.

**TAKE NOTE** of the following:

- Operating system fingerprints are occasionally wrong due to spoofed and/or crafted packets that are made to look like they originated from a specific operating system.
- Certain revisions or patchlevels of an operating system may change the stack’s behavior and cause it to either not match what’s in the fingerprints file or to match another entry altogether.
- OSFP only works on the TCP SYN packet; it will not work on other protocols or on already established connections.

## IP Options

By default, PF blocks packets with IP options set. This can make the job more difficult for OS fingerprinting utilities like nmap. If you have an application that requires the passing of these packets, such as multicast or IGMP, you can use the ‘allow-opts’ directive:

```pf
pass in quick on fxp0 all allow-opts
```

## Filtering Ruleset Example

Below is an example of a filtering ruleset. The machine running PF is acting as a firewall between a small, internal network and the Internet. Only the filter rules are shown; ‘queueing’, [’nat’](nat.md)
, [‘rdr’](https://www.openbsdhandbook.com/pf/rdr)
, etc, have been left out of this example.

```pf
int_if  = "dc0"
lan_net = "192.168.0.0/24"

# table containing all IP addresses assigned to the firewall
table <firewall> const { self }

# don't filter on the loopback interface
set skip on lo0

# scrub incoming packets
match in all scrub (no-df)

# set up a default deny policy
block all

# activate spoofing protection for all interfaces
block in quick from urpf-failed

# only allow ssh connections from the local network if it's from the
# trusted computer, 192.168.0.15\. use "block return" so that a TCP RST is
# sent to close blocked connections right away. use "quick" so that this
# rule is not overridden by the "pass" rules below.
block return in quick on $int_if proto tcp from ! 192.168.0.15 to $int_if port ssh

# pass all traffic to and from the local network.
# these rules will create state entries due to the default
# "keep state" option which will automatically be applied.
pass in  on $int_if from $lan_net
pass out on $int_if to   $lan_net

# pass tcp, udp, and icmp out on the external (Internet) interface.
# tcp connections will be modulated, udp/icmp will be tracked statefully.
pass out on egress proto { tcp udp icmp } all modulate state

# allow ssh connections in on the external interface as long as they're
# NOT destined for the firewall (i.e., they're destined for a machine on
# the local network). log the initial packet so that we can later tell
# who is trying to connect.
# Uncomment last part to use the tcp syn proxy to proxy the connection.
pass in log on egress proto tcp to ! <firewall> port ssh # synproxy state
```
