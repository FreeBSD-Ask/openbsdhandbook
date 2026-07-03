# Audit OpenSSH

## Synopsis

1. Install `ssh-audit` with [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
   .
2. Run `ssh-audit` against the target to inventory supported algorithms.
3. Constrain key exchange, host-key, and MAC algorithms in [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)
   .
4. Validate configuration with [sshd(8)](https://man.openbsdhandbook.com/sshd.8/)
   and restart via [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
   .
5. Re-run `ssh-audit` to confirm the intended policy.
6. Optionally remove small Diffie–Hellman moduli per [moduli(5)](https://man.openbsdhandbook.com/moduli.5/)
   .
7. Optionally rotate host keys with [ssh-keygen(1)](https://man.openbsdhandbook.com/ssh-keygen.1/)
   .

## Overview

This chapter describes how to assess and harden an OpenSSH server on OpenBSD using the `ssh-audit` utility. The workflow is to install and run `ssh-audit`, constrain algorithms in [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)
, validate the daemon with [sshd(8)](https://man.openbsdhandbook.com/sshd.8/)
, restart via [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
, and optionally curate Diffie–Hellman moduli per [moduli(5)](https://man.openbsdhandbook.com/moduli.5/)
and rotate host keys with [ssh-keygen(1)](https://man.openbsdhandbook.com/ssh-keygen.1/)
.

A **key exchange algorithm (KEX)** establishes shared secrets. A **host-key algorithm** authenticates the server identity. A **message authentication code (MAC)** authenticates message integrity. Restricting these sets reduces attack surface while maintaining required client compatibility.

## Install and Run ssh-audit

Install the package with [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
and perform a local audit.

```sh
# pkg_add ssh-audit
$ ssh-audit localhost
```

To audit a remote host, specify the target and optional port. IPv6 literals require brackets.

```sh
$ ssh-audit example.org
$ ssh-audit -p 2222 example.org
$ ssh-audit -p 22 '[2001:db8::1]'
```

The utility reports the server banner and the offered KEX, host keys, ciphers, and MACs, highlighting weak or deprecated choices.

## Harden Algorithm Policy in sshd\_config

Edit `/etc/ssh/sshd_config` to permit a concise, modern subset of algorithms. The commands below append conservative settings suitable for a wide range of clients. Review and adjust to match the deployment’s compatibility requirements. See [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)
for directive semantics.

**Key exchange.** Prefer Curve25519 and strong finite-field Diffie–Hellman groups. The hybrid `sntrup761x25519-sha512@openssh.com` may be included where supported by both server and clients.

```sh
# sh -c 'printf "\n# Restrict key exchange\nKexAlgorithms \
curve25519-sha256,curve25519-sha256@libssh.org,\
diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,\
diffie-hellman-group-exchange-sha256\n" >> /etc/ssh/sshd_config'
```

**MACs.** Prefer encrypt-then-MAC variants and avoid SHA-1.

```sh
# sh -c 'printf "# Restrict MACs\nMACs \
umac-128-etm@openssh.com,hmac-sha2-256-etm@openssh.com,\
hmac-sha2-512-etm@openssh.com\n" >> /etc/ssh/sshd_config'
```

**Host keys.** Prefer Ed25519 and RSA with SHA-2 signatures.

```sh
# sh -c 'printf "# Restrict HostKey algorithms\nHostKeyAlgorithms \
ssh-ed25519,rsa-sha2-256,rsa-sha2-512\n" >> /etc/ssh/sshd_config'
```

When reducing `HostKeyAlgorithms`, ensure that `HostKey` file paths in `/etc/ssh/sshd_config` reference only the retained key types. For example, omit `ssh_host_ecdsa_key` when ECDSA is disabled. See [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)
.

## Validate and Apply the Configuration

Test the daemon configuration and restart the service. Use [sshd(8)](https://man.openbsdhandbook.com/sshd.8/)
with `-t` to validate settings and [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
to restart.

```sh
# sshd -t
# rcctl restart sshd
```

Re-run the audit to confirm the intended algorithm set is offered.

```sh
$ ssh-audit localhost
```

## Optional: Remove Small Diffie–Hellman Moduli

Group-exchange parameters reside in `/etc/ssh/moduli`. Filter out entries with size below 3071 bits to target at least approximately 128-bit strength. Refer to [moduli(5)](https://man.openbsdhandbook.com/moduli.5/)
for field details.

```sh
# awk '$5 >= 3071' /etc/ssh/moduli > /etc/ssh/moduli.safe
# mv -f /etc/ssh/moduli.safe /etc/ssh/moduli
# rcctl restart sshd
```

## Optional: Rotate Host Keys

When deprecating algorithms or strengthening parameters, rotate host keys. Back up existing keys, remove unwanted types, and generate Ed25519 and RSA (3072-bit) keys. Use [ssh-keygen(1)](https://man.openbsdhandbook.com/ssh-keygen.1/)
.

```sh
# cd /etc/ssh
# install -d -m 0700 ./backup-keys && mv ssh_host_* ./backup-keys/
# ssh-keygen -t ed25519 -N '' -f /etc/ssh/ssh_host_ed25519_key
# ssh-keygen -t rsa -b 3072 -o -a 100 -N '' -f /etc/ssh/ssh_host_rsa_key
```

Ensure that `HostKey` directives reference the new files:

```
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key
```

Validate and restart:

```sh
# sshd -t
# rcctl restart sshd
```

## Client Compatibility Considerations

Tightening policies can break older clients that lack support for Curve25519 or RSA SHA-2 signatures. Where legacy access is unavoidable, conditional match blocks in `sshd_config` can scope exceptions, or a separate bastion with a relaxed policy can be provided. See [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)
and [ssh(1)](https://man.openbsdhandbook.com/ssh.1/)
.

## Verification

Use `ssh-audit` to confirm removal of weak items such as SHA-1 MACs, small DH groups, or disabled host-key types. Monitor authentication and negotiation logs via the system’s [syslog(3)](https://man.openbsdhandbook.com/syslog.3/)
facilities after deployment.
