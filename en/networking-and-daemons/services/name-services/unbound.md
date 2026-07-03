# Unbound

## Synopsis

`unbound(8)` is a validating, recursive, caching DNS resolver included in the OpenBSD base system. It is enabled by default for local name resolution and is well-suited for both workstations and server environments. Unbound prioritizes simplicity, security, and performance, supporting features such as DNSSEC validation, local zones, and DNS-over-TLS.

This chapter describes how to configure and manage `unbound(8)` on OpenBSD, including advanced options for DNS privacy and local network integration. A comparison with alternative DNS services is provided to clarify the differences between resolver and authoritative roles.

## DNS Services Comparison

The table below highlights the key differences between the three DNS services typically used on OpenBSD: `unbound(8)`, `nsd(8)`, and BIND (`named(8)`).

| Feature | unbound(8) | nsd(8) | named(8) (BIND) |
| --- | --- | --- | --- |
| Purpose | Resolver | Authoritative | Both |
| Included in OpenBSD base | Yes | Yes | No |
| Recursive resolution | Yes | No | Yes |
| Authoritative server | No | Yes | Yes |
| DNSSEC validation | Yes | No | Yes |
| DNS-over-TLS support | Yes | No | Yes |
| Complexity | Low | Low | High |
| Use case | Client systems | Hosting zones | Mixed environments |

Use `unbound(8)` when secure and private DNS resolution is required. Use `nsd(8)` to serve DNS zones. Use BIND (`named`) only when both roles or advanced DNS features are required beyond what `unbound` and `nsd` provide.

## Enabling Unbound

Unbound is pre-installed on OpenBSD and enabled by default for local resolution.

To verify its status:

```sh
# rcctl check unbound
```

To enable it explicitly:

```sh
# rcctl enable unbound
# rcctl start unbound
```

Unbound’s configuration is located at:

```sh
/etc/unbound/unbound.conf
```

## Default Configuration

The default configuration provides DNS resolution via localhost, with caching and DNSSEC validation enabled.

Check `/etc/resolv.conf`:

```sh
nameserver 127.0.0.1
lookup file bind
```

This directs the system resolver to use Unbound running on the loopback interface.

## Configuration

The default `/var/unbound/etc/unbound.conf` is auto-generated. Custom configuration may be placed in `/etc/unbound/unbound.conf` or in `/var/unbound/etc/unbound.conf` if the base configuration is disabled.

To override and use a static configuration:

1. Disable automatic configuration generation:

```sh
# rcctl disable resolvd
```

2. Create a configuration file:

```
server:
    verbosity: 1
    interface: 127.0.0.1
    access-control: 127.0.0.0/8 allow
    cache-max-ttl: 86400
    cache-min-ttl: 3600
    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    auto-trust-anchor-file: "/var/unbound/db/root.key"
    prefetch: yes

forward-zone:
    name: "."
    forward-tls-upstream: yes
    forward-addr: 1.1.1.1@853       # Cloudflare DNS over TLS
    forward-addr: 9.9.9.9@853       # Quad9 DNS over TLS
```

Then restart Unbound:

```sh
# rcctl restart unbound
```

## DNS over TLS

To enable secure DNS queries with DNS-over-TLS, use the `forward-zone` block and specify upstream servers with TLS ports (`@853`). Unbound must be compiled with TLS support (OpenBSD’s base version includes it).

Verify resolution and DNSSEC validation:

```sh
$ drill openbsd.org
$ drill -D openbsd.org
```

`-D` shows DNSSEC status.

## Local Zone and Host Overrides

Static host mappings can be added using `local-data` or `local-zone`:

```
local-zone: "example.test." static
local-data: "host1.example.test. IN A 192.168.1.10"
local-data: "host2.example.test. IN AAAA fd00::1"
```

This is useful for overriding names internally without relying on external DNS.

## Using DNSSEC

DNSSEC is enabled by default. Trust anchors are managed via:

```sh
/var/unbound/db/root.key
```

To refresh the trust anchor:

```sh
# unbound-anchor -a /var/unbound/db/root.key
```

DNSSEC validation failures will result in SERVFAIL responses to prevent spoofing.

## Integration with dhclient

By default, `dhclient(8)` integrates with `resolvd(8)` which may overwrite `/etc/resolv.conf`.

To avoid conflicts:

1. Disable `resolvd`:

```sh
# rcctl disable resolvd
```

2. Set `resolv.conf` manually:

```sh
# echo 'nameserver 127.0.0.1' > /etc/resolv.conf
```

3. Prevent `dhclient` from overwriting it:

```sh
# chflags schg /etc/resolv.conf
```

To reverse:

```sh
# chflags noschg /etc/resolv.conf
```

## Monitoring and Debugging

Basic status:

```sh
# unbound-control status
```

To query statistics:

```sh
# unbound-control stats
```

To flush cache:

```sh
# unbound-control flush
```

To trace a resolution:

```sh
# unbound-control lookup openbsd.org
```

Logs are written to `/var/log/messages`. Increase verbosity in configuration to debug:

```
server:
    verbosity: 3
```

Then restart:

```sh
# rcctl restart unbound
```

## Example Usage

To test local resolution:

```sh
$ doas drill openbsd.org
    # Resolve openbsd.org using the local resolver
$ doas unbound-control status
    # Show current status including uptime, version, and statistics
```

## Summary of Key Files and Directories

| File/Directory | Description |
| --- | --- |
| `/etc/unbound/unbound.conf` | Optional local configuration |
| `/var/unbound/etc/unbound.conf` | Active configuration (default) |
| `/var/unbound/db/root.key` | DNSSEC root trust anchor |
| `/etc/resolv.conf` | System resolver configuration |
| `/var/log/messages` | Default location for unbound logging |
