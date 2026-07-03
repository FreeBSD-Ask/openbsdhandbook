# PF Policies

## Introduction

Packet tagging is a way of marking packets with an internal identifier that can later be used in filter and translation rule criteria. With tagging, it’s possible to do such things as create “trusts” between interfaces and determine if packets have been processed by translation rules. It’s also possible to move away from rule-based filtering and to start doing policy-based filtering.

## Assigning Tags to Packets

To add a tag to a packet, use the ’tag’ keyword:

```pf
pass in on $int_if all tag INTERNAL_NET
```

The tag ‘INTERNAL\_NET’ will be added to any packet which matches the above rule.

A tag can also be assigned using a [macro](lists-and-macros.md#macros)
. For instance:

```pf
name = "INTERNAL_NET"
pass in on $int_if all tag $name
```

There are a set of predefined macros which can also be used.

- ‘$if’ - The interface
- ‘$srcaddr’ - Source IP address
- ‘$dstaddr’ - Destination IP address
- ‘$srcport’ - The source port specification
- ‘$dstport’ - The destination port specification
- ‘$proto’ - The protocol
- ‘$nr’ - The rule number

These macros are expanded at ruleset load time and NOT at runtime.

Tagging follows these rules:

- Tags are “sticky.” Once a tag is applied to a packet by a matching rule, it is never removed. It can, however, be replaced with a different tag.
- Because of a tag’s “stickiness,” a packet can have a tag even if the last matching rule doesn’t use the ’tag’ keyword.
- A packet is only ever assigned a maximum of one tag at a time.
- Tags are *internal* identifiers. Tags are not sent out over the wire.
- Tag names can be up to 63 characters long.

Take the following ruleset as an example.

```pf
pass in on $int_if tag INT_NET
pass in quick on $int_if proto tcp to port 80 tag INT_NET_HTTP
pass in quick on $int_if from 192.168.1.5
```

- Packets coming in on ‘$int\_if’ will be assigned a tag of ‘INT\_NET’ by rule #1.
- TCP packets coming in on ‘$int\_if’ and destined for port 80 will first be assigned a tag of ‘INT\_NET’ by rule #1. That tag will then be replaced with the ‘INT\_NET\_HTTP’ tag by rule #2.
- Packets coming in on ‘$int\_if’ from 192.168.1.5 will be tagged one of two ways. If the packet is destined for TCP port 80 it will match rule #2 and be tagged with ‘INT\_NET\_HTTP’. Otherwise, the packet will match rule #3 but will be tagged with ‘INT\_NET’. Because the packet matches rule #1, the ‘INT\_NET’ tag is applied and is not removed unless a subsequently matching rule specifies a tag (this is the “stickiness” of a tag).

## Checking for Applied Tags

To check for previously applied tags, use the ’tagged’ keyword:

```pf
pass out on egress tagged INT_NET
```

Outgoing packets on the external interface must be tagged with the ‘INT\_NET’ tag in order to match the above rule. Inverse matching can also be done by using the ‘!’ operator:

```pf
pass out on egress ! tagged WIFI_NET
```

## Policy Filtering

Policy filtering takes a different approach to writing a filter ruleset. A policy is defined which sets the rules for what types of traffic is passed and what types are blocked. Packets are then classified into the policy based on the traditional criteria of source/destination IP address/port, protocol, etc. For example, examine the following firewall policy:

- Traffic from the internal LAN to the Internet is permitted (LAN\_INET) and must be translated (LAN\_INET\_NAT).
- Traffic from the internal LAN to the DMZ is permitted (LAN\_DMZ).
- Traffic from the Internet to servers in the DMZ is permitted (INET\_DMZ).
- Traffic from the Internet that’s being redirected to [spamd(8)](https://man.openbsdhandbook.com/spamd.8/)
  is permitted (SPAMD).
- All other traffic is blocked.

Note how the policy covers *all* traffic that will be passing through the firewall. The item in parenthesis indicates the tag that will be used for that policy item.

Rules now need to be written to classify packets into the policy.

```pf
block all
pass out on egress inet tag LAN_INET_NAT tagged LAN_INET nat-to ($ext_if)
pass in  on $int_if from $int_net tag LAN_INET
pass in  on $int_if from $int_net to $dmz_net tag LAN_DMZ
pass in  on egress proto tcp to $www_server port 80 tag INET_DMZ
pass in  on egress proto tcp from <spamd> to port smtp tag SPAMD rdr-to 127.0.0.1 port 8025
```

Now the rules that define the policy are set.

```pf
pass in  quick on egress tagged SPAMD
pass out quick on egress tagged LAN_INET_NAT
pass out quick on $dmz_if tagged LAN_DMZ
pass out quick on $dmz_if tagged INET_DMZ
```

Now that the whole ruleset is setup, changes are a matter of modifying the classification rules. For example, if a POP3/SMTP server is added to the DMZ, it will be necessary to add classification rules for POP3 and SMTP traffic, like so:

```ini
mail_server = "192.168.0.10"
[...]
pass in on egress proto tcp to $mail_server port { smtp, pop3 } tag INET_DMZ
```

Email traffic will now be passed as part of the INET\_DMZ policy entry.

The complete ruleset:

```sh
int_if      = "dc0"
dmz_if      = "dc1"
int_net     = "10.0.0.0/24"
dmz_net     = "192.168.0.0/24"
www_server  = "192.168.0.5"
mail_server = "192.168.0.10"

table <spamd> persist file "/etc/spammers"
# classification -- classify packets based on the defined firewall policy.
block all
pass out on egress inet tag LAN_INET_NAT tagged LAN_INET nat-to (egress)
pass in on $int_if from $int_net tag LAN_INET
pass in on $int_if from $int_net to $dmz_net tag LAN_DMZ
pass in on egress proto tcp to $www_server port 80 tag INET_DMZ
pass in on egress proto tcp from <spamd> to port smtp tag SPAMD rdr-to 127.0.0.1 port 8025

# policy enforcement -- pass/block based on the defined firewall policy.
pass in  quick on egress tagged SPAMD
pass out quick on egress tagged LAN_INET_NAT
pass out quick on $dmz_if tagged LAN_DMZ
pass out quick on $dmz_if tagged INET_DMZ
```

## Tagging Ethernet Frames

Tagging can be performed at the ethernet level if the machine doing the tagging/filtering is also acting as a bridge. By creating bridge filter rules that use the ’tag’ keyword, PF can be made to filter based on the source or destination MAC address. Bridge(4) rules are created using the ifconfig command. Example:

```sh
# ifconfig bridge0 rule pass in on fxp0 src 0:de:ad:be:ef:0 tag USER1
```

And then in ‘pf.conf’:

```pf
pass in on fxp0 tagged USER1
```
