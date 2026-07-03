# ftpd

## Synopsis

OpenBSD includes full support for the File Transfer Protocol (FTP), including both a command-line client (`ftp(1)`) and a simple, secure FTP server (`ftpd(8)`). While FTP is largely replaced by secure alternatives such as `sftp` (via `sshd`) and `httpd` for file downloads, it remains useful for compatibility with legacy systems, automation tasks, and minimal environments.

This chapter documents the use of both the FTP client and server on OpenBSD. It also compares FTP to its alternatives, describes active versus passive modes, and provides examples of common operations.

## Features

- `ftp(1)` supports both FTP and HTTP downloads
- Supports active and passive modes
- Handles anonymous and authenticated access
- Secure default configuration of `ftpd(8)` with chroot
- Suitable for automation and scripting
- Included in the OpenBSD base system

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

## FTP Client

The base system includes `ftp(1)`, a secure and script-friendly command-line utility capable of retrieving files over both FTP and HTTP. It supports passive and active transfer modes, URL-based invocation, HTTP redirection, and proxy support.

### Basic Usage

To download a file via FTP:

```sh
$ ftp ftp://ftp.openbsd.org/pub/OpenBSD/7.8/README
```

To download a file via HTTP:

```sh
$ ftp https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/install78.img
```

To download and rename:

```sh
$ ftp -o obsd.img https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/install78.img
```

To download multiple files with a glob expression:

```sh
$ ftp ftp://ftp.example.com/pub/*.txt
```

Use quotes to prevent shell globbing.

### Interactive Mode

When invoked without a URL, `ftp(1)` enters interactive mode:

```sh
$ ftp
ftp> open ftp.openbsd.org
ftp> login anonymous
ftp> cd pub/OpenBSD/7.8/amd64
ftp> get install78.img
ftp> quit
```

In this mode, standard FTP commands are accepted, such as `cd`, `get`, `put`, `ls`, and `bye`.

### Common Flags

- `-p`: Enforce passive mode (default)
- `-A`: Use active mode
- `-o <filename>`: Write output to a specific file
- `-M`: Mirror a directory
- `-R`: Automatically retry failed transfers
- `-V`: Verbose mode

### Active vs Passive Mode

FTP supports two modes for establishing data connections: **active** and **passive**. The choice impacts firewall and NAT compatibility.

#### Active Mode

In active mode, the **client tells the server** which IP address and port to connect back to. This is often blocked by client-side firewalls or NAT devices.

Flow:

1. Client connects to server on TCP port 21 (control connection).
2. Client issues `PORT` command, telling server its own IP and port.
3. Server opens a new connection back to the client (data connection).

```sh
$ ftp -A ftp://ftp.example.com
```

#### Passive Mode

In passive mode, the **server tells the client** which port to connect to. The client initiates both control and data connections. This is the default in OpenBSD.

Flow:

1. Client connects to server on port 21.
2. Client issues `PASV` command.
3. Server replies with IP and port.
4. Client connects to the server’s port (data connection).

```sh
$ ftp -p ftp://ftp.example.com
```

Passive mode is generally recommended unless a specific server requires active mode.

### Proxy Support

To use `ftp(1)` behind a proxy:

Set the `FTP_PROXY` or `HTTP_PROXY` environment variable:

```sh
$ export FTP_PROXY=http://proxy.example.com:8080
$ ftp ftp://ftp.example.com
```

### Scripting Downloads

Example download script:

```sh
#!/bin/sh
URL="https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/install78.img"
OUT="install78.img"

ftp -o "$OUT" "$URL"
```

### HTTP Authentication

For password-protected HTTP URLs:

```sh
$ ftp https://user:pass@example.com/private/file.tar.gz
```

Credentials are visible in the process list. Use with care.

## FTP Server

The base system includes `ftpd(8)`, a secure FTP daemon suitable for basic file serving in trusted networks or chrooted environments. The server supports both anonymous and authenticated sessions.

### Enabling the Server

To enable and start the FTP daemon:

```sh
# rcctl enable ftpd
# rcctl start ftpd
```

### Configuration Options

The default behavior requires no configuration file. Behavior is controlled by flags passed via `rcctl(8)`.

View current settings:

```sh
# rcctl get ftpd flags
```

Set to allow anonymous access and chroot:

```sh
# rcctl set ftpd flags -A
```

Start the daemon:

```sh
# rcctl restart ftpd
```

#### Common Flags

- `-A`: Enable anonymous FTP
- `-D`: Do not detach (useful for debugging)
- `-S`: Log all session activity
- `-l`: Log all files transferred

### Anonymous FTP

To enable anonymous access:

1. Create the anonymous home directory:

   ```sh
   # mkdir -p /home/ftp/pub
   # chown root:wheel /home/ftp
   # chmod 755 /home/ftp
   ```
2. Place files under `/home/ftp/pub`.
3. Start `ftpd` with the `-A` flag:

   ```sh
   # rcctl set ftpd flags -A
   # rcctl restart ftpd
   ```

Users may now connect with:

```sh
$ ftp ftp://hostname/pub/
```

Anonymous users are chrooted to `/home/ftp`.

### Authenticated FTP

FTP logins use system accounts. Grant shell-less access by setting the shell to `/sbin/nologin`:

```sh
# useradd -s /sbin/nologin ftpuser
# passwd ftpuser
```

Then place files under the user’s home directory.

### Logging

FTP logs appear in `/var/log/messages`. Enable session logging via the `-S` flag:

```sh
# rcctl set ftpd flags "-A -S"
# rcctl restart ftpd
```

## Alternatives to FTP

| Protocol | Tool | Encrypted | Notes |
| --- | --- | --- | --- |
| SFTP | `sftp(1)` | Yes | Uses SSH; preferred alternative |
| HTTP | `httpd(8)` | Optional | Easy to serve files via HTTP |
| SCP | `scp(1)` | Yes | Simple SSH-based file copy |
| rsync | `rsync` | Optional | Available via packages, efficient |

Use FTP only when required for legacy or compatibility purposes. Otherwise, prefer encrypted protocols such as `sftp`.

## Summary of Key Commands

| Command | Description |
| --- | --- |
| `ftp ftp://...` | Retrieve file over FTP (client) |
| `ftp -o file URL` | Save to specific filename |
| `rcctl enable ftpd` | Enable FTP daemon |
| `rcctl set ftpd flags ...` | Set server options |
| `rcctl start ftpd` | Start FTP daemon |
| `rcctl restart ftpd` | Apply new configuration |

## Security Considerations

FTP is an insecure protocol by design, transmitting credentials and data in plaintext. For external use or untrusted networks, use `sftp` or another encrypted transport.

FTP may be acceptable for:

- Internal or air-gapped networks
- Public file distribution (anonymous-only)
- Specific automation tasks with legacy systems
