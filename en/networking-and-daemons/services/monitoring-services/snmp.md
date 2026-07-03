# SNMP

## Synopsis

**SNMP (Simple Network Management Protocol)** is a protocol used to monitor and manage devices on a network. It allows external systems to query information such as interface status, uptime, system description, and resource usage.

OpenBSD includes a native SNMP daemon, `snmpd(8)`, in the base system. It provides read-only SNMPv1 and SNMPv2c service suitable for secure local and remote monitoring.

An alternative implementation, **Net-SNMP**, is available via packages and offers SNMPv3, write support, and extensive MIB extension support. For most use cases, `snmpd(8)` is sufficient and easier to secure.

## SNMP Daemon Comparison

| Feature | [snmpd](snmp.md) (base) | Net-SNMP (`pkg_add net-snmp`) |
| --- | --- | --- |
| SNMP Versions | v1, v2c (read-only) | v1, v2c, v3 (read/write, auth) |
| Configuration | `/etc/snmpd.conf` | `/etc/snmp/snmpd.conf` or CLI |
| Daemon | `snmpd(8)` | `snmpd` from Net-SNMP |
| Integration | Native OpenBSD features | More portable, Linux-like |
| SNMPv3 Support | No | Yes |
| Access Control | Limited to community and source | Full view/user control |
| Default Security | Strong defaults, localhost-only | Exposes wide access unless hardened |

Use `snmpd(8)` unless you require SNMPv3, set/walk access, or cross-platform MIB extensions.

## Enabling snmpd(8)

The native SNMP daemon, `snmpd(8)`, is part of the base system and configured via `/etc/snmpd.conf`.

Enable and start the service:

```sh
# rcctl enable snmpd
# rcctl start snmpd
```

The default configuration binds only to `localhost` and allows queries with the community string `public`.

Test locally:

```sh
$ snmpctl walk community public
```

## Basic Configuration

Edit `/etc/snmpd.conf` to define community access and allowed source addresses.

Example configuration exposing basic system information to a private subnet:

```
listen on 192.0.2.1

community "public" source 192.0.2.0/24

system description "OpenBSD Router"
system location "Data Center 1"
system contact "noc@example.org"

sensor temperature
sensor fan
sensor voltage

include pf
include interfaces
```

This:

- Listens on interface IP 192.0.2.1
- Accepts SNMP queries from the `192.0.2.0/24` subnet
- Adds `system.*` values for monitoring software
- Exposes `pf` state table and interface stats
- Enables sensor readings (if supported by `sysctl`)

Reload configuration after editing:

```sh
# rcctl reload snmpd
```

## SNMP Query Examples

Use `snmpctl(8)` to test locally:

```sh
$ snmpctl walk community public
$ snmpctl get community public oid system.description
```

From remote systems (e.g., using Net-SNMP tools):

```sh
$ snmpwalk -v2c -c public 192.0.2.1
$ snmpget -v2c -c public 192.0.2.1 sysUpTime.0
```

To retrieve interface statistics:

```sh
$ snmpwalk -v2c -c public 192.0.2.1 interfaces
```

If `pf` inclusion is configured:

```sh
$ snmpwalk -v2c -c public 192.0.2.1 pf
```

## Security Considerations

SNMPv1 and SNMPv2c use **plaintext community strings**. To limit exposure:

- **Restrict `source` addresses** in `snmpd.conf`
- Bind to specific interfaces using `listen on`
- Use firewall rules to limit incoming UDP/161

Example `pf.conf` rules:

```pf
block in proto udp to port 161
pass in on em0 proto udp from 192.0.2.0/24 to (em0) port 161
```

Avoid exposing SNMP to the Internet. SNMPv3 with authentication is only supported by Net-SNMP.

## Logging

All access events and errors are logged via `syslog`:

```sh
# tail -f /var/log/daemon
```

Log level is fixed; verbose or debug output is not available via `rcctl`.

## System Integration

`sensorsd(8)` integrates with `snmpd` to expose system temperature, voltage, and fan data.

Enable `sensorsd`:

```sh
# rcctl enable sensorsd
# rcctl start sensorsd
```

These metrics are automatically exposed if `sensor` is specified in `snmpd.conf`.

For interface and firewall statistics, the directives `include interfaces` and `include pf` respectively pull data from `ifconfig` and `pfctl`.

## Net-SNMP (Optional)

If SNMPv3, writable access, or MIB support is required:

```sh
# pkg_add net-snmp
```

Configuration for Net-SNMP is more complex and uses its own syntax. For SNMPv3:

```sh
createUser readonly MD5 "password" DES
rouser readonly
```

For full control, refer to `/usr/local/share/examples/net-snmp/snmpd.conf`.
