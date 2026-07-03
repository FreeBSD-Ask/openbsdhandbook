# smtpd

## Synopsis

OpenSMTPD is the default mail transfer agent (MTA) in OpenBSD. It is implemented as the `smtpd(8)` daemon and designed to be secure, simple, and suitable for many use cases including local delivery, relaying, authenticated submission, and receiving mail for virtual domains.

This chapter describes how to configure `smtpd(8)` for common and advanced scenarios, including TLS, filtering, and support for additional modules via `opensmtpd-extras`.

## Features

- Secure by default, with privilege separation and sane defaults
- Local delivery to system users
- Relay via authenticated submission or smarthost
- Full IPv6 and TLS support
- Filtering and address rewriting
- Virtual user/domain support
- Integration with `filter-*` plugins and extras packages

## Enabling smtpd

To enable and start the `smtpd` service at boot:

```sh
# rcctl enable smtpd
# rcctl start smtpd
```

The daemon will read its configuration from `/etc/mail/smtpd.conf`.

## Basic Configuration

A minimal `smtpd.conf(5)` file enabling local delivery and outgoing mail might look like this:

```
listen on lo0

action "local" mbox
action "outbound" relay

match from local for local action "local"
match from local for any action "outbound"
```

This configuration:

- Listens only on the loopback interface
- Defines one action for local mailbox delivery and one action for outbound relay
- Allows local users to send mail to other local users
- Relays outbound mail from local users to other systems using DNS

To apply changes:

```sh
# smtpctl reload
```

## Accepting Mail from the Internet

To receive mail from the outside world:

```pf
pki example.org cert "/etc/mail/certs/example.org.crt"
pki example.org key "/etc/mail/certs/example.org.key"

listen on egress tls pki example.org

action "inbound" maildir
match from any for domain example.org action "inbound"
```

This example:

- Defines the certificate and key used by the listener
- Listens on the external interface with TLS enabled
- Accepts mail from the internet for the domain `example.org`
- Delivers it to `~/Maildir` in the recipient’s home directory

To use TLS, a valid certificate must be placed under `/etc/mail/certs/`:

```sh
# mkdir -p /etc/mail/certs
# cp fullchain.pem /etc/mail/certs/example.org.crt
# cp privkey.pem /etc/mail/certs/example.org.key
```

Ensure permissions are correct:

```sh
# chown root:_smtpd /etc/mail/certs/*
# chmod 640 /etc/mail/certs/*
```

Then define the certificate using the `pki` block in `smtpd.conf`:

```
pki example.org cert "/etc/mail/certs/example.org.crt"
pki example.org key "/etc/mail/certs/example.org.key"
```

## Outgoing Mail via Smarthost

To relay all outgoing mail through a remote mail server (e.g., ISP or upstream relay):

```
action "relay" relay host smtp+auth://user@smtp.example.net auth <secrets>
match for any action "relay"
```

Define the authentication secret in `/etc/mail/secrets`:

```
user@smtp.example.net password123
```

Secure the file:

```sh
# chmod 600 /etc/mail/secrets
# chown root:_smtpd /etc/mail/secrets
```

## Virtual Domains and Users

To receive mail for domains not mapped to system users, use virtual aliases:

### Define Table for Virtual Mappings

Create `/etc/mail/virtuals`:

```
info@example.org      alice
admin@example.org     bob
contact@example.org   contactuser
```

Convert it to a usable table:

```sh
# makemap -t aliases /etc/mail/virtuals > /etc/mail/virtuals.db
```

### Update smtpd.conf

```pf
table aliases file:/etc/mail/virtuals.db

action "virtuals" mbox virtual <aliases>
match from any for domain example.org action "virtuals"
```

## Filtering Mail

`filter` blocks allow simple yet effective filtering of incoming messages. Examples include:

### Greylisting

Enable basic greylisting:

```
filter "greylist" phase connect match !auth
filter "greylist" phase connect match !relay rdns
filter "greylist" phase connect match !relay sender
filter "greylist" phase connect match !relay helo

listen on egress filter "greylist"
```

### Blocking Specific Senders or Addresses

Create a table:

```
table badsenders file:/etc/mail/badsenders.txt
```

With contents:

```
spammer@example.com
junk@example.net
```

Then:

```
filter "blocklist" phase connect match sender <badsenders> reject
listen on egress filter "blocklist"
```

### Filtering with opensmtpd-extras

Install extra plugins:

```sh
# pkg_add opensmtpd-extras
```

This package includes:

- `filter-rspamd` – integrates with Rspamd
- `filter-clamav` – integrates with ClamAV
- `filter-dkimsign` – signs outbound mail with DKIM
- `filter-dnsbl` – DNS blocklist checking

Example DKIM signing:

```
filter "dkim" proc-exec "filter-dkimsign -d example.org -s default -k /etc/mail/dkim.key"

listen on egress filter "dkim"
```

## Local Delivery Options

OpenSMTPD can deliver to either `mbox` (traditional mailbox format) or `maildir` (per-message file format). Set the method in an `action` and select it from a `match` rule:

```pf
action "inbound" maildir
match from any for domain example.org action "inbound"
```

Ensure home directories contain `Maildir/` or `mbox` as appropriate.

## Administrative Commands

Use `smtpctl(8)` to manage the mail system:

```sh
# smtpctl show stats
```

Show statistics for messages processed, sessions handled, and more.

```sh
# smtpctl show queue
```

List the mail queue.

```sh
# smtpctl schedule all
```

Immediately attempt to send all queued messages.

```sh
# smtpctl remove <msg-id>
```

Remove a specific message from the queue.

```sh
# smtpctl log verbose
```

Enable verbose logging for debugging.

## Logging and Troubleshooting

Logs are written to `/var/log/maillog`. Tail the log for real-time updates:

```sh
# tail -f /var/log/maillog
```

For additional diagnostics:

```sh
# smtpd -dv
```

Runs `smtpd` in debug mode with verbose output.

## Summary of Key Configuration Files

| File | Purpose |
| --- | --- |
| `/etc/mail/smtpd.conf` | Main configuration file for smtpd |
| `/etc/mail/secrets` | Authentication credentials for relay hosts |
| `/etc/mail/virtuals` | Virtual user mappings (if used) |
| `/etc/mail/certs/` | TLS certificate and key storage |
| `/etc/mail/aliases` | Local user aliases (via `newaliases`) |

All changes to `smtpd.conf` or related tables require a reload:

```sh
# smtpctl reload
```
