# rsync

## Synopsis

`rsync` is a fast and versatile file-copying tool used to synchronize files and directories between local and remote systems. It performs delta-based file transfers, making it especially efficient for backups and large data synchronization tasks. Though not included in the OpenBSD base system, `rsync` is available via packages and integrates well with `ssh(1)` for secure transport.

This chapter documents the installation, configuration, and usage of `rsync` on OpenBSD in both client and server roles. It includes examples of common usage scenarios, secure setup, and how to operate a standalone `rsyncd` daemon.

## Installation

To install `rsync` from the OpenBSD packages collection:

```sh
# pkg_add rsync
```

This installs the `rsync` binary and associated files to `/usr/local/bin/rsync`.

## Client Usage

The most common use case for `rsync` is as a client to synchronize files to or from a local or remote system. It supports multiple transports, with `ssh(1)` being the default for remote operations.

### Syntax

```sh
rsync [options] SOURCE DEST
```

Examples are provided below.

### Local Copy

To copy a directory locally:

```sh
$ rsync -av /home/user/documents/ /backup/documents/
```

- `-a`: Archive mode (preserves permissions, symlinks, etc.)
- `-v`: Verbose output
- Trailing slash in source ensures contents are copied, not the directory itself.

### Remote Copy Over SSH

To copy files to a remote host over SSH:

```sh
$ rsync -avz /home/user/data/ user@remote.example.com:/home/user/backup/
```

To copy from a remote host:

```sh
$ rsync -avz user@remote.example.com:/var/log/ ./logs/
```

- `-z`: Enable compression
- `-e ssh`: Can be omitted; `rsync` defaults to SSH transport

To specify a non-standard SSH port:

```sh
$ rsync -av -e "ssh -p 2222" ./site/ user@remote.example.com:/var/www/
```

### Exclude Files

Exclude temporary files:

```sh
$ rsync -av --exclude '*.tmp' ./ src/ user@host:/srv/
```

To exclude a list of patterns:

```sh
$ rsync -av --exclude-from=exclude.txt ./ src/ user@host:/srv/
```

### Deleting Removed Files

Synchronize and delete files on the destination not present in the source:

```sh
$ rsync -av --delete ./syncdir/ user@host:/srv/syncdir/
```

### Dry Run

Show what would be done without performing it:

```sh
$ rsync -av --dry-run ./docs/ user@host:/backup/docs/
```

## Running an rsync Server

`rsync` can be run as a standalone daemon to provide files over the `rsync://` protocol. Unlike typical SSH-based usage, this mode does not use SSH and requires explicit configuration and activation.

### rsyncd Configuration

When running as a daemon, `rsync` reads its configuration from `/etc/rsyncd.conf`, or another file specified via the `--config` option. The format resembles an INI file, with one or more modules defined.

Example configuration:

```ini
uid = _rsync
gid = _rsync
use chroot = yes
max connections = 4
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid

[public]
    path = /srv/rsync/public
    comment = Public files
    read only = yes
    list = yes

[backup]
    path = /srv/rsync/private
    comment = Backup files
    auth users = backupuser
    secrets file = /etc/rsyncd.secrets
    read only = no
    list = no
```

This example defines two modules:

- `public`: A read-only public directory.
- `backup`: A password-protected writable module restricted to `backupuser`.

### Authentication Secrets

If any module uses `auth users`, a secrets file must be specified. This file maps usernames to passwords:

```
backupuser:password123
```

Set secure permissions on the secrets file:

```sh
# chmod 600 /etc/rsyncd.secrets
```

### Starting the Daemon

To start the daemon manually for testing:

```sh
# /usr/local/bin/rsync --daemon
```

To enable `rsyncd` at boot, the recommended approach is to install a custom `rc.d` script, as described in the next section.

## rc.d Integration

OpenBSD does not include an `rc.d` script for `rsyncd` by default. To manage the daemon using `rcctl(8)`, create a custom script at `/etc/rc.d/rsyncd`:

```sh
# vi /etc/rc.d/rsyncd
```

```sh
#!/bin/ksh
# PROVIDE: rsyncd
# REQUIRE: DAEMON
# KEYWORD: shutdown

. /etc/rc.d/rc.subr

rc_reload=NO
rc_cmd="/usr/local/bin/rsync --daemon"

rc_flags=""
rc_pre() {
    checkyesno rsyncd_flags
}

rc_start() {
    $rc_cmd
}

rc_stop() {
    pkill -xf "$rc_cmd"
}

run_rc_command "$1"
```

Set proper permissions and enable the service:

```sh
# chmod +x /etc/rc.d/rsyncd
# rcctl enable rsyncd
# rcctl start rsyncd
```

## Connecting to an rsync Daemon

To connect to a public module on an `rsync` server:

```sh
$ rsync rsync://rsync.example.com/public/
```

To synchronize with authentication:

```sh
$ rsync rsync://backupuser@rsync.example.com/backup/ --password-file=/etc/rsync.pass
```

Where `/etc/rsync.pass` contains:

```
password123
```

Ensure proper file permissions:

```sh
# chmod 600 /etc/rsync.pass
```

## Comparison with Alternatives

| Tool | Encrypted | Suitable For | Notes |
| --- | --- | --- | --- |
| `rsync` | Optional | Large file sets | Delta transfers; daemon or over SSH |
| `scp` | Yes | Simple copies | Always encrypted; no resume support |
| `sftp` | Yes | Interactive use | SSH-based file transfers |
| `ftp` | No | Anonymous access | Legacy; less secure |
| `http` | Optional | Public downloads | Efficient for static file serving |

## Security Considerations

- When using the daemon mode (`rsync://`), **no encryption** is performed.
- Always prefer SSH-based `rsync` for secure transfers.
- Protect secrets files (`rsyncd.secrets`, `--password-file`) with strict permissions.
- Use `chroot`, UID/GID restrictions, and appropriate file permissions when exposing `rsyncd`.

## Summary of Common Commands

| Command | Description |
| --- | --- |
| `pkg_add rsync` | Install `rsync` on OpenBSD |
| `rsync -avz src/ user@host:/dest/` | Copy with archive mode and compression |
| `rsync -av --delete src/ user@host:/dest/` | Sync and delete extraneous files |
| `rsync -e "ssh -p 2222"` | Use custom SSH port |
| `rsync --daemon` | Launch rsync server daemon |
| `rsync rsync://host/module/` | Connect to a remote rsync daemon |
