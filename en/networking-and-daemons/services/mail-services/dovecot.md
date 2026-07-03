# Dovecot

## Synopsis

[Dovecot](https://dovecot.org/)
is a secure and high-performance mail delivery agent (MDA) and IMAP/POP3 server. It is commonly used alongside a mail transfer agent (MTA) such as `smtpd(8)`, Postfix, or Exim to provide access to local mailboxes via IMAP or POP3 protocols. Dovecot supports standard mailbox formats (`mbox`, `Maildir`), secure authentication (including PAM and SQL backends), and strong TLS integration.

This chapter explains how to install, configure, and enable Dovecot on OpenBSD for common deployment scenarios, including local delivery, virtual mailboxes, and secure IMAP access.

## Features

- IMAP4rev1 and POP3 protocols with full TLS support
- Mailbox format support: `mbox`, `Maildir`, `dbox`
- Authentication via PAM, passwd, SQL, or static userdb
- Integrated LMTP server for local delivery
- Support for virtual users and chrooted mail storage
- Built-in sieve filtering (via plugin)
- Efficient indexing and fast mail access
- Compatible with `smtpd(8)`, Postfix, and Exim

## Installing Dovecot

Dovecot is not included in the OpenBSD base system. Install it using `pkg_add(1)`:

```sh
# pkg_add dovecot
```

This installs the Dovecot daemon and example configuration files under `/etc/dovecot/`.

## Enabling the Service

Enable Dovecot to start at boot:

```sh
# rcctl enable dovecot
# rcctl start dovecot
```

The main configuration file is `/etc/dovecot/dovecot.conf`. Additional configuration is located in `/etc/dovecot/conf.d/`.

## Configuration

Dovecot provides a modular configuration system. Each major subsystem (e.g., protocols, authentication, mail storage) has a separate configuration file in `conf.d/`. These files are included from the main `dovecot.conf` using `!include conf.d/*.conf`.

### Example dovecot.conf

```conf
log_path = /var/log/dovecot.log
protocols = imap pop3 lmtp
disable_plaintext_auth = yes
mail_privileged_group = _dovecot
!include conf.d/*.conf
```

### Mail Location

Define the mailbox storage format and location. For system users using `Maildir` under their home directory:

File: `/etc/dovecot/conf.d/10-mail.conf`

```conf
mail_location = maildir:~/Maildir
mail_access_groups = _dovecot
```

Alternatively, for `mbox`:

```conf
mail_location = mbox:~/mail:INBOX=/var/mail/%u
```

Ensure each user has a `Maildir/` or `mail/` directory initialized.

### Authentication

File: `/etc/dovecot/conf.d/10-auth.conf`

To authenticate system users (from `/etc/passwd`):

```conf
disable_plaintext_auth = yes
auth_mechanisms = plain login
!include auth-system.conf.ext
```

The `auth-system.conf.ext` file configures authentication against `/etc/passwd` and `login.conf`.

For virtual users (e.g., SQL or passwd-style files), refer to `auth-passwdfile.conf.ext`.

### TLS Configuration

File: `/etc/dovecot/conf.d/10-ssl.conf`

```conf
ssl = required
ssl_cert = </etc/ssl/mail.example.org.crt
ssl_key  = </etc/ssl/private/mail.example.org.key
```

Certificates may be obtained via [acme-client(1)] or another ACME implementation. Dovecot opens the certificate and key before dropping root privileges, so keep the private key readable only by `root` (for example, mode `0600`).

### Listener Configuration

File: `/etc/dovecot/conf.d/10-master.conf`

Defines the socket listeners for IMAP, POP3, and LMTP.

To listen on all interfaces:

```
service imap-login {
  inet_listener imap {
    port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}

service pop3-login {
  inet_listener pop3 {
    port = 110
  }
  inet_listener pop3s {
    port = 995
    ssl = yes
  }
}
```

To disable POP3:

```conf
protocols = imap lmtp
```

### LMTP Delivery from smtpd or Postfix

File: `/etc/dovecot/conf.d/10-master.conf`

```
service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = _postfix
    group = _postfix
  }
}
```

In `smtpd.conf`, configure LMTP delivery:

```
action "local" lmtp "/var/spool/postfix/private/dovecot-lmtp"
match for local action "local"
```

## Mailbox Initialization

Each user must have a properly initialized mailbox directory. For `Maildir`:

```sh
$ maildirmake ~/Maildir
$ chmod -R go-rwx ~/Maildir
```

Dovecot will automatically create missing subdirectories when the user logs in.

## Testing the Configuration

Check Dovecot’s configuration for syntax errors:

```sh
# dovecot -n
# dovecot -a
```

Tail the log for authentication or mail access issues:

```sh
# tail -f /var/log/dovecot.log
```

## Using TLS

Verify IMAP over TLS using `openssl`:

```sh
$ openssl s_client -connect mail.example.org:993
```

Dovecot will show a log entry upon successful connection and authentication.

## User Authentication Examples

### System Users

No extra configuration is needed if users already exist in `/etc/passwd` and have home directories with Maildirs.

### Virtual Users with passwd-file

Create `/etc/dovecot/users`:

```
user1@example.org:{PLAIN}password1:1001:1001::/var/vmail/user1::userdb_mail=maildir:/var/vmail/user1/Maildir
```

File: `/etc/dovecot/conf.d/auth-passwdfile.conf.ext`

```
passdb {
  driver = passwd-file
  args = scheme=PLAIN /etc/dovecot/users
}

userdb {
  driver = static
  args = uid=vmail gid=vmail home=/var/vmail/%n
}
```

Set ownership and permissions:

```sh
# chown _dovecot:_dovecot /etc/dovecot/users
# chmod 0600 /etc/dovecot/users
```

## Dovecot Sieve Support

Install sieve support via plugin:

```sh
# pkg_add dovecot-pigeonhole
```

Edit `/etc/dovecot/conf.d/20-managesieve.conf`:

```
protocols = $protocols sieve

service managesieve-login {
  inet_listener sieve {
    port = 4190
  }
}

plugin {
  sieve = ~/.dovecot.sieve
  sieve_dir = ~/sieve
}
```

Restart Dovecot to apply changes.

## Administrative Tools

- `doveadm user` — list known users
- `doveadm mailbox list` — show user mailboxes
- `doveadm auth test` — verify login authentication
- `doveadm log errors` — show recent errors
- `doveadm stats` — show performance metrics
- `doveadm reload` — reload configuration
- `doveadm kick` — disconnect users

Example:

```sh
# doveadm auth test alice@example.org password
passdb: alice@example.org auth succeeded
```

## File Locations

| File | Purpose |
| --- | --- |
| `/etc/dovecot/dovecot.conf` | Main configuration file |
| `/etc/dovecot/conf.d/` | Modular subsystem configuration |
| `/var/log/dovecot.log` | Main log file |
| `/etc/dovecot/users` | passwd-style file for virtual users |
| `/etc/ssl/mail.example.org.*` | TLS certificate and key |
| `/var/vmail/` | Home directory root for virtual users |

## Security and Hardening

- Do not allow plaintext logins unless under TLS (`disable_plaintext_auth = yes`)
- Run Dovecot under its own user (`_dovecot`)
- Use strong TLS certificates
- Use `auth_chroot` for virtual mail storage where possible
- Restrict file permissions on sensitive files such as `/etc/dovecot/users`
