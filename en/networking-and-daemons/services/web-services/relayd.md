# relayd

## Synopsis

**relayd(8)** is OpenBSD’s native application-layer proxy and filtering daemon. It supports:

- Reverse proxying (HTTP/HTTPS)
- TLS termination with optional re-encryption
- HTTP/HTTPS load balancing with active health checks
- Layer 7 filtering, header rewriting, and redirection

relayd is included in the OpenBSD base system and integrates tightly with `pf(4)` and `httpd(8)`. It is configured via `/etc/relayd.conf`.

Use relayd when you need to expose HTTP or HTTPS services securely, filter traffic at the application layer, or distribute requests across multiple backends.

## Common Use Cases

- Terminate TLS and forward HTTP to a local backend
- Filter and forward incoming HTTPS to multiple application servers
- Load-balance FastCGI traffic to PHP-FPM sockets
- Implement virtual hosting with separate domains and TLS keys

## Basic TLS Terminating Reverse Proxy

The most common relayd use case is to accept TLS connections and forward them unencrypted to a backend.

### Example: TLS frontend to local httpd(8)

```
relay "tlsproxy" {
    listen on egress port 443 tls
        tls certificate "/etc/ssl/example.org.fullchain.pem"
        tls key "/etc/ssl/private/example.org.key"

    forward to 127.0.0.1 port 8080
}
```

On the backend, httpd(8) should listen on localhost port 8080:

```
server "example.org" {
    listen on 127.0.0.1 port 8080
    root "/htdocs/example"
}
```

Start both services:

```sh
# rcctl enable relayd httpd
# rcctl start relayd httpd
```

Allow traffic on port 443 in `pf.conf`:

```pf
pass in on egress proto tcp to port 443
```

## Backend Pool with Load Balancing

relayd can distribute traffic across multiple backend hosts using a **table**:

```pf
table <webservers> { 192.0.2.10, 192.0.2.11, 192.0.2.12 }

relay "cluster" {
    listen on egress port 80
    forward to <webservers> port 80
}
```

Backends are monitored by default via TCP connect checks.

To use HTTP health checks:

```pf
table <webservers> {
    192.0.2.10 check http "/" code 200
    192.0.2.11 check http "/status" code 200
}
```

View real-time status:

```sh
# relayctl show hosts
```

## Redirecting HTTP to HTTPS

relayd supports inline HTTP redirection:

```
relay "http-redirect" {
    listen on egress port 80
    protocol "redirect"
}

protocol "redirect" {
    match request path "*" forward to "https://example.org"
}
```

This listens on port 80 and redirects all HTTP traffic to HTTPS.

## Using relayd with FastCGI

To serve FastCGI backends (e.g., PHP-FPM) through httpd(8), relayd can forward requests to Unix sockets:

```pf
table <phpfpm> { socket "/run/php-fpm.sock" }

relay "fcgi" {
    listen on 127.0.0.1 port 9000
    forward to <phpfpm>
}
```

In httpd.conf:

```
server "site" {
    listen on egress port 80
    root "/htdocs/site"
    location "*.php" {
        fastcgi socket "127.0.0.1" port 9000
    }
}
```

Note: In most cases, httpd(8) can connect directly to the socket without relayd.

## Logging and Monitoring

Enable logging in `/etc/relayd.conf`:

```
log updates
log state
```

Tail logs via syslog:

```sh
# tail -f /var/log/daemon
```

Query runtime state:

```sh
# relayctl show sessions
# relayctl show hosts
# relayctl statistics
```

Reload configuration without restart:

```sh
# rcctl reload relayd
```

## Security Considerations

- relayd runs as `_relayd` and is not chrooted
- TLS private keys must be readable by `_relayd`, typically 640 owned by `root:_relayd`
- Ensure only intended IPs can access TLS ports
- Combine with `pf(4)` for full traffic filtering

## Example pf.conf Rule

```pf
block in all
pass in on egress proto tcp from any to (self) port 443
pass in on egress proto tcp from any to (self) port 80
```

Limit access to internal-only relays by using loopback or RFC1918 address bindings.

## Service Management

Enable at boot and manage via `rcctl`:

```sh
# rcctl enable relayd
# rcctl start relayd
# rcctl check relayd
```

Check for syntax errors:

```sh
# relayd -n -f /etc/relayd.conf
```

Reload configuration:

```sh
# rcctl reload relayd
```
