# PF Shortcuts

## Introduction

PF offers many ways in which a ruleset can be simplified. Some good examples are by using [macros](lists-and-macros.md#macros)
and [lists](lists-and-macros.md#lists)
. In addition, the ruleset language, or grammar, also offers some shortcuts for making a ruleset simpler. As a general rule of thumb, the simpler a ruleset is, the easier it is to understand and to maintain.

## Using Macros

Macros are useful because they provide an alternative to hard-coding addresses, port numbers, interfaces names, etc., into a ruleset. Did a server’s IP address change? No problem, just update the macro; no need to mess around with the filter rules that you’ve spent time and energy perfecting for your needs.

A common convention in PF rulesets is to define a macro for each network interface. If a network card ever needs to be replaced with one that uses a different driver, the macro can be updated and the filter rules will function as before. Another benefit is when installing the same ruleset on multiple machines. Certain machines may have different network cards in them, and using macros to define the network interfaces allows the rulesets to be installed with minimal editing. Using macros to define information in a ruleset that is subject to change, such as port numbers, IP addresses, and interface names, is recommended practice.

```conf
# define macros for each network interface
IntIF = "dc0"
ExtIF = "fxp0"
DmzIF = "fxp1"
```

Another common convention is using macros to define IP addresses and network blocks. This can greatly reduce the maintenance of a ruleset when IP addresses change.

```conf
# define our networks
IntNet = "192.168.0.0/24"
ExtAdd = "192.0.2.4"
DmzNet = "10.0.0.0/24"
```

If the internal network ever expanded or was renumbered into a different IP block, the macro can be updated:

```
IntNet = "{ 192.168.0.0/24, 192.168.1.0/24 }"
```

Once the ruleset is reloaded, everything will work as before.

## Using Lists

Let’s look at a good set of rules to have in your ruleset to handle [RFC 1918](https://tools.ietf.org/html/rfc1918)
addresses that just shouldn’t be floating around the Internet, and when they are, are usually trying to cause trouble:

```pf
block in  quick on egress inet from 127.0.0.0/8 to any
block in  quick on egress inet from 192.168.0.0/16 to any
block in  quick on egress inet from 172.16.0.0/12 to any
block in  quick on egress inet from 10.0.0.0/8 to any
block out quick on egress inet from any to 127.0.0.0/8
block out quick on egress inet from any to 192.168.0.0/16
block out quick on egress inet from any to 172.16.0.0/12
block out quick on egress inet from any to 10.0.0.0/8
```

Now look at the following simplification:

```pf
block in quick  on egress inet from { 127.0.0.0/8, 192.168.0.0/16, \
   172.16.0.0/12, 10.0.0.0/8 } to any
block out quick on egress inet from any to { 127.0.0.0/8, \
   192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8 }
```

The ruleset has been reduced from eight lines down to two. Things get even better when macros are used in conjunction with a list:

```pf
NoRouteIPs = "{ 127.0.0.0/8, 192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8 }"

block in  quick on egress from $NoRouteIPs to any
block out quick on egress from any to $NoRouteIPs
```

Note that macros and lists simplify the ‘pf.conf’ file, but the lines are actually expanded by [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
into multiple rules. So, the above example actually expands to the following rules:

```pf
block in  quick on egress inet from 127.0.0.0/8 to any
block in  quick on egress inet from 192.168.0.0/16 to any
block in  quick on egress inet from 172.16.0.0/12 to any
block in  quick on egress inet from 10.0.0.0/8 to any
block out quick on egress inet from any to 10.0.0.0/8
block out quick on egress inet from any to 172.16.0.0/12
block out quick on egress inet from any to 192.168.0.0/16
block out quick on egress inet from any to 127.0.0.0/8
```

As you can see, the PF expansion is purely a convenience for the writer and maintainer of the ‘pf.conf’ file, not an actual simplification of the rules processed by pf.

Macros can be used to define more than just addresses and ports; they can be used anywhere in a PF rules file:

```pf
pre  = "pass in quick on ep0 inet proto tcp from "
post = "to any port { 80, 6667 }"

$pre 198.51.100.80 $post
$pre 203.0.113.79  $post
$pre 203.0.113.178 $post
```

Expands to:

```pf
pass in quick on ep0 inet proto tcp from 198.51.100.80 to any port = 80
pass in quick on ep0 inet proto tcp from 198.51.100.80 to any port = 6667
pass in quick on ep0 inet proto tcp from 203.0.113.79  to any port = 80
pass in quick on ep0 inet proto tcp from 203.0.113.79  to any port = 6667
pass in quick on ep0 inet proto tcp from 203.0.113.178 to any port = 80
pass in quick on ep0 inet proto tcp from 203.0.113.178 to any port = 6667
```

## PF Grammar

PF’s grammar is quite flexible which, in turn, allows for great flexibility in a ruleset. PF is able to infer certain keywords which means that they don’t have to be explicitly stated in a rule, and keyword ordering is relaxed such that it isn’t necessary to memorize strict syntax.

### Elimination of Keywords

To define a “default deny” policy, two rules are used:

```pf
block in  all
block out all
```

This can now be reduced to:

```
block
```

When no direction is specified, PF will assume the rule applies to packets moving in both directions.

Similarly, the ‘from any to any’ and ‘all’ clauses can be left out of a rule, for example:

```pf
block in on rl0 all
pass  in quick log on rl0 proto tcp from any to any port 22
```

can be simplified as:

```pf
block in on rl0
pass  in quick log on rl0 proto tcp to port 22
```

The first rule blocks all incoming packets from anywhere to anywhere on rl0, and the second rule passes in TCP traffic on rl0 to port 22.

### ‘Return’ Simplification

A ruleset used to block packets and reply with a TCP RST or ICMP Unreachable response could look like this:

```pf
block in all
block return-rst  in proto tcp all
block return-icmp in proto udp all
block out all
block return-rst  out proto tcp all
block return-icmp out proto udp all
```

This can be simplified as:

```
block return
```

When PF sees the ‘return’ keyword, it’s smart enough to send the proper response, or no response at all, depending on the protocol of the packet being blocked.

### Keyword Ordering

The order in which keywords are specified is flexible in most cases. For example, a rule written as:

```pf
pass in log quick on rl0 proto tcp to port 22 flags S/SA queue ssh label ssh
```

can also be written as:

```pf
pass in quick log on rl0 proto tcp to port 22 queue ssh label ssh flags S/SA
```

Other, similar variations will also work.
