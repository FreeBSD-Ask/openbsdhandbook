# PF Tables

## Introduction

A table is used to hold a group of IPv4 and/or IPv6 addresses. Lookups against a table are very fast and consume less memory and processor time than [lists](lists-and-macros.md#lists)
. For this reason, a table is ideal for holding a large group of addresses as the lookup time on a table holding 50,000 addresses is only slightly more than for one holding 50 addresses. Tables can be used in the following ways:

- Source and/or destination address in rules.
- Translation and redirection addresses ’nat-to’ and ‘rdr-to’ rule options, respectively.
- Destination address in ‘route-to’, ‘reply-to’ and ‘dup-to’ rule options.

Tables are created either in /etc/pf.conf or by using pfctl.

## Configuration

In ‘pf.conf’, tables are created using the ’table’ directive. The following attributes may be specified for each table:

- ‘const’ - the contents of the table cannot be changed once the table is created. When this attribute is not specified, pfctl may be used to add or remove addresses from the table at any time, even when running with a securelevel of two or greater.
- ‘persist’ - causes the kernel to keep the table in memory even when no rules refer to it. Without this attribute, the kernel will automatically remove the table when the last rule referencing it is flushed.

Example:

```pf
table <goodguys> { 192.0.2.0/24 }
table <rfc1918>  const { 192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8 }
table <spammers> persist
block in on fxp0 from { <rfc1918>, <spammers> } to any
pass  in on fxp0 from <goodguys> to any
```

Addresses can also be specified using the negation (or “not”) modifier, such as:

```pf
table <goodguys> { 192.0.2.0/24, !192.0.2.5 }
```

The ‘goodguys’ table will now match all addresses in the 192.0.2.0/24 network except for 192.0.2.5.

Note that table names are always enclosed in < > angled brackets.

Tables can also be populated from text files containing a list of IP addresses and networks:

```pf
table <spammers> persist file "/etc/spammers"
block in on fxp0 from <spammers> to any
```

The file ‘/etc/spammers’ would contain a list of IP addresses and/or [CIDR](https://web.archive.org/web/20150213012421/http://public.swbell.net/dedicated/cidr.html)
network blocks, one per line.

## Manipulating with ‘pfctl’

Tables can be manipulated on the fly by using pfctl. For instance, to add entries to the table created above:

```sh
# **pfctl -t spammers -T add 203.0.113.0/24**
```

This will also create the table if it doesn’t already exist. To list the addresses in a table:

```sh
# **pfctl -t spammers -T show**
```

The ‘-v’ argument can also be used with ‘-T show’ to display statistics for each table entry. To remove addresses from a table:

```sh
# **pfctl -t spammers -T delete 203.0.113.0/24**
```

For more information on manipulating tables with ‘pfctl’, please read the [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
man page.

## Specifying Addresses

In addition to being specified by IP address, hosts may also be specified by their hostname. When the hostname is resolved to an IP address, all resulting IPv4 and IPv6 addresses are placed into the table. IP addresses can also be entered into a table by specifying a valid interface name, interface group, or the ‘self’ keyword. The table will then contain all IP addresses assigned to that interface or group, or to the machine (including loopback addresses), respectively.

One limitation when specifying addresses is that ‘0.0.0.0/0’ and ‘0/0’ will not work in tables. The alternative is to hard code that address or use a [macro](lists-and-macros.md#macros)
.

## Address Matching

An address lookup against a table will return the most narrowly matching entry. This allows for the creation of tables such as:

```pf
table <goodguys> { 172.16.0.0/16, !172.16.1.0/24, 172.16.1.100 }

block in on dc0
pass  in on dc0 from <goodguys>
```

Any packet coming in through ‘dc0’ will have its source address matched against the table ‘’:

- 172.16.50.5 - narrowest match is 172.16.0.0/16; packet matches the table and will be passed
- 172.16.1.25 - narrowest match is !172.16.1.0/24; packet matches an entry in the table but that entry is negated (uses the “!” modifier); packet does not match the table and will be blocked
- 172.16.1.100 - exactly matches 172.16.1.100; packet matches the table and will be passed
- 10.1.4.55 - does not match the table and will be blocked
