# vsftpd

## Synopsis

**vsftpd (Very Secure FTP Daemon)** is a minimal, high-performance FTP server with a strong focus on security. It is commonly used in environments where low resource usage and correctness are more important than flexibility.

Unlike `ProFTPD`, `vsftpd` does not support extensive module systems or virtual user databases natively. However, it provides a simple, robust FTP service that supports:

- Passive and active FTP modes
- Anonymous and local user logins
- Chrooting users
- FTPS encryption

vsftpd is not part of the OpenBSD base system and must be installed from packages.

## FTP Server Comparison

The OpenBSD Handbook documents three FTP server implementations:

| Feature | [ftpd](ftpd.md) (base) | [vsftpd](vsftpd.md) (pkg) | [ProFTPD](proftpd.md) (pkg) |
| --- | --- | --- | --- |
| Source | Included in base system | Available via `pkg_add` | Available via `pkg_add` |
| TLS (FTPS) Support | No | Yes | Yes |
| Chrooting | Global `/ftp` | Per-user (`chroot_local_user`) | Per-user (`DefaultRoot`) |
| Anonymous FTP | Yes | Yes | Yes |
| Virtual Users | No | No | Yes (`AuthUserFile`, etc.) |
| Configuration Style | Built-in flags only | Flat config file | Modular, Apache-style |
| Logging | syslog | xferlog-compatible file | syslog or custom file |
| FTPS Mode | Not supported | Explicit FTPS (TLS) | Explicit FTPS (TLS) |
| Resource Usage | Very low | Low | Moderate |
| Access Control | Minimal | Moderate | Extensive |
| Use Case Fit | Minimal install sets | Secure public/private FTP | Advanced FTP with fine control |

- **[ftpd](ftpd.md)**is ideal for simple, anonymous-only FTP on trusted networks.
- **[vsftpd](vsftpd.md)**is appropriate when TLS and strict isolation are required with low overhead.
- **[ProFTPD](proftpd.md)**is suited for environments that require flexibility, virtual user support, and complex policy enforcement.

## Installation

Install `vsftpd` using the packages system:

```sh
# pkg_add vsftpd
```

This installs the daemon (`vsftpd`) and the default configuration file:

```text
/etc/vsftpd.conf
```

## Basic Configuration

A minimal configuration supporting both anonymous access and system user login might look like this:

```conf
listen=YES
listen_ipv6=NO

anonymous_enable=YES
local_enable=YES
write_enable=NO
local_umask=022

chroot_local_user=YES

ftpd_banner=Welcome to OpenBSD vsftpd

# Limit passive ports for firewalling
pasv_min_port=49152
pasv_max_port=49200

# Use unprivileged user
nopriv_user=_vsftpd

# Secure log location
xferlog_enable=YES
xferlog_file=/var/log/vsftpd.log
```

Create a home directory for anonymous access:

```sh
# mkdir -p /var/ftp/pub
# chown root:wheel /var/ftp
# chown -R _vsftpd:_vsftpd /var/ftp/pub
```

To allow system users to log in (e.g., `ftpuser`):

```sh
# useradd -m -s /sbin/nologin ftpuser
# passwd ftpuser
# mkdir /home/ftpuser/incoming
# chown ftpuser:ftpuser /home/ftpuser/incoming
```

## Starting the Service

Start `vsftpd` manually to test:

```sh
# /usr/local/sbin/vsftpd /etc/vsftpd.conf
```

To run it automatically at boot, append to `/etc/rc.local`:

```sh
if [ -x /usr/local/sbin/vsftpd ]; then
    echo -n ' vsftpd'; /usr/local/sbin/vsftpd /etc/vsftpd.conf
fi
```

Alternatively, create a custom `rc.d` script for use with `rcctl`.

## FTPS (TLS) Support

vsftpd supports FTPS (FTP over SSL/TLS). To enable it:

1. Generate or obtain a certificate and key pair.

```sh
# mkdir -p /etc/ssl/vsftpd
# openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout /etc/ssl/vsftpd/server.key \
  -out /etc/ssl/vsftpd/server.crt \
  -days 365
```

2. Set ownership and permissions:

```sh
# chmod 600 /etc/ssl/vsftpd/server.key
# chmod 644 /etc/ssl/vsftpd/server.crt
```

3. Modify the configuration:

```conf
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES

rsa_cert_file=/etc/ssl/vsftpd/server.crt
rsa_private_key_file=/etc/ssl/vsftpd/server.key

ssl_tlsv1_2=YES
ssl_tlsv1_3=YES
```

Restart `vsftpd`. FTPS clients such as lftp or FileZilla can now connect securely.

## Passive and Active FTP

By default, vsftpd supports **passive mode**, which is recommended for most environments behind firewalls.

To restrict the port range for passive data connections:

```conf
pasv_min_port=49152
pasv_max_port=49200
```

OpenBSD’s `pf.conf` should be updated to permit control and data ports:

```pf
pass in on $int_if proto tcp from any to (self) port 21
pass in on $int_if proto tcp from any to (self) port 49152:49200
```

Active FTP requires client firewalls to permit incoming connections, which may not be feasible.

## Logging and Monitoring

vsftpd logs to `/var/log/vsftpd.log` in xferlog format if `xferlog_enable=YES` is set.

Example:

```sh
# tail -f /var/log/vsftpd.log
```

To test access:

```sh
$ ftp localhost
$ lftp ftp://ftpuser@localhost
```

Use `tcpdump` or `netstat -an` to verify connection attempts and port usage.
