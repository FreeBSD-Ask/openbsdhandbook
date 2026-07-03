# Redis

## Synopsis

**Redis** is an in-memory key-value store designed for high performance, low latency, and a wide range of data structures such as strings, hashes, sets, sorted sets, and streams. It supports optional disk persistence, replication, and Lua scripting.

Redis is commonly used for caching, messaging, job queues, and ephemeral data storage.

Redis is **not included in the OpenBSD base system**, but is available via packages. The Redis server runs as a daemon (`redis-server`) and is managed with `rcctl(8)`.

## Installation

Install the Redis package:

```sh
# pkg_add redis
```

This installs:

- `redis-server`: the main daemon
- `redis-cli`: command-line interface client
- Default configuration at `/etc/redis.conf`

## Configuration

The main configuration file is:

```text
/etc/redis.conf
```

A minimal configuration example:

```sh
bind 127.0.0.1
port 6379

daemonize no
supervised auto
pidfile /var/run/redis.pid
logfile /var/log/redis.log

dir /var/redis
dbfilename dump.rdb
```

Key options:

- `bind`: restricts listening to loopback
- `port`: default is 6379
- `daemonize`: `no` for `rcctl` compatibility
- `dir`: working directory for data persistence
- `logfile`: optional file logging (otherwise logs to syslog)

Create the data directory if not present:

```sh
# mkdir -p /var/redis
# chown _redis:_redis /var/redis
```

## Enabling the Service

Enable and start Redis:

```sh
# rcctl enable redis
# rcctl start redis
```

Check status:

```sh
# rcctl check redis
```

View logs:

```sh
# tail -f /var/log/redis.log
```

## Basic Usage

Connect using the Redis client:

```sh
$ redis-cli
127.0.0.1:6379> SET keyname "hello"
OK
127.0.0.1:6379> GET keyname
"hello"
```

Exit the client with `CTRL+D` or `QUIT`.

## Persistence

Redis can save the dataset to disk in two ways:

1. **RDB (snapshotting)** — default method using `dump.rdb`
2. **AOF (append-only file)** — logs each write operation

To enable AOF persistence in `/etc/redis.conf`:

```
appendonly yes
appendfilename "appendonly.aof"
```

Restart Redis to apply:

```sh
# rcctl restart redis
```

Backups can be made by copying the `.rdb` or `.aof` files from `/var/redis`.

## Remote Access and Security

By default, Redis binds to `127.0.0.1` only.

To allow access from remote hosts (not recommended unless protected):

1. Change bind address in `/etc/redis.conf`:

```
bind 127.0.0.1 192.0.2.1
```

2. (Optional) Set a password:

```
requirepass "strongsecret"
```

3. Use `pf(4)` to restrict port 6379:

```pf
pass in on $int_if proto tcp from 192.0.2.0/24 to port 6379
```

4. Restart Redis:

```sh
# rcctl restart redis
```

Accessing with authentication:

```sh
$ redis-cli -a strongsecret
```

**Important:** Redis does not support TLS natively in OpenBSD’s package (as of 7.8). To proxy TLS, use `relayd(8)` or `stunnel`.

## Service Management

Standard service commands:

```sh
# rcctl enable redis
# rcctl start redis
# rcctl restart redis
# rcctl check redis
```
