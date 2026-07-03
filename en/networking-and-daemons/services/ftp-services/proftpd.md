# ProFTPD

## Synopsis

**ProFTPD** is a highly configurable and standards-compliant FTP server that supports features such as virtual hosts, TLS encryption, chrooted sessions, anonymous and authenticated access, and integration with external user databases.

Unlike OpenBSD’s built-in `ftpd(8)`, which provides a minimal and secure FTP service, **ProFTPD** is available via packages and is suitable for administrators needing more advanced features or compatibility with legacy FTP clients.

This chapter explains how to install and configure ProFTPD on OpenBSD, including TLS encryption and optional virtual user support.

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

ProFTPD is available from the OpenBSD packages collection. Install it as follows:

```sh
# pkg_add proftpd
```

This installs the main daemon (`proftpd`), default configuration files, and optional modules.

The default configuration file is located at:

```text
/etc/proftpd/proftpd.conf
```

## Basic Configuration

The default configuration enables a single FTP site with basic settings. Begin by creating or editing `/etc/proftpd/proftpd.conf`:

```
ServerName                      "ProFTPD on OpenBSD"
ServerType                      standalone
DefaultServer                   on

Port                            21
Umask                           022
MaxInstances                    30

User                            _proftpd
Group                           _proftpd

DefaultRoot                    ~
RequireValidShell              off

<Limit LOGIN>
  AllowAll
</Limit>
```

- The service listens on port 21.
- `User` and `Group` define the credentials under which the daemon runs.
- `DefaultRoot ~` chroots users to their home directories.
- `RequireValidShell off` allows logins for users without a valid shell (e.g., `/sbin/nologin`).

Create home directories and test with an unprivileged system user:

```sh
# useradd -m -s /sbin/nologin ftpuser
# passwd ftpuser
```

Ensure proper ownership:

```sh
# chown ftpuser:_proftpd /home/ftpuser
```

## Starting the Service

Start the daemon manually to test:

```sh
# /usr/local/sbin/proftpd -c /etc/proftpd/proftpd.conf
```

To start ProFTPD at boot, add this line to `/etc/rc.local`:

```sh
if [ -x /usr/local/sbin/proftpd ]; then
    echo -n ' proftpd'; /usr/local/sbin/proftpd
fi
```

Alternatively, create an `rc.d` script to use with `rcctl(8)`.

To stop the service:

```sh
# pkill proftpd
```

## TLS Support

ProFTPD supports encrypted FTP using **FTPS** (not to be confused with `sftp`, which is based on SSH).

Create a TLS key and certificate (self-signed or via ACME):

```sh
# mkdir -p /etc/ssl/proftpd
# openssl req -x509 -newkey rsa:4096 -keyout /etc/ssl/proftpd/server.key -out /etc/ssl/proftpd/server.crt -days 365 -nodes
# chmod 600 /etc/ssl/proftpd/server.key
```

Then enable TLS in the configuration:

```
<IfModule mod_tls.c>
  TLSEngine                  on
  TLSLog                     /var/log/proftpd_tls.log
  TLSProtocol                TLSv1.2 TLSv1.3
  TLSRSACertificateFile      /etc/ssl/proftpd/server.crt
  TLSRSACertificateKeyFile   /etc/ssl/proftpd/server.key
  TLSOptions                 NoCertRequest
  TLSVerifyClient            off
  TLSRequired                on
</IfModule>
```

Restart the daemon and connect using an FTPS-capable client such as FileZilla or lftp:

```sh
$ lftp ftps://ftpuser@hostname
```

## Anonymous FTP

To enable anonymous access:

```
<Anonymous /home/ftp/pub>
  User                          _proftpd
  Group                         _proftpd
  UserAlias                     anonymous ftp
  RequireValidShell             off
  <Limit WRITE>
    DenyAll
  </Limit>
</Anonymous>
```

Create the directory:

```sh
# mkdir -p /home/ftp/pub
# chown _proftpd:_proftpd /home/ftp/pub
# chmod 755 /home/ftp/pub
```

Anonymous users will be able to retrieve files from this directory but cannot upload.

## Virtual Users

To create users that exist only in ProFTPD’s own user database (not system users), use the `mod_auth_file` module and an AuthUserFile.

Define the file in your configuration:

```
AuthUserFile /etc/proftpd/ftpd.passwd
AuthGroupFile /etc/proftpd/ftpd.group
AuthOrder mod_auth_file.c
```

Generate entries using standard format:

```sh
# ftpasswd --passwd --name=vuser1 --home=/srv/ftp/vuser1 --shell=/sbin/nologin --file=/etc/proftpd/ftpd.passwd
# ftpasswd --group --name=ftpgroup --gid=2001 --member=vuser1 --file=/etc/proftpd/ftpd.group
```

Create home directories and set permissions:

```sh
# mkdir -p /srv/ftp/vuser1
# chown _proftpd:_proftpd /srv/ftp/vuser1
```

## Logging

Logs are written to `/var/log/proftpd.log` and `/var/log/proftpd_tls.log` by default. These paths can be adjusted with the `SystemLog` and `TLSLog` directives.

Example:

```sh
tail -f /var/log/proftpd.log
```

## Firewall Considerations

ProFTPD uses:

- TCP port **21** for command connections
- A dynamic range of ports for **passive mode data connections**

To avoid firewall issues, explicitly define a passive port range:

```
PassivePorts 49152 49200
```

Then allow those ports in your `pf.conf`:

```pf
pass in on $int_if proto tcp from any to (self) port 21
pass in on $int_if proto tcp from any to (self) port 49152:49200
```

Do not expose FTP to the Internet unless TLS is used and firewalling is strict.
