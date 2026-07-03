# Security

This chapter introduces basic privilege escalation options for OpenBSD systems. Prefer the smallest practical privilege boundary for routine administration, and reserve full root shells for tasks that require them.

OpenBSD provides [doas(1)](https://man.openbsdhandbook.com/doas.1/)
in the base system. Package-provided alternatives such as sudo are available, but they are not installed by default. Direct root login through [su(1)](https://man.openbsdhandbook.com/su.1/)
should be used sparingly because it requires the root password and grants a full root shell.

## Doas

The [doas(1)](https://man.openbsdhandbook.com/doas.1/)
utility allows a user to run commands as another user. Administrators most often use it to run selected commands as root.

The configuration file is [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/)
at `/etc/doas.conf`. This file does not exist by default. OpenBSD provides an example in `/etc/examples/`:

```
# cp /etc/examples/doas.conf /etc/
```

The default example allows members of the `wheel` group to run commands as root. Unlike many sudo configurations, `doas` asks for authentication for each command unless the `persist` option is configured.

```
# Allow wheel by default
permit persist keepenv :wheel
```

By default, `doas` runs commands as root unless another target user is specified with `-u`. Commands run through `doas` are logged to `/var/log/secure`.

To verify that `doas` works for an administrative account, run a command that requires elevated privileges, such as [vipw(8)](https://man.openbsdhandbook.com/vipw.8/)
:

```sh
$ doas vipw
```

If the account is not permitted or the configuration is incorrect, the command fails. Verify group membership with [id(1)](https://man.openbsdhandbook.com/id.1/)
or by checking the `wheel` group entry with [getent(1)](https://man.openbsdhandbook.com/getent.1/)
:

```sh
$ id
$ getent group wheel
```

More examples are available in [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/)
.

## Sudo

The sudo utility is widely used on Unix-like systems. It is available on OpenBSD as a package, but it is not part of the default installation because `doas` is available in the base system.

Install sudo with [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
:

```sh
# pkg_add sudo
```

By default, only root can use sudo. To allow members of the `wheel` group, edit `/etc/sudoers` with `visudo` and enable the appropriate rule.

The \*\*visudo\*\* command checks the syntax of `/etc/sudoers` before writing it and helps prevent misconfiguration. Use it instead of editing the file directly.

```conf
# Uncomment to allow people in group wheel to run all commands
# and set environment variables.
# %wheel        ALL=(ALL) SETENV: ALL
```

Check sudo privileges with:

```sh
$ sudo -l
```

A permitted user should see a rule similar to:

```
(ALL) SETENV: ALL
```

If the user is not in the `wheel` group, sudo reports that the user may not run sudo on the host. Add the user to `wheel` with [usermod(8)](https://man.openbsdhandbook.com/usermod.8/)
:

```
# usermod -G wheel USER
```

## Su

The [su(1)](https://man.openbsdhandbook.com/su.1/)
utility starts a shell as another user. Administrators commonly use it to become root, but this grants a full root shell and requires the root password. Prefer `doas` for routine administrative commands.

Only users in the `wheel` group may become root with `su`:

```sh
$ su -
```
