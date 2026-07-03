# Rspamd

## Synopsis

Rspamd is a fast, modular, and extensible spam filtering system designed to process large volumes of email efficiently. It evaluates messages using a variety of methods including regular expressions, statistical analysis, SPF/DKIM/DMARC checks, and URL reputation. Rspamd supports integration with MTAs such as `smtpd(8)`, Postfix, and Exim via the milter protocol or by using proxy workers.

This chapter describes the installation and configuration of Rspamd on OpenBSD, focusing on its integration with OpenBSD’s `smtpd(8)` and other supported mail transfer agents.

## Features

- Spam classification and scoring
- Built-in support for DKIM, SPF, and DMARC validation
- Statistical filtering with Redis backend
- Milter and proxy protocol support
- Web-based administration and status interface
- Lua scripting engine for custom rules
- Integration with antivirus and antiphishing backends

## Installation

Rspamd is not included in the OpenBSD base system. Install it using `pkg_add(1)`:

```sh
# pkg_add rspamd
```

Dependencies such as Redis and optional Lua modules will be installed automatically.

To enable and start Rspamd:

```sh
# rcctl enable rspamd
# rcctl start rspamd
```

The main configuration directory is located at:

```text
/etc/rspamd/
```

The runtime data is located in:

```text
/var/db/rspamd/
```

## Architecture Overview

Rspamd uses a multi-process architecture composed of the following components:

- **controller**: handles configuration reload, statistics, and HTTP requests
- **normal workers**: perform spam checks and filtering
- **proxy workers**: used when running behind MTAs in milter or proxy mode
- **rspamc**: the client tool used to submit messages for classification or learning

## Configuration Structure

Rspamd separates its configuration into several files:

| File or Directory | Purpose |
| --- | --- |
| `/etc/rspamd/rspamd.conf` | Main configuration file |
| `/etc/rspamd/worker-proxy.inc` | Proxy worker settings |
| `/etc/rspamd/worker-controller.inc` | Controller (HTTP admin interface) settings |
| `/etc/rspamd/worker-normal.inc` | Standard filtering worker settings |
| `/etc/rspamd/modules.d/` | Individual module configurations |

After making changes to any configuration files, restart the service:

```sh
# rcctl restart rspamd
```

## Enabling the Web Interface

The controller listens on `localhost:11334` by default. To enable remote access:

Edit `/etc/rspamd/worker-controller.inc`:

```
controller {
  bind_socket = "0.0.0.0:11334";
  password = "$2$myadminpasswordhash"; # generate using rspamadm
  enable_password = "$2$myenablepasswordhash";
  secure_ip = "127.0.0.1";
}
```

Generate the password hash using:

```sh
# rspamadm pw
```

It is advisable to limit access to trusted networks using `secure_ip` or firewall rules.

Access the web UI at:

```
http://localhost:11334/
```

## Integration with smtpd(8)

Rspamd can be integrated into `smtpd(8)` using its proxy mode with the `filter` directive.

### Example: /etc/mail/smtpd.conf

```pf
filter rspamd proc-exec "/usr/local/bin/rspamd-milter --socket /tmp/rspamd-milter.sock"

listen on egress tls pki mail.example.org filter "rspamd"
listen on localhost
match from any for domain "example.org" action "local"
```

Make sure Rspamd is running in milter mode and listening on the specified socket.

Edit `/etc/rspamd/rspamd.conf` to include:

```
worker "proxy" {
  bind_socket = "/tmp/rspamd-milter.sock mode=0666";
  timeout = 120s;
  milter = yes;
}
```

Then restart both services:

```sh
# rcctl restart rspamd
# rcctl restart smtpd
```

## Integration with Postfix

To use Rspamd as a milter with Postfix:

In `/etc/rspamd/rspamd.conf`:

```
worker "proxy" {
  bind_socket = "/var/run/rspamd/rspamd.sock mode=0660 owner=_rspamd group=_postfix";
  milter = yes;
}
```

In `/etc/postfix/main.cf`:

```
smtpd_milters = unix:/var/run/rspamd/rspamd.sock
non_smtpd_milters = unix:/var/run/rspamd/rspamd.sock
milter_protocol = 6
milter_mail_macros = i {mail_addr} {client_addr} {client_name} {auth_authen}
```

Then restart Postfix:

```sh
# postfix reload
```

## Mail Scanning with rspamc

The `rspamc` tool can be used to manually scan messages, train the Bayesian classifier, or check scores:

```sh
$ rspamc -h localhost:11333 < message.eml
$ rspamc learn_spam < spam.eml
$ rspamc learn_ham < ham.eml
```

## Using Redis for Statistics

Rspamd uses Redis for statistical storage and rate-limiting. Install and enable Redis:

```sh
# pkg_add redis
# rcctl enable redis
# rcctl start redis
```

In `/etc/rspamd/local.d/classifier-bayes.conf`:

```
backend = "redis";
servers = "127.0.0.1";
```

Restart Rspamd:

```sh
# rcctl restart rspamd
```

## DKIM Signing and Verification

### Signing Outgoing Mail

Generate DKIM keys:

```sh
# rspamadm dkim_keygen -d example.org -s mail > mail.dkim.key
# mv mail.dkim.key /etc/rspamd/dkim/example.org.mail.key
```

Set permissions:

```sh
# chown _rspamd:_rspamd /etc/rspamd/dkim/example.org.mail.key
# chmod 0400 /etc/rspamd/dkim/example.org.mail.key
```

In `/etc/rspamd/local.d/dkim_signing.conf`:

```
domain {
  example.org {
    selector = "mail";
    path = "/etc/rspamd/dkim/example.org.mail.key";
  }
}
```

Publish the DKIM public key in DNS (output by `dkim_keygen`).

### Verifying DKIM

Rspamd verifies DKIM automatically if the `dkim.conf` module is enabled. Configuration is usually in `/etc/rspamd/modules.d/dkim.conf`.

## DMARC and SPF

Ensure the following modules are enabled and configured:

- `/etc/rspamd/modules.d/dmarc.conf`
- `/etc/rspamd/modules.d/spf.conf`

Use public DNS resolvers or configure `/etc/resolv.conf` for correct lookups.

## Bayes Training

Rspamd learns spam and ham from messages. Submit them via `rspamc`:

```sh
$ rspamc learn_spam < spam.eml
$ rspamc learn_ham < ham.eml
```

To automate training from a mail folder, use `rspamadm` or configure the `neural` and `learned` modules.

## Logging and Monitoring

Logs are written to:

```text
/var/log/rspamd/rspamd.log
```

Use `rspamc stat` to view live statistics:

```sh
$ rspamc stat
```

Use `rspamadm configdump` to inspect effective configuration:

```sh
# rspamadm configdump
```

## Security Considerations

- Run Rspamd as `_rspamd` with limited privileges
- Use local sockets or restrict TCP listener access
- Protect the controller interface with strong passwords and IP restrictions
- Regularly update configuration files and rulesets
- Validate SPF, DKIM, and DMARC policies for incoming mail

## File Locations

| Path | Description |
| --- | --- |
| `/etc/rspamd/` | Configuration directory |
| `/var/log/rspamd/rspamd.log` | Log file |
| `/var/db/rspamd/` | Runtime and statistical data |
| `/etc/rspamd/dkim/` | DKIM private keys |
| `/usr/local/bin/rspamd-milter` | Milter helper for smtpd(8) |
