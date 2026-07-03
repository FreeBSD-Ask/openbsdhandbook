# PF Anchors

## Introduction

In addition to the main ruleset, PF can also evaluate sub-rulesets. Since sub-rulesets can be manipulated on the fly by using pfctl, they provide a convenient way of dynamically altering an active ruleset. Whereas a [table](tables.md)
is used to hold a dynamic list of addresses, a sub-ruleset is used to hold a dynamic set of rules. A sub-ruleset is attached to the main ruleset by using an ‘anchor’.

Anchors can be nested which allows for sub-rulesets to be chained together. Anchor rules will be evaluated relative to the anchor in which they are loaded. For example, anchor rules in the main ruleset will create anchor attachment points with the main ruleset as their parent, and anchor rules loaded from files with the ’load anchor’ directive will create anchor points with that anchor as their parent.

## Anchors

An anchor is a collection of rules, tables and other anchors that has been assigned a name. When PF comes across an ‘anchor’ rule in the main ruleset, it will evaluate the rules contained within the anchor point as it evaluates rules in the main ruleset. Processing will then continue in the main ruleset unless the packet matches a filter rule that uses the ‘quick’ option, in which case the match will be considered final and will abort the evaluation of rules in both the anchor and the main rulesets.

For example:

```pf
block on     egress
pass  out on egress

anchor goodguys
```

This ruleset sets a default deny policy on the egress interface for incoming and outgoing traffic, which is then statefully passed out, and an anchor rule is created named ‘goodguys’. Anchors can be populated with rules by three methods:

- using a ’load’ rule
- using pfctl
- specifying the rules inline of the main ruleset

The ’load’ rule causes ‘pfctl’ to populate the specified anchor by reading rules from a text file. The ’load’ rule must be placed after the ‘anchor’ rule.

For example:

```pf
anchor goodguys
load anchor goodguys from "/etc/anchor-goodguys-ssh"
```

To add rules to an anchor using ‘pfctl’, the following type of command can be used:

```sh
# echo "pass in proto tcp from 192.0.2.3 to any port 22" | pfctl -a goodguys -f -
```

Rules can also be saved to (and loaded from) a text file. For example, you could append the following two lines to the ‘/etc/anchor-goodguys-www’ file:

```pf
pass in proto tcp from 192.0.2.3 to any port 80
pass in proto tcp from 192.0.2.4 to any port { 80 443 }
```

Then applied with:

```sh
# pfctl -a goodguys -f /etc/anchor-goodguys-www
```

To load rules directly from the main ruleset, enclose the anchor rules in a brace-delimited block:

```pf
anchor "goodguys" {
        pass in proto tcp from 192.168.2.3 to port 22
}
```

Inline anchors can also contain more anchors.

```pf
allow = "{ 192.0.2.3 192.0.2.4 }"

anchor "goodguys" {
  anchor {
       pass in proto tcp from 192.0.2.3 to port 80
  }
  pass in proto tcp from $allow to port 22
}
```

With inline anchors, the name of the anchor becomes optional. Note how the nested anchor in the above example does not have a name. The macro ‘$allow’ is created outside of the anchor (in the main ruleset) and is then used within the anchor.

Rules can be loaded into an anchor using the same syntax and options as rules loaded into the main ruleset. One caveat is that, unless you’re using inline anchors, any [macros](lists-and-macros.md#macros)
that are used must also be defined within the anchor itself; macros that are defined in the parent ruleset are *not* visible from the anchor.

Since anchors can be nested, it’s possible to specify that all child anchors within a specified anchor be evaluated:

```pf
anchor "spam/*"
```

This syntax causes each rule within each anchor attached to the ‘spam’ anchor to be evaluated. The child anchors will be evaluated in alphabetical order but are not descended into recursively. Anchors are always evaluated relative to the anchor in which they’re defined.

Each anchor, as well as the main ruleset, exist separately from the other rulesets. Operations done on one ruleset, such as flushing the rules, do not affect any of the others. In addition, removing an anchor point from the main ruleset does not destroy the anchor or any child anchors that are attached to that anchor. An anchor is not destroyed until it’s flushed of all rules using pfctl and there are no child anchors within the anchor.

## Anchor Options

Optionally, ‘anchor’ rules can specify interface, protocol, source and destination address, tag, etc, using the same syntax as other rules. When such information is given, ‘anchor’ rules are only processed if the packet matches the ‘anchor’ rule’s definition.

For example:

```pf
block          on egress
pass       out on egress
anchor ssh in  on egress proto tcp to port 22
```

The rules in the anchor ‘ssh’ are only evaluated for TCP packets destined for port 22 that come in on the egress interface. Rules are then added to the ‘anchor’ like so:

```sh
# echo "pass in from 192.0.2.10 to any" | pfctl -a ssh -f -
```

So, even though the filter rule doesn’t specify an interface, protocol or port, the host 192.0.2.10 will only be permitted to connect using SSH because of the ‘anchor’ rule’s definition.

The same syntax can be applied to inline anchors.

```pf
allow = "{ 192.0.2.3 192.0.2.4 }"
anchor "goodguys" in proto tcp {
   anchor proto tcp to port 80 {
      pass from 192.0.2.3
   }
   anchor proto tcp to port 22 {
      pass from $allow
   }
}
```

## Manipulating Anchors

Manipulation of anchors is performed via pfctl. It can be used to add and remove rules from an anchor without reloading the main ruleset.

To list all the rules in the anchor named ‘ssh’:

```sh
# pfctl -a ssh -s rules
```

To flush all rules from the same anchor:

```sh
# pfctl -a ssh -F rules
```

For a full list of commands, please see [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
.
