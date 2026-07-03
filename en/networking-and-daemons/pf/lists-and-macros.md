# PF Lists and Macros

## Lists

A list allows the specification of multiple similar criteria within a rule. For example, multiple protocols, port numbers, addresses, etc. So, instead of writing one filter rule for each IP address that needs to be blocked, one rule can be written by specifying the IP addresses in a list. Lists are defined by specifying items within ‘{ }’ brackets.

When pfctl parses the configuration files and encounters a list during loading of the ruleset, it creates multiple rules, one for each item in the list. For example:

```pf
block out on fxp0 from { 192.168.0.1, 10.5.32.6 } to any
```

gets expanded to:

```pf
block out on fxp0 from 192.168.0.1 to any
block out on fxp0 from 10.5.32.6 to any
```

Multiple lists can be specified within a rule:

```pf
match in on fxp0 proto tcp to port { 22 80 } rdr-to 192.168.0.6
block out on fxp0 proto { tcp udp } from { 192.168.0.1, 10.5.32.6 } \
   to any port { ssh https }
```

The commas between list items are optional.

Lists can also contain nested lists:

```pf
trusted = "{ 192.168.1.2 192.168.5.36 }"
pass in inet proto tcp from { 10.10.0.0/24 $trusted } to port 22
```

Beware of constructs like the following, dubbed “negated lists,” which are a common mistake:

```pf
pass in on fxp0 from { 10.0.0.0/8, !10.1.2.3 }
```

While the intended meaning is usually to match “any address within 10.0.0.0/8, except for 10.1.2.3,” the rule expands to:

```pf
pass in on fxp0 from 10.0.0.0/8
pass in on fxp0 from !10.1.2.3
```

which matches any possible address. Instead, a [table](tables.md)
should be used.

## Macros

Macros are user-defined variables that can hold IP addresses, port numbers, interface names, etc. Macros can reduce the complexity of a PF ruleset and also make maintaining a ruleset much easier.

Macro names must start with a letter and may contain letters, digits and underscores. Macro names cannot be reserved words such as ‘pass’, ‘out’ or ‘queue’.

```pf
ext_if = "fxp0"

block in on $ext_if from any to any
```

This creates a macro named ’ext\_if’. When a macro is referred to after it’s been created, its name is preceded with a ‘$’ character.

Macros can also expand to lists, such as:

```
friends = "{ 192.168.1.1, 10.0.2.5, 192.168.43.53 }"
```

Macros can be defined recursively. Since macros are not expanded within quotes the following syntax must be used:

```
host1      = "192.168.1.1"
host2      = "192.168.1.2"
all_hosts  = "{" $host1 $host2 "}"
```

The macro ‘$all\_hosts’ now expands to 192.168.1.1, 192.168.1.2.
