# BIND

## Synopsis

BIND (Berkeley Internet Name Domain) is a full-featured DNS server implementation maintained by ISC. Unlike `unbound(8)` or `nsd(8)`, which each serve one DNS function (resolver or authoritative server), `named(8)` from BIND can act as both. BIND is suitable for advanced use cases that require flexible DNS views, dynamic updates, or complex zone configurations. However, due to its complexity and historical security issues, it is not included in the OpenBSD base system and must be installed from packages.

This chapter explains how to install, configure, and operate BIND on OpenBSD 7.8, including authoritative and recursive modes.

## DNS Services Comparison

To clarify BIND’s position among DNS software available on OpenBSD, consider the following table:

| Feature | unbound(8) | nsd(8) | named (BIND) |
| --- | --- | --- | --- |
| Purpose | Resolver | Authoritative | Both |
| Included in OpenBSD base | Yes | Yes | No |
| Recursive resolution | Yes | No | Yes |
| Authoritative server | No | Yes | Yes |
| DNSSEC validation | Yes | No (serve only) | Yes |
| DNS-over-TLS support | Yes | No | Yes |
| Complexity | Low | Low | High |
| Use case | Clients | DNS hosting | Complex/mixed DNS setups |

## Installation

BIND is not part of the OpenBSD base system. Install it using the package system:

```sh
# pkg_add isc-bind
```

This installs `named(8)` and its configuration tools under `/etc/named/` and `/var/named/`.

To enable BIND at boot:

```sh
# rcctl enable isc_named
```

To start the service immediately:

```sh
# rcctl start isc_named
```

## Configuration

The main configuration file is:

```sh
/etc/named/named.conf
```

By default, BIND uses `/var/named` as its working directory and stores zone files and runtime data there.

### Example Recursive Resolver

To configure BIND as a caching recursive resolver:

```
options {
    directory "/var/named";
    allow-query { any; };
    recursion yes;
    allow-recursion { any; };
    listen-on { 127.0.0.1; };
    forwarders {
        8.8.8.8;
        1.1.1.1;
    };
};
```

After saving the configuration:

```sh
# named-checkconf /etc/named/named.conf
# rcctl restart isc_named
```

Use `drill` or `dig` to verify:

```sh
$ drill @127.0.0.1 openbsd.org
```

### Example Authoritative Server

To configure BIND to serve a zone for `example.com`:

```
zone "example.com" {
    type master;
    file "example.com.zone";
};
```

Create the zone file:

```
$ORIGIN example.com.
$TTL 3600
@   IN  SOA ns1.example.com. hostmaster.example.com. (
        2025080401 ; serial
        3600       ; refresh
        1800       ; retry
        604800     ; expire
        86400 )    ; minimum
    IN  NS  ns1.example.com.
    IN  NS  ns2.example.com.
ns1 IN  A   192.0.2.1
ns2 IN  A   192.0.2.2
www IN  A   192.0.2.100
```

Store this file as `/var/named/example.com.zone`.

Check syntax and reload:

```sh
# named-checkzone example.com /var/named/example.com.zone
# rcctl restart isc_named
```

### Logging

Logging must be explicitly enabled:

```
logging {
    channel default_log {
        file "/var/named/named.log";
        severity info;
        print-time yes;
    };
    category default { default_log; };
};
```

Ensure the directory and file are writable:

```sh
# touch /var/named/named.log
# chown _named:_named /var/named/named.log
```

## DNSSEC Support

BIND supports DNSSEC signing and validation. For recursive resolvers, validation is enabled by:

```
options {
    dnssec-validation auto;
};
```

To serve signed zones, use `dnssec-keygen` and `dnssec-signzone`:

```sh
# dnssec-keygen -a ECDSAP256SHA256 -n ZONE example.com
# dnssec-signzone -o example.com example.com.zone Kexample.com.*
```

Update the zone file and reload as usual.

## Runtime Control

BIND uses `rndc(8)` for control:

```sh
# rndc reload
# rndc flush
# rndc status
```

To enable `rndc`, generate keys and add control configuration:

```
include "/etc/named/rndc.key";

controls {
    inet 127.0.0.1 allow { localhost; } keys { "rndc-key"; };
};
```

Run:

```sh
# rndc-confgen -a
```

This creates `/etc/named/rndc.key`.

## Example Usage

### As a Local Resolver

To use BIND locally:

1. Enable and configure with `recursion yes`
2. Set `/etc/resolv.conf`:

```
nameserver 127.0.0.1
```

### As Authoritative Server

To serve a zone:

1. Create `zone` section in `named.conf`
2. Create the zone file in `/var/named`
3. Test and reload configuration

Use `drill` to query:

```sh
$ drill @192.0.2.1 www.example.com
```

## Files and Directories

| Path | Purpose |
| --- | --- |
| `/etc/named/named.conf` | BIND configuration |
| `/var/named/` | Zone and runtime files |
| `/etc/named/rndc.key` | Key for `rndc` control |
| `/var/named/named.log` | Optional logging output |

## Security Considerations

Due to its size and complexity, BIND has historically had more security vulnerabilities than minimal alternatives. It should be run in a chroot or jailed environment (OpenBSD does this via `chroot(2)` automatically). Prefer `unbound(8)` or `nsd(8)` when functionality permits.
