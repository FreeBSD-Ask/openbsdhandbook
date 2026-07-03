# sshd

## Synopsis

`sshd` is the OpenSSH daemon that accepts incoming SSH connections and provides encrypted remote shell access and secure file transfer. On OpenBSD, `sshd` is part of the base system and is enabled by default after installation. Typical uses include secure administration, file transfer via `sftp(1)` and `scp(1)`, remote command execution, and secure tunneling.

For service control, use [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
. For configuration, edit [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)
. The daemon benefits from [pledge(2)](https://man.openbsdhandbook.com/pledge.2/)
, [unveil(2)](https://man.openbsdhandbook.com/unveil.2/)
, and network control with [pf(4)](https://man.openbsdhandbook.com/pf.4/)
.

## Service Management

Enable at boot and start immediately with [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
:

```sh
# rcctl enable sshd
# rcctl start sshd
# rcctl check sshd
```

Host keys are generated during installation. If any are missing, create them with [ssh-keygen(1)](https://man.openbsdhandbook.com/ssh-keygen.1/)
:

```sh
# ssh-keygen -A
```

## Configuration Basics

The daemon reads `/etc/ssh/sshd_config`. One directive is permitted per line, and lines beginning with `#` are comments. Validate changes with [sshd(8)](https://man.openbsdhandbook.com/sshd.8/)
and restart via [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
.

```sh
# sshd -t
# rcctl restart sshd
```

A concise baseline configuration:

```
Port 22
AddressFamily any
ListenAddress 0.0.0.0
ListenAddress ::

PermitRootLogin no
PasswordAuthentication no
ChallengeResponseAuthentication no
PubkeyAuthentication yes

# Access control
AllowUsers admin

# Session hygiene
ClientAliveInterval 300
ClientAliveCountMax 2

# Subsystem (enabled by default)
Subsystem sftp /usr/libexec/sftp-server
```

This configuration disables root login and password-based authentication, enables public key authentication, limits access to a specific account, and sets conservative keepalive parameters. Adjust `ListenAddress` as required.

## Client Configuration

The OpenBSD base system includes the client programs [ssh(1)](https://man.openbsdhandbook.com/ssh.1/)
, [sftp(1)](https://man.openbsdhandbook.com/sftp.1/)
, and [scp(1)](https://man.openbsdhandbook.com/scp.1/)
. This section covers key creation, key installation, and common client operations. Client-side configuration is read from [ssh\_config(5)](https://man.openbsdhandbook.com/ssh_config.5/)
.

### Generate an SSH Key Pair

Create an Ed25519 key pair with an identifying comment. Use [ssh-keygen(1)](https://man.openbsdhandbook.com/ssh-keygen.1/)
. Protect the private key with a passphrase.

```sh
$ ssh-keygen -t ed25519 -C "[admin@example.com](mailto:admin@example.com)" -f \~/.ssh/id\_ed25519
```

For hardware-backed security keys (FIDO/U2F), generate an `ed25519-sk` key:

```sh
$ ssh-keygen -t ed25519-sk -C "[admin@example.com](mailto:admin@example.com)" -f \~/.ssh/id\_ed25519\_sk
```

Ensure appropriate permissions on the `.ssh` directory and files:

```sh
$ chmod 700 \~/.ssh
$ chmod 600 \~/.ssh/id\_\* \~/.ssh/id\_\*.pub
```

### Load Keys into ssh-agent (optional)

An agent can hold decrypted keys in memory. Start the agent and add the key with [ssh-agent(1)](https://man.openbsdhandbook.com/ssh-agent.1/)
and [ssh-add(1)](https://man.openbsdhandbook.com/ssh-add.1/)
.

```sh
$ eval "\$(ssh-agent -s)"
$ ssh-add \~/.ssh/id\_ed25519
```

### Install the Public Key on the Server

If the `ssh-copy-id` helper is available on the client system, it can append the public key to the remote account’s `~/.ssh/authorized_keys`. The utility is not part of OpenBSD base; when installed, usage is:

```sh
$ ssh-copy-id [admin@host.example.com](mailto:admin@host.example.com)
```

When `ssh-copy-id` is not available, append the public key over SSH using standard tools:

```sh
$ ssh [admin@host.example.com](mailto:admin@host.example.com) 'mkdir -p \~/.ssh && chmod 700 \~/.ssh && touch \~/.ssh/authorized\_keys && chmod 600 \~/.ssh/authorized\_keys'
$ cat \~/.ssh/id\_ed25519.pub | ssh [admin@host.example.com](mailto:admin@host.example.com) 'cat >> \~/.ssh/authorized\_keys'
```

To verify the first connection and record the server host key, connect once and confirm the fingerprint. For non-interactive retrieval, [ssh-keyscan(1)](https://man.openbsdhandbook.com/ssh-keyscan.1/)
can populate `~/.ssh/known_hosts`:

```sh
$ ssh-keyscan -t ed25519 host.example.com >> \~/.ssh/known\_hosts
```

### Per-Host Client Configuration

Create a host stanza in `~/.ssh/config` to define defaults for a target. See [ssh\_config(5)](https://man.openbsdhandbook.com/ssh_config.5/)
.

```
Host prod
HostName host.example.com
User admin
Port 22
IdentityFile \~/.ssh/id\_ed25519
IdentitiesOnly yes
ForwardAgent no
ForwardX11 no
```

After saving the file, connect with a short alias:

```sh
$ ssh prod
```

### Interactive Login and Remote Command Execution

Establish an interactive session or execute a single remote command with [ssh(1)](https://man.openbsdhandbook.com/ssh.1/)
.

```sh
$ ssh [admin@host.example.com](mailto:admin@host.example.com)
$ ssh [admin@host.example.com](mailto:admin@host.example.com) 'uname -a'
```

### Secure Copy (scp)

Transfer files or directories. Use `-r` for recursive copies.

```sh
$ scp ./report.txt [admin@host.example.com](mailto:admin@host.example.com):/home/admin/
$ scp -r ./site/ [admin@host.example.com](mailto:admin@host.example.com):/var/www/
$ scp [admin@host.example.com](mailto:admin@host.example.com):/var/log/daemon /tmp/daemon.log
```

### SFTP File Transfers

Open an interactive SFTP session or perform one-off transfers with [sftp(1)](https://man.openbsdhandbook.com/sftp.1/)
.

```sh
$ sftp [admin@host.example.com](mailto:admin@host.example.com)
sftp> pwd
sftp> lpwd
sftp> put ./archive.tar.gz /home/admin/
sftp> get /etc/daily /tmp/daily
sftp> mkdir /home/admin/uploads
sftp> ls -l /home/admin
sftp> exit
```

Non-interactive SFTP can transfer files directly:

```sh
$ sftp [admin@host.example.com](mailto:admin@host.example.com):/etc/daily /tmp/daily
$ sftp /var/log/messages [admin@host.example.com](mailto:admin@host.example.com):/home/admin/
```

### Known Hosts and First-Use Validation

On first connection, [ssh(1)](https://man.openbsdhandbook.com/ssh.1/)
will prompt to trust the server host key and append it to `~/.ssh/known_hosts`. To pre-validate, compare the presented fingerprint with one generated on the server:

```sh
# On the server (run as root to read host keys):

# ssh-keygen -l -f /etc/ssh/ssh\_host\_ed25519\_key.pub
```

### Common Options for Reliability and Performance

Control connection reuse and timeouts in `~/.ssh/config`:

```
Host \*
ServerAliveInterval 60
ServerAliveCountMax 2
ControlMaster auto
ControlPersist 5m
ControlPath \~/.ssh/cm-%r@%h:%p
```

`ControlMaster` enables multiplexing over a single TCP connection; subsequent sessions to the same host start faster and avoid repeated authentication.

## Hardening

### Integrate Algorithm Policy with ssh-audit

Inventory and assess offered algorithms with `ssh-audit`. See the companion chapter: [/ssh-audit](ssh-audit.md)
. Based on audit findings, restrict key exchange (KEX), host-key, and MAC algorithms in [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)
.

A conservative, broadly compatible policy:

```
# Key exchange
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,\
diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,\
diffie-hellman-group-exchange-sha256

# MACs (encrypt-then-MAC variants)
MACs umac-128-etm@openssh.com,hmac-sha2-256-etm@openssh.com,\
hmac-sha2-512-etm@openssh.com

# Host key algorithms (server authentication)
HostKeyAlgorithms ssh-ed25519,rsa-sha2-256,rsa-sha2-512
```

When reducing `HostKeyAlgorithms`, ensure `HostKey` file directives reference only retained types (for example, omit `ssh_host_ecdsa_key` if ECDSA is disabled). Validate and reload:

```sh
# sshd -t
# rcctl restart sshd
```

To curate Diffie–Hellman group-exchange parameters, filter `/etc/ssh/moduli` to entries of at least 3071 bits; see [moduli(5)](https://man.openbsdhandbook.com/moduli.5/)
.

```sh
# awk '$5 >= 3071' /etc/ssh/moduli > /etc/ssh/moduli.safe
# mv -f /etc/ssh/moduli.safe /etc/ssh/moduli
# rcctl restart sshd
```

Re-run `ssh-audit` after each change and review for unintended regressions.

### Access Control

Restrict exposure and narrow the authenticated population.

```
# Bind to explicit addresses and non-standard port if required
Port 22
ListenAddress 192.0.2.10
# ListenAddress 2001:db8::10

# Permit only specific users or groups
AllowUsers admin opsuser
# or:
# AllowGroups wheel ops
```

With [pf(4)](https://man.openbsdhandbook.com/pf.4/)
, limit ingress to trusted sources. See the PF chapters and the pfctl cheat sheet at [/pf/cheat\_sheet/](../../pf/cheat_sheet.md)
.

```pf
# Example PF fragment (context-dependent)
block in on egress proto tcp to port ssh
pass  in on egress proto tcp from 198.51.100.0/24 to port ssh
```

Reload with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
:

```sh
# pfctl -f /etc/pf.conf
```

### Authentication and Multi-Factor Options

OpenSSH supports several approaches to multi-factor authentication without external PAM modules.

1. **Public key + password (two independent factors).** Require both a private key and a password using [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)
   `AuthenticationMethods`.

   ```
   PubkeyAuthentication yes
   PasswordAuthentication yes
   AuthenticationMethods publickey,password
   ```

   This setting denies access unless both factors succeed.
2. **Security keys (FIDO/U2F).** Security-key-backed keys (for example, `sk-ssh-ed25519@openssh.com`) provide possession-plus-touch verification. Generate on the client with [ssh-keygen(1)](https://man.openbsdhandbook.com/ssh-keygen.1/)
   :

   ```sh
   $ ssh-keygen -t ed25519-sk -f ~/.ssh/id_ed25519_sk
   ```

   Ensure that server policy allows the corresponding public key algorithm (do not exclude `sk-ssh-ed25519@openssh.com` if `PubkeyAcceptedAlgorithms` is restricted) and that public key authentication remains enabled.
3. **Keyboard-interactive second factors via BSD Authentication.** OpenBSD uses bsd\_auth(3) rather than PAM. Additional BSD Auth backends can supply one-time codes (for example, TOTP) via `keyboard-interactive`. After installing and configuring an appropriate backend, require a second factor with:

   ```
   KbdInteractiveAuthentication yes
   AuthenticationMethods publickey,keyboard-interactive
   ```

   Backend selection and provisioning are backend-specific and outside the scope of this chapter.

Additional hardening measures:

```
# Reduce brute-force surface
MaxAuthTries 3
LoginGraceTime 30
MaxStartups 10:30:100

# Disable features not required
PermitTunnel no
AllowAgentForwarding no
AllowTcpForwarding no
X11Forwarding no
GatewayPorts no
PrintMotd no
UseDNS no
```

### Subsystem Isolation and Chrooted SFTP

Constrain SFTP-only accounts with a chroot and the internal SFTP subsystem:

```
Match User sftpjailed
    ChrootDirectory /home/sftpjailed
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

The chroot directory must be owned by `root` and must not be writable by the target account; see details in [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)
.

## Monitoring and Troubleshooting

Check service status and listen sockets:

```sh
# rcctl check sshd
# sockstat -4 -6 | grep sshd
```

Inspect authentication logs (facility `auth`):

```sh
# tail -f /var/log/authlog
```

Common issues include syntax errors (`sshd -t`), incorrect permissions on `~/.ssh` or `authorized_keys`, missing public keys, or firewall policy blocking the port.

Fix user key permissions:

```sh
$ chmod 700 ~/.ssh
$ chmod 600 ~/.ssh/authorized_keys
$ chown "$USER":"$USER" ~/.ssh ~/.ssh/authorized_keys
```
