# memcached

## Synopsis

**memcached** is a high-performance in-memory key-value store used primarily for caching data to reduce database load and latency in dynamic web applications. It provides a simple TCP protocol for setting and retrieving short-lived values and is often used by frameworks like Django, Rails, and PHP.

memcached is available on OpenBSD via packages and integrates with `rcctl(8)` for service management.

Unlike Redis, memcached is not persistent and supports only very simple key-value operations.

## Installation

Install memcached using the package system:

```sh
# pkg_add memcached
```

This installs:

- `/usr/local/bin/memcached` — the daemon
- `/etc/rc.d/memcached` — `rcctl` control script

## Configuration

memcached is typically configured via command-line options. These options can be set persistently by editing the `rcctl` flags:

```sh
# rcctl set memcached flags "-m 64 -l 127.0.0.1 -p 11211"
```

Common options:

- `-m 64` — allocate 64 MB of RAM
- `-l 127.0.0.1` — bind only to localhost
- `-p 11211` — default TCP port

To apply changes:

```sh
# rcctl enable memcached
# rcctl start memcached
```

## Testing Basic Usage

Use the `telnet(1)` client to test memcached:

```sh
$ telnet localhost 11211
Trying 127.0.0.1...
Connected to localhost.
set testkey 0 300 5
hello
STORED
get testkey
VALUE testkey 0 5
hello
END
```

- `set <key> <flags> <ttl> <bytes>`
- `get <key>`

Keys expire after the specified TTL in seconds.

To quit:

```
quit
```

## Using with Clients

memcached is supported by many programming languages. Examples:

- PHP: `memcached` or `memcache` extension
- Python: `python3-memcached` or `pymemcache`
- Ruby: `dalli` gem

Example Python usage:

```python
import memcache
mc = memcache.Client(['127.0.0.1:11211'])
mc.set('greeting', 'hello', time=300)
print(mc.get('greeting'))
```

## Security Considerations

memcached does **not support authentication or TLS**. It must be run on trusted networks only.

By default, it should be bound to `127.0.0.1`. If remote access is necessary:

1. Adjust flags:

```sh
# rcctl set memcached flags "-m 128 -l 192.0.2.1 -p 11211"
```

2. Restrict access using `pf(4)`:

```pf
pass in on $int_if proto tcp from 192.0.2.0/24 to port 11211
block in on $int_if proto tcp from any to port 11211
```

3. Reload `pf` and restart memcached:

```sh
# pfctl -f /etc/pf.conf
# rcctl restart memcached
```

Never expose memcached to the public Internet.

## Service Management

To start and enable at boot:

```sh
# rcctl enable memcached
# rcctl start memcached
```

To stop or restart:

```sh
# rcctl stop memcached
# rcctl restart memcached
```

Check status:

```sh
# rcctl check memcached
```

## Logging

memcached logs to syslog. View logs with:

```sh
# tail -f /var/log/daemon
```

To increase verbosity, add `-v` or `-vv` to flags:

```sh
# rcctl set memcached flags "-m 64 -v"
# rcctl restart memcached
```
