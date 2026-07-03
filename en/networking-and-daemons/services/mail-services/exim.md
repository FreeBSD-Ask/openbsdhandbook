# Exim

## Synopsis

**Exim** is a highly configurable mail transfer agent (MTA) designed for Unix systems. Originally developed at the University of Cambridge, it offers a rich feature set and fine-grained control over mail routing, rewriting, and authentication. Exim is well suited to complex mail environments requiring deep customization.

This chapter explains how to install and configure Exim on OpenBSD, covering use cases including local delivery, TLS encryption, relay via a smarthost, and basic content filtering. While not part of the OpenBSD base system, Exim is available as a package and can be used as an alternative to Postfix or `smtpd(8)` when more advanced mail logic is required.

## Features

- Full Sendmail-style command-line compatibility
- Fine-grained routing and rewriting policies
- SMTP, submission, and LMTP support
- TLS, SASL, and DKIM support
- Virtual domains and user handling
- Flexible logging and queue management

## Installing Exim

To install the Exim package:

```sh
# pkg_add exim
```

To avoid conflicts with the default `smtpd(8)` service, disable it:

```sh
# rcctl disable smtpd
# rcctl stop smtpd
```

Enable and start Exim:

```sh
# rcctl enable exim
# rcctl start exim
```

Exim’s configuration is located in:

```text
/usr/local/exim/configure
```

If this file does not exist, copy the default template:

```sh
# cp /usr/local/share/examples/exim/configure.default /usr/local/exim/configure
```

## Configuration Overview

Unlike Postfix, Exim uses a **single monolithic configuration file**: `/usr/local/exim/configure`.

The file is divided into sections, including:

- `MAIN CONFIGURATION SETTINGS`
- `ACL CONFIGURATION`
- `ROUTERS CONFIGURATION`
- `TRANSPORTS CONFIGURATION`
- `RETRY CONFIGURATION`
- `REWRITE CONFIGURATION`
- `AUTHENTICATION CONFIGURATION`

Each section controls a different aspect of Exim’s behavior, and directives are order-sensitive.

After editing the configuration, test it:

```sh
# exim -bV
# exim -C /usr/local/exim/configure -bV
```

Reload Exim to apply changes:

```sh
# rcctl reload exim
```

## Minimal Configuration Example

A simple configuration suitable for local delivery and outgoing mail via DNS:

### `/usr/local/exim/configure`

```
primary_hostname = mail.example.org
domainlist local_domains = @ : localhost
domainlist relay_to_domains =
hostlist   relay_from_hosts = 127.0.0.1

begin routers
  dnslookup:
    driver = dnslookup
    domains = ! +local_domains
    transport = remote_smtp

begin transports
  remote_smtp:
    driver = smtp

  local_delivery:
    driver = appendfile
    directory = $home/Maildir
    maildir_format

begin retry
*                      F,2h,10m; G,16h,1h,1.5; F,4d,6h

begin rewrite

begin authenticators
```

This configuration:

- Uses Maildir delivery for local users
- Sends all outgoing mail via DNS MX resolution
- Does not relay for unauthenticated remote users

## Accepting Internet Mail

To allow incoming mail from external systems:

1. Ensure `domainlist local_domains` includes the system’s FQDN or desired domain(s):

   ```conf
   domainlist local_domains = example.org : localhost
   ```
2. Make sure the system listens on external interfaces (default behavior):

   ```sh
   # netstat -an | grep 25
   ```
3. Open TCP port 25 in `pf.conf` if required.
4. Ensure proper MX DNS records are configured.

## TLS Configuration

To enable TLS, create or install a valid certificate and private key, e.g.:

- `/etc/ssl/example.org.crt`
- `/etc/ssl/private/example.org.key`

In the `MAIN CONFIGURATION` section, add:

```conf
tls_advertise_hosts = *
tls_certificate = /etc/ssl/example.org.crt
tls_privatekey = /etc/ssl/private/example.org.key
```

To force STARTTLS:

```conf
tls_require_ciphers = HIGH
```

Check TLS support:

```sh
$ openssl s_client -starttls smtp -connect mail.example.org:25
```

## Smarthost (Relay) Configuration

To relay all outgoing mail via a provider:

1. Add authentication credentials to `/usr/local/exim/passwd.client`:

   ```
   mail.example.org:login@example.org:password123
   ```
2. Restrict file permissions:

   ```sh
   # chmod 600 /usr/local/exim/passwd.client
   ```
3. Add a router and transport to use the smarthost:

### Routers Section

```conf
smarthost:
  driver = manualroute
  domains = ! +local_domains
  transport = remote_smtp_smarthost
  route_list = * smtp.provider.net
```

### Transports Section

```conf
remote_smtp_smarthost:
  driver = smtp
  hosts_require_auth = *
  hosts_require_tls = *
```

### Authentication Section

```conf
plain:
  driver = plaintext
  public_name = PLAIN
  client_send = : login@example.org : password123
```

Reload Exim:

```sh
# rcctl reload exim
```

## Virtual Domains and Users

For systems hosting multiple domains with virtual users, Exim supports multiple lookup mechanisms.

A basic configuration using a flat file:

### `/etc/exim/virtual.aliases`

```
admin@example.org    localuser
info@example.org     infoaccount
```

### Routers Section

```
virtual_aliases:
  driver = redirect
  domains = example.org
  data = ${lookup{$local_part@$domain}lsearch{/etc/exim/virtual.aliases}}
```

Create the file and set permissions:

```sh
# touch /etc/exim/virtual.aliases
# chmod 644 /etc/exim/virtual.aliases
```

## Filtering and ACLs

Exim provides fine-grained ACLs for spam blocking and policy enforcement.

To reject non-TLS connections or unauthenticated relays, modify the `acl_smtp_rcpt` section:

```conf
accept  authenticated = *
accept  hosts = +relay_from_hosts
deny    message = Relay not permitted
```

For basic spam header filtering:

```
deny message = Spam detected
     condition = ${if match{$h_Subject:}{(?i)free money}}
```

Exim can also integrate with external spam filters such as SpamAssassin, rspamd, and ClamAV via the `transport_filter` or `av_scanner` options.

## Mail Queue and Logs

To view the queue:

```sh
# exim -bp
```

To flush the queue:

```sh
# exim -qff
```

To examine logs:

```sh
# tail -f /var/log/exim/mainlog
```

To view a message log:

```sh
# exim -Mvh <message-id>
# exim -Mvb <message-id>
```

## Testing

Send a message from the command line:

```sh
$ echo test | mail -s "Test from Exim" user@example.org
```

Check that it appears in the mail queue or log files.

## Summary of Key Configuration Files

| File | Purpose |
| --- | --- |
| `/usr/local/exim/configure` | Main configuration file |
| `/etc/exim/virtual.aliases` | Virtual alias mapping |
| `/usr/local/exim/passwd.client` | SMTP authentication credentials |
| `/var/log/exim/mainlog` | Main log file |

## Alternatives

Exim is best suited for administrators who require full control over mail routing, rewriting, and authentication. For simpler use cases, [Postfix](postfix.md)
or the base system’s [smtpd](smtpd.md)
daemon may offer easier configuration and tighter OpenBSD integration.
