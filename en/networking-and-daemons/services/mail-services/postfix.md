# Postfix

## Synopsis

Postfix is a full-featured mail transfer agent (MTA) known for its performance, ease of configuration, and strong focus on security. It is available as a package on OpenBSD and can be used as a drop-in replacement for the default `smtpd(8)` daemon when more advanced capabilities are required, such as compatibility with complex mail environments, detailed policy controls, or extensive logging.

This chapter provides a detailed guide to installing and configuring Postfix on OpenBSD, including common tasks such as sending and receiving mail, configuring TLS encryption, relaying via a smarthost, setting up virtual domains and users, and integrating spam and antivirus filters.

## Features

- Secure and modular design
- Supports SMTP, SMTPS, and submission ports
- Authentication and encryption with TLS
- Extensive queue management
- Flexible configuration using `main.cf` and `master.cf`
- Support for aliases, virtual domains, and LDAP integration
- Integration with `dovecot`, `spamassassin`, `amavisd`, and `rspamd`

## Installing Postfix

To install Postfix from packages:

```sh
# pkg_add postfix
```

To prevent conflicts with OpenSMTPD, disable `smtpd(8)` if it is enabled:

```sh
# rcctl disable smtpd
# rcctl stop smtpd
```

Enable and start the Postfix service:

```sh
# rcctl enable postfix
# rcctl start postfix
```

The main configuration directory is:

```sh
/etc/postfix/
```

## Configuration

Postfix uses two main configuration files:

- `/etc/postfix/main.cf` — Primary configuration
- `/etc/postfix/master.cf` — Service definitions and process control

After making changes, reload the configuration:

```sh
# postfix reload
```

Check syntax and configuration:

```sh
# postfix check
```

## Minimal Configuration

A basic setup sufficient for local delivery and outgoing mail relay through DNS:

```conf
myhostname = mail.example.org
myorigin = /etc/mailname
mydestination = localhost
relayhost =
inet_interfaces = loopback-only
inet_protocols = all
```

This configuration:

- Sets the mail hostname
- Relays outgoing messages using DNS
- Accepts mail only from the local machine

To enable delivery to local users, ensure `mailbox_command` or `mailbox_transport` is set appropriately, for example:

```conf
home_mailbox = Maildir/
```

Ensure that each user has a `~/Maildir/` directory.

## Accepting Internet Mail

To accept mail from remote systems:

```conf
inet_interfaces = all
myhostname = mail.example.org
mydestination = mail.example.org, localhost.localdomain, localhost
```

To restrict incoming mail to a specific domain:

```conf
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination
```

Ensure MX records for `example.org` point to the correct public IP address.

## TLS Encryption

To enable TLS for incoming and outgoing mail:

```conf
smtpd_tls_cert_file = /etc/ssl/example.org.crt
smtpd_tls_key_file = /etc/ssl/private/example.org.key
smtpd_use_tls = yes
smtpd_tls_auth_only = yes

smtp_tls_security_level = may
smtp_tls_note_starttls_offer = yes
```

Ensure proper permissions:

```sh
# chown root:_postfix /etc/ssl/private/example.org.key
# chmod 640 /etc/ssl/private/example.org.key
```

To force all clients to use encryption:

```conf
smtpd_tls_security_level = encrypt
```

## Outgoing Mail via Smarthost

To relay all outbound messages through a provider:

```conf
relayhost = [smtp.example.net]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_use_tls = yes
```

Create `/etc/postfix/sasl_passwd`:

```ini
[smtp.example.net]:587 user@example.net:password123
```

Convert and secure the file:

```sh
# postmap /etc/postfix/sasl_passwd
# chmod 600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
# chown root:_postfix /etc/postfix/sasl_passwd*
```

Reload Postfix:

```sh
# postfix reload
```

## Virtual Domains and Users

To host mail for domains and users not mapped to system accounts:

### Define `/etc/postfix/virtual`

```
admin@example.org    adminuser
sales@example.org    salesuser
info@example.org     infoaccount
```

Create and map the file:

```sh
# postmap /etc/postfix/virtual
```

Update `main.cf`:

```conf
virtual_alias_domains = example.org
virtual_alias_maps = hash:/etc/postfix/virtual
```

Reload Postfix:

```sh
# postfix reload
```

## Filtering Mail

Postfix supports a variety of content filters via content inspection and milter integration.

### Basic Header and Body Checks

To block specific subjects or bodies, use:

- `/etc/postfix/header_checks`
- `/etc/postfix/body_checks`

Enable in `main.cf`:

```conf
header_checks = regexp:/etc/postfix/header_checks
body_checks = regexp:/etc/postfix/body_checks
```

### Integration with SpamAssassin and ClamAV

Install `amavisd`:

```sh
# pkg_add amavisd-new spamassassin clamav
```

Configure `main.cf`:

```conf
content_filter = smtp-amavis:[127.0.0.1]:10024
```

Edit `/etc/postfix/master.cf` to include:

```conf
smtp-amavis unix - - n - 2 smtp
  -o smtp_data_done_timeout=1200
  -o smtp_send_xforward_command=yes
  -o disable_dns_lookups=yes

127.0.0.1:10025 inet n - n - - smtpd
  -o content_filter=
  -o receive_override_options=no_unknown_recipient_checks
  -o smtpd_helo_restrictions=
  -o smtpd_client_restrictions=
  -o smtpd_sender_restrictions=
  -o smtpd_recipient_restrictions=permit_mynetworks,reject
  -o mynetworks=127.0.0.0/8
  -o smtpd_authorized_xforward_hosts=127.0.0.0/8
```

## Administrative Commands

Check the queue:

```sh
# mailq
```

Flush the queue:

```sh
# postfix flush
```

View logs:

```sh
# tail -f /var/log/maillog
```

Examine configuration:

```sh
# postconf -n
```

## Testing

Send a test message:

```sh
$ mail -s "Test Postfix" user@example.org < /dev/null
```

Check if Postfix is listening:

```sh
# netstat -an | grep 25
```

Verify TLS:

```sh
$ openssl s_client -starttls smtp -connect mail.example.org:25
```

## Summary of Key Configuration Files

| File | Purpose |
| --- | --- |
| `/etc/postfix/main.cf` | Main configuration |
| `/etc/postfix/master.cf` | Daemon/service process configuration |
| `/etc/postfix/virtual` | Virtual address mapping |
| `/etc/postfix/sasl_passwd` | SMTP authentication credentials |
| `/var/log/maillog` | Postfix logging output |
| `/etc/postfix/header_checks` | Regex for filtering message headers |
| `/etc/postfix/body_checks` | Regex for filtering message bodies |

```sh
# postfix reload
```

Always reload after modifying configuration files.

## Alternatives

Postfix is suitable for most use cases, but lighter alternatives such as `smtpd(8)` (OpenSMTPD) may be preferred for systems requiring simplicity and tight OpenBSD integration.
