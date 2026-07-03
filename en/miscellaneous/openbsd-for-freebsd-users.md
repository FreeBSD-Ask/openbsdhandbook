# OpenBSD for FreeBSD Users

This quickstart introduces FreeBSD administrators to OpenBSD by mapping familiar concepts to OpenBSD tooling and conventions. It highlights practical differences; it is not an exhaustive comparison nor a discussion of philosophy. The guide assumes OpenBSD 7.8 is already installed and you have command-line access.

## Shells

On OpenBSD, the default shell for both root and regular users is the Korn shell, [ksh(1)](https://man.openbsdhandbook.com/ksh.1/)
. This differs from FreeBSD, where the root account defaults to `tcsh`. OpenBSD’s [ksh(1)](https://man.openbsdhandbook.com/ksh.1/)
implements a superset of the traditional Bourne shell language.

Alternative shells are available as packages; see [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
and [chsh(1)](https://man.openbsdhandbook.com/chsh.1/)
.

**Recommendation:** Do not change root’s shell to a package-provided shell. Non-base shells live under `/usr/local/bin`, which might be unavailable in a limited-recovery scenario. The base [ksh(1)](https://man.openbsdhandbook.com/ksh.1/)
resides in `/bin`.

```sh
# install common alternative shells (as root)
# pkg_add bash zsh

$ doas pkg_add bash zsh
$ chsh -s /usr/local/bin/zsh
```

## Privilege escalation: doas

OpenBSD provides [doas(1)](https://man.openbsdhandbook.com/doas.1/)
in the base system for privilege escalation. Configure it via [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/)
. A documented example exists in `/etc/examples/`.

```sh
$ doas cp /etc/examples/doas.conf /etc/doas.conf
  # Start from the example configuration
```

By default, each invocation prompts for a password. To cache authentication in a manner similar to some `sudo` configurations, add `persist`:

```
permit persist keepenv :wheel
```

A `sudo` package is available if required; see sudo(8).

## Software management

OpenBSD separates the base system from packages. Prefer prebuilt packages; see [packages(7)](https://man.openbsdhandbook.com/packages.7/)
. Building from ports is documented in [ports(7)](https://man.openbsdhandbook.com/ports.7/)
.

### Installing and removing packages

Install prebuilt packages with [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
:

```sh
$ doas pkg_add nginx
```

Remove packages with [pkg\_delete(1)](https://man.openbsdhandbook.com/pkg_delete.1/)
:

```sh
$ doas pkg_delete nginx
```

List installed packages with [pkg\_info(1)](https://man.openbsdhandbook.com/pkg_info.1/)
:

```sh
$ pkg_info
```

### Updating (same release)

OpenBSD ships binary patches for the **base system** via [syspatch(8)](https://man.openbsdhandbook.com/syspatch.8/)
.

```sh
$ doas syspatch -c
  # Show available base patches
$ doas syspatch
  # Apply base patches; reboot if required
```

Update installed packages to the latest for your release with:

```sh
$ doas pkg_add -Uu
  # Upgrade all packages within the current release
```

### Upgrading (to a new release)

Use [sysupgrade(8)](https://man.openbsdhandbook.com/sysupgrade.8/)
to fetch and perform a release upgrade. After the reboot, merge configuration changes with [sysmerge(8)](https://man.openbsdhandbook.com/sysmerge.8/)
if prompted, then update packages.

```sh
$ doas sysupgrade
  # Upgrade base to the next release

$ doas pkg_add -Uu
  # Update packages after the base upgrade
```

## Networking

Interface **names** are driver-based on both systems, so `em0`, `re0`, and similar names are familiar. The **configuration model** differs.

### Interface configuration with hostname.if

Per-interface configuration lives in [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
, where `if` is the interface name. For example, `/etc/hostname.em0`:

Static IPv4:

```
inet 10.0.0.100 255.255.255.0
```

Static IPv6:

```
inet6 2001:db8:6000:9344::154 64
```

DHCP:

```
dhcp
```

Temporary, runtime changes can be made with [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
:

```sh
$ doas ifconfig em0 10.0.0.100 255.255.255.0
```

Apply configuration from files using [netstart(8)](https://man.openbsdhandbook.com/netstart.8/)
:

```sh
$ doas sh /etc/netstart
  # Reload all interfaces
$ doas sh /etc/netstart em0
  # Reload a single interface
```

### Hostname

Set the system’s fully qualified domain name in [myname(5)](https://man.openbsdhandbook.com/myname.5/)
(`/etc/myname`). The name must resolve via `/etc/hosts` or DNS.

```
host.example.com
```

Reload networking with `sh /etc/netstart` after changes.

### Default gateway

Set the default gateway(s) in [mygate(5)](https://man.openbsdhandbook.com/mygate.5/)
(`/etc/mygate`). One address per line; the first of each family is used.

```
192.0.2.1
2001:db8:6000:9344::1
```

Reload networking with `sh /etc/netstart`.

### DNS resolvers

Configure resolvers in [resolv.conf(5)](https://man.openbsdhandbook.com/resolv.conf.5/)
:

```
nameserver 192.0.2.1
lookup file bind
```

Reload networking with `sh /etc/netstart`.

## Daemons and startup

OpenBSD uses the traditional BSD init and rc system; see [init(8)](https://man.openbsdhandbook.com/init.8/)
, [rc(8)](https://man.openbsdhandbook.com/rc.8/)
, and [rc.conf(8)](https://man.openbsdhandbook.com/rc.conf.8/)
. System defaults are in `/etc/rc.conf`. Do not edit it directly; override and localize settings in `/etc/rc.conf.local`.

FreeBSD administrators commonly use `service(8)` and `sysrc(8)`. On OpenBSD, use [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
to control and enable daemons.

To enable the base web server, [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
, at boot, either use `rcctl enable httpd` or set an empty flags line in `rc.conf.local`:

```conf
httpd_flags=
```

Control daemons with [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
:

```sh
$ doas rcctl start httpd # Start the httpd service immediately
$ doas rcctl stop httpd # Stop the running httpd service
$ doas rcctl reload httpd # Reload httpd configuration without a full restart
$ doas rcctl enable httpd # Enable httpd to start at boot
$ doas rcctl disable httpd # Disable automatic start at boot
```

## Common equivalents

| Task (FreeBSD) | OpenBSD tool or file | Purpose |
| --- | --- | --- |
| `pkg install nginx` | `pkg_add nginx` | Install a package from the repository |
| `pkg delete nginx` | `pkg_delete nginx` | Remove a package |
| `pkg upgrade` | `pkg_add -Uu` | Upgrade all packages in the current release |
| `freebsd-update fetch install` | `syspatch` | Apply base system patches |
| `freebsd-update -r 14.1-RELEASE upgrade` | `sysupgrade` | Upgrade to the next OpenBSD release |
| `service sshd start` | `rcctl start sshd` | Start a service |
| `sysrc sshd_enable=YES` | `rcctl enable sshd` | Enable a service at boot |
| `sysrc ifconfig_em0="DHCP"` | `/etc/hostname.em0` with `dhcp` | Configure an interface for DHCP |
| `service netif restart` | `sh /etc/netstart [if]` | Reload network configuration |
| `hostname="host.example.com"` in `/etc/rc.conf` | `/etc/myname` | Set system hostname |
| `defaultrouter="192.0.2.1"` in `/etc/rc.conf` | `/etc/mygate` | Set default gateway |

For further details, consult the referenced manual pages on this site.
