# NSD

## Synopsis

`nsd(8)` is a high-performance authoritative-only DNS server developed by NLnet Labs. It is included in the OpenBSD base system and is designed to serve DNS zones securely and efficiently without supporting recursive queries. Unlike `unbound(8)`, which performs DNS resolution for clients, `nsd(8)` answers queries about domains it is explicitly configured to serve.

This chapter describes the configuration and management of `nsd(8)` on OpenBSD for serving DNS zones to the public or within internal networks.

## DNS Services Comparison

To understand where `nsd(8)` fits into the DNS ecosystem on OpenBSD, consider the following comparison:

| Feature | unbound(8) | nsd(8) | named(8) (BIND) |
| --- | --- | --- | --- |
| Purpose | Resolver | Authoritative | Both |
| Included in OpenBSD base | Yes | Yes | No |
| Recursive resolution | Yes | No | Yes |
| Authoritative server | No | Yes | Yes |
| DNSSEC validation | Yes | No (serves DNSSEC data) | Yes |
| DNS-over-TLS support | Yes | No | Yes |
| Complexity | Low | Low | High |
| Use case | Client systems | Hosting zones | Mixed environments |

## Installation

`nsd(8)` is included in the OpenBSD 7.8 base system. No external packages are required.

Enable `nsd` at boot:

```sh
# rcctl enable nsd
```

Start the daemon:

```sh
# rcctl start nsd
```

The main configuration directory is `/var/nsd`. Configuration files are stored in:

```sh
/var/nsd/etc/nsd.conf
```

Zone files are typically stored in:

```sh
/var/nsd/zones/
```

## Configuration

### Base Configuration File

An example minimal configuration for a domain `example.com` is shown below:

```
server:
    hide-version: yes
    ip-address: 192.0.2.1

zone:
    name: "example.com"
    zonefile: "example.com.zone"
```

The `server:` section defines daemon-wide options. The `zone:` section defines each domain to serve.

Once this file is created at `/var/nsd/etc/nsd.conf`, test it:

```sh
# nsd-checkconf /var/nsd/etc/nsd.conf
```

### Zone File Format

Zone files use standard BIND-style syntax:

```
$ORIGIN example.com.
$TTL 3600
@   IN  SOA ns1.example.com. hostmaster.example.com. (
        2025080401 ; serial
        3600       ; refresh
        1800       ; retry
        604800     ; expire
        86400 )    ; minimum TTL
    IN  NS  ns1.example.com.
    IN  NS  ns2.example.com.
ns1 IN  A   192.0.2.1
ns2 IN  A   192.0.2.2
www IN  A   192.0.2.100
```

Store this file as `/var/nsd/zones/example.com.zone`.

### Permissions

Ensure ownership and permissions are correct:

```sh
# chown -R _nsd:_nsd /var/nsd
# chmod -R 755 /var/nsd
```

## Zone Compilation and Reload

Before `nsd(8)` can serve zones, it must compile them into a binary database:

```sh
# nsd-control rebuild
```

This command reads the configuration and compiles the zone data. Then reload:

```sh
# nsd-control reload
```

To check operational status:

```sh
# nsd-control status
```

## DNSSEC with NSD

`nsd(8)` does not validate DNSSEC but can serve signed zone data. Signing must be performed externally (e.g., with `ldns-signzone`).

Example for signing a zone file:

```sh
# ldns-signzone example.com.zone Kexample.com.+008+12345.key Kexample.com.+008+12345.private
```

Update `nsd.conf` to reference the signed zone file, and rebuild/reload as usual.

## Controlling NSD

Use `nsd-control(8)` for runtime control:

```sh
# nsd-control status
# nsd-control reload
# nsd-control addzone example.com /var/nsd/etc/nsd.conf
# nsd-control delzone example.com
```

The `nsd-control` utility requires a control key and certificate, created with:

```sh
# nsd-control-setup
```

This generates `/var/nsd/etc/nsd_control.key` and `.pem` files.

## Logging and Debugging

Logging is sent to `syslog(3)` under the `daemon` facility. To view logs:

```sh
# tail -f /var/log/daemon
```

Increase verbosity for debugging:

```
server:
    verbosity: 2
```

Then restart the service:

```sh
# rcctl restart nsd
```

## Example Usage

To serve the domain `example.com`:

1. Place zone file at `/var/nsd/zones/example.com.zone`
2. Add configuration to `/var/nsd/etc/nsd.conf`
3. Run:

```sh
# nsd-checkconf
# nsd-control rebuild
# rcctl restart nsd
```

To verify:

```sh
$ drill @192.0.2.1 www.example.com
```

## Summary of Key Files and Directories

| File/Directory | Description |
| --- | --- |
| `/var/nsd/etc/nsd.conf` | NSD configuration file |
| `/var/nsd/zones/` | Zone data files |
| `/var/nsd/db/nsd.db` | Compiled binary zone database |
| `/var/nsd/run/nsd.pid` | PID file for nsd(8) |
| `/var/nsd/etc/nsd_control.*` | nsd-control TLS keys and certs |
| `/var/log/daemon` | Log output via syslog |
