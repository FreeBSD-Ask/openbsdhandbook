# syslogd

## Synopsis

The `syslogd(8)` daemon is responsible for collecting and distributing log messages from the OpenBSD system and its services. It listens for log entries from the kernel and user processes via the `syslog(3)` interface and routes them according to rules defined in the configuration file `/etc/syslog.conf`.

By default, `syslogd` writes messages to text files in `/var/log/`. It can also forward messages to remote log hosts using UDP or TLS-encrypted TCP. On OpenBSD, `syslogd` runs with strict privilege separation and filesystem access constraints and is enabled by default.

This chapter describes how to customize logging behavior, filter and redirect logs, enable remote logging, and control logging behavior at runtime.

## Default Operation

When OpenBSD boots, `syslogd` is started automatically. It reads its configuration from `/etc/syslog.conf` and begins listening for local log messages over the Unix domain socket `/dev/log`.

Log files are rotated periodically by `newsyslog(8)` according to settings in `/etc/newsyslog.conf`.

To confirm that `syslogd` is running:

```sh
$ ps -aux | grep syslogd
```

To check its status via `rcctl(8)`:

```sh
# rcctl check syslogd
```

## Configuration: /etc/syslog.conf

The `/etc/syslog.conf` file defines which messages are written where. Each line consists of a *selector* (facility.level) and an *action*. For example:

```
auth.info               /var/log/authlog
cron.*                  /var/log/cron
mail.err                /dev/console
*.notice;auth,authpriv.none  /var/log/messages
```

This configuration performs the following:

- Writes authentication logs at info level or higher to `/var/log/authlog`
- Logs all `cron` messages to `/var/log/cron`
- Sends mail errors to the console
- Logs all other notices except those from `auth` and `authpriv` to `/var/log/messages`

To apply changes:

```sh
# kill -HUP $(cat /var/run/syslog.pid)
```

### Facilities and Levels

Facilities group related services (e.g., `auth`, `cron`, `mail`). Levels indicate severity, from most critical to least:

- `emerg` – system is unusable
- `alert` – action must be taken immediately
- `crit` – critical condition
- `err` – error condition
- `warning` – warning
- `notice` – normal but significant
- `info` – informational
- `debug` – debug-level messages

The special level `none` can be used to exclude a facility.

## Remote Logging

To forward logs to a remote syslog server, add a line such as the following to `/etc/syslog.conf`:

```
*.*    @loghost.example.net
```

For TLS-encrypted logging, use the `tls` prefix:

```
*.*    tls://loghost.example.net
```

Then restart `syslogd`:

```sh
# rcctl restart syslogd
```

Remote logging requires DNS resolution and network access. Ensure appropriate rules exist in `pf.conf` if packet filtering is enabled.

### Accepting Remote Logs

To allow incoming logs from other hosts, add the `-u` or `-U` flags to `syslogd` via `rcctl`:

```sh
# rcctl set syslogd flags -u
# rcctl restart syslogd
```

The `-u` flag enables unencrypted UDP reception. For encrypted TCP logging (TLS), use `-T` and configure certificates as described in `syslogd(8)`.

## Runtime Control

To disable `syslogd`, for example in a chrooted or isolated environment:

```sh
# rcctl disable syslogd
# rcctl stop syslogd
```

To re-enable it later:

```sh
# rcctl enable syslogd
# rcctl start syslogd
```

## Viewing and Managing Logs

Most log files are stored under `/var/log/`. Common files include:

- `/var/log/messages` — general system activity
- `/var/log/authlog` — authentication events
- `/var/log/daemon` — messages from long-running services
- `/var/log/cron` — scheduled task output
- `/var/log/maillog` — mail subsystem messages

To follow log output in real time:

```sh
# tail -f /var/log/messages
```

## Log Rotation and Retention

Log rotation is handled by `newsyslog(8)`, which is invoked daily from `/etc/daily`. The configuration file `/etc/newsyslog.conf` defines which logs are rotated, how many archives to keep, and when to compress them.

A typical entry:

```
/var/log/messages     600  5     100  *     Z
```

This retains five compressed archives of `/var/log/messages`, rotating the log once it exceeds 100 KB.

## Troubleshooting

If log files are empty or missing:

- Ensure that `syslogd` is running
- Confirm that `pf(4)` is not blocking outbound or inbound syslog traffic
- Verify that relevant selectors exist in `/etc/syslog.conf`
- Ensure there is sufficient disk space under `/var`

To test whether log messages are reaching the daemon:

```sh
$ logger -p user.info "This is a test log entry"
```

Then inspect the appropriate log file (such as `/var/log/messages`) for the test entry.
