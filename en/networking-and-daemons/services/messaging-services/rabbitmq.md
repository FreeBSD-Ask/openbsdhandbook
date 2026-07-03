# RabbitMQ

## Synopsis

**RabbitMQ** is a message broker that implements the AMQP (Advanced Message Queuing Protocol). It is used to decouple services and distribute tasks using queues, topics, and publish-subscribe mechanisms.

RabbitMQ supports persistent queues, acknowledgements, clustering, TLS encryption, and various plugins including web-based management.

RabbitMQ is available on OpenBSD via the packages system. It runs on top of the **Erlang runtime** and communicates over TCP (AMQP by default on port 5672, and optionally HTTPS on port 15671 for management).

## Installation

Install both RabbitMQ and its Erlang runtime dependencies:

```sh
# pkg_add rabbitmq
```

This installs:

- `rabbitmq-server`: the main daemon
- `rabbitmqctl`: administrative CLI
- `rabbitmq-plugins`: plugin management utility
- Default config directory: `/etc/rabbitmq/`

RabbitMQ will refuse to run without a writable data directory.

## Service Initialization

The RabbitMQ service must be initialized before starting:

```sh
# install -d -o _rabbitmq -g _rabbitmq /var/rabbitmq
```

Enable and start the service:

```sh
# rcctl enable rabbitmq
# rcctl start rabbitmq
```

Check status:

```sh
# rabbitmqctl status
```

By default, the daemon listens on:

- TCP port **5672** (AMQP)
- TCP port **25672** (Erlang distribution, not used unless clustered)

## Configuration

RabbitMQ uses a directory-based config under `/etc/rabbitmq/`.

There are two main files:

- `/etc/rabbitmq/rabbitmq-env.conf`: shell environment overrides
- `/etc/rabbitmq/rabbitmq.conf`: main configuration file (in INI-style format)

Example `/etc/rabbitmq/rabbitmq.conf`:

```conf
# Disable loopback restriction
loopback_users.guest = false

# Default listener
listeners.tcp.default = 127.0.0.1:5672

# Enable TLS
# listeners.ssl.default = 0.0.0.0:5671
# ssl_options.certfile = /etc/ssl/rabbitmq/server.crt
# ssl_options.keyfile = /etc/ssl/rabbitmq/server.key
# ssl_options.cacertfile = /etc/ssl/rabbitmq/ca.crt
```

Restart RabbitMQ after configuration changes:

```sh
# rcctl restart rabbitmq
```

## User and Access Control

To create a new user and virtual host:

```sh
# rabbitmqctl add_user myuser secretpass
# rabbitmqctl add_vhost myvhost
# rabbitmqctl set_permissions -p myvhost myuser ".*" ".*" ".*"
```

Disable the default `guest` user for security:

```sh
# rabbitmqctl delete_user guest
```

List users and vhosts:

```sh
# rabbitmqctl list_users
# rabbitmqctl list_vhosts
```

## Web-Based Management Plugin

To enable the RabbitMQ management interface:

```sh
# rabbitmq-plugins enable rabbitmq_management
# rcctl restart rabbitmq
```

This exposes:

- **Port 15672**: web UI (HTTP)
- **Port 15671**: web UI (HTTPS, if TLS is configured)

Access it at <http://localhost:15672>
using any created user credentials.

Create an admin user:

```sh
# rabbitmqctl add_user adminuser strongpass
# rabbitmqctl set_user_tags adminuser administrator
```

## TLS Support

To enable TLS, add to `/etc/rabbitmq/rabbitmq.conf`:

```conf
listeners.ssl.default = 0.0.0.0:5671
ssl_options.certfile = /etc/ssl/rabbitmq/server.crt
ssl_options.keyfile = /etc/ssl/rabbitmq/server.key
ssl_options.cacertfile = /etc/ssl/rabbitmq/ca.crt
ssl_options.verify = verify_peer
ssl_options.fail_if_no_peer_cert = true
```

Ensure correct permissions:

```sh
# chown _rabbitmq:_rabbitmq /etc/ssl/rabbitmq/*
# chmod 640 /etc/ssl/rabbitmq/*
```

Then restart:

```sh
# rcctl restart rabbitmq
```

## Firewall Configuration

If running outside localhost, allow ports through `pf.conf`:

```pf
pass in on $int_if proto tcp from trusted_nets to port { 5672 5671 15672 }
```

Avoid exposing RabbitMQ to untrusted networks unless TLS is enabled.

## Service Management

Standard service control:

```sh
# rcctl enable rabbitmq
# rcctl start rabbitmq
# rcctl check rabbitmq
# rcctl restart rabbitmq
```

Check runtime info:

```sh
# rabbitmqctl status
# rabbitmqctl list_queues
# rabbitmqctl list_connections
```

## Logging

RabbitMQ logs are written to:

```text
/var/log/rabbitmq/
```

Use `tail -f` to follow log output:

```sh
# tail -f /var/log/rabbitmq/rabbit@$(hostname -s).log
```
