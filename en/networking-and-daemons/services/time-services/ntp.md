# NTP

## Synopsis

Accurate system time is essential for logging, cryptographic protocols, file timestamps, and consistent operation of distributed systems. OpenBSD includes a secure, minimalist NTP daemon, `ntpd(8)`, in the base system. This service allows the system clock to be synchronized with remote time servers and, optionally, to provide time synchronization to other systems.

This chapter describes how to configure `ntpd(8)` for typical client use and for serving time to a local network.

## Default Behavior

The NTP daemon is enabled by default in OpenBSD. During system startup, it performs a one-time time step using `ntpd -s` if the system clock is significantly skewed, and then continues background adjustments gradually.

The default configuration uses the public NTP pool:

```
servers pool.ntp.org
```

This default is sufficient for most systems and does not require modification unless specific servers are preferred.

## Configuration File

The configuration file for `ntpd` is located at `/etc/ntpd.conf`. It consists of plain text directives. The most common are:

- `servers <address>` — Use a pool of time servers.
- `server <address>` — Use a single, specified time server.
- `listen on <interface>` — Enable NTP server functionality by listening on the specified interface or address.

### Example: Client-Only Configuration

```
servers pool.ntp.org
```

This configuration uses multiple geographically distributed servers selected from the NTP pool.

### Example: Client and Server Configuration

```
servers pool.ntp.org
listen on egress
```

This example synchronizes time from external servers and serves NTP to local clients over the system’s `egress` interface.

## Enabling and Starting ntpd

To verify that the NTP service is enabled and running:

```sh
# rcctl check ntpd
```

To manually enable and start the service:

```sh
# rcctl enable ntpd
# rcctl start ntpd
```

## Inspecting Synchronization Status

Use `ntpctl(8)` to query the status of the daemon and its peers:

```sh
$ ntpctl -s status
$ ntpctl -s peers
```

These commands report information such as synchronization state, stratum level, offset, and network delay.

## Manual Clock Initialization

In situations where the system clock is significantly incorrect—such as after a power outage or on systems without a real-time clock—`ntpd` can be run in one-time initialization mode:

```sh
# ntpd -s
```

This steps the clock immediately. It is run automatically at boot by default.

To run in foreground for debugging:

```sh
# ntpd -s -d
```

## Logging and Troubleshooting

`ntpd` logs activity to `/var/log/messages`. To monitor this output in real time:

```sh
# tail -f /var/log/messages
```

Ensure that outbound UDP port 123 is permitted through any packet filtering rules. If time synchronization fails:

- Verify that DNS resolution is working.
- Use `ntpctl -s peers` to check reachability and response from peers.
- Confirm that no other time services (such as `rdate`) are active simultaneously.

## Serving Time to Other Systems

To allow other systems on a local network to synchronize against the OpenBSD host, configure `ntpd` with a `listen on` directive:

```
servers pool.ntp.org
listen on em0
```

This configuration queries external NTP servers while listening for requests on interface `em0`.

This approach is useful in environments where multiple systems must synchronize with a common internal time source or where external network access is restricted.

## Limitations and Behavior

OpenBSD’s `ntpd` focuses on simplicity and security. It intentionally omits cryptographic authentication features found in some other NTP implementations. Support for leap seconds is automatic and does not require additional configuration.

`ntpd` is suitable for most client and internal server use cases, but not for highly specialized or authenticated NTP deployments.
