# OpenBSD for macOS Users

This quickstart introduces macOS administrators to OpenBSD 7.8 by mapping familiar concepts to OpenBSD tooling and conventions. It highlights practical differences; it is not an exhaustive comparison nor a discussion of philosophy. The guide assumes OpenBSD is already installed and you have command-line access.

## Shells

On OpenBSD, the default shell for both root and regular users is the Korn shell, [ksh(1)](https://man.openbsdhandbook.com/ksh.1/)
. On modern macOS systems, new user accounts default to `zsh`. OpenBSD’s [ksh(1)](https://man.openbsdhandbook.com/ksh.1/)
implements a superset of the traditional Bourne shell language.

Alternative shells (e.g., Bash, Zsh) are available as packages; see [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
and [chsh(1)](https://man.openbsdhandbook.com/chsh.1/)
.

**Recommendation:** Do not change root’s shell to a package-provided shell. Non-base shells live under `/usr/local/bin`, which may be unavailable in a limited-recovery scenario. The base [ksh(1)](https://man.openbsdhandbook.com/ksh.1/)
resides in `/bin`.

```sh
# Install common alternative shells (from a regular account via doas(1))
$ doas pkg_add bash zsh  # Install Bash and Zsh
$ chsh -s /usr/local/bin/zsh  # Change your login shell to Zsh
```

## Privilege escalation: doas (instead of macOS sudo)

OpenBSD provides [doas(1)](https://man.openbsdhandbook.com/doas.1/)
in the base system for privilege escalation (macOS ships `sudo`). Configure OpenBSD’s tool via [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/)
. A documented example exists in `/etc/examples/`.

```sh
$ doas cp /etc/examples/doas.conf /etc/doas.conf  # Start from the example configuration
```

By default, each invocation prompts for a password. To cache authentication similarly to many sudo configurations, add `persist`:

```
permit persist keepenv :wheel
```

A `sudo` package is available if required; see sudo(8).

## Software management

Unlike macOS, where third-party package managers such as Homebrew or MacPorts are common, OpenBSD provides an official, integrated binary package system. Prefer prebuilt packages; see [packages(7)](https://man.openbsdhandbook.com/packages.7/)
. Building from ports is documented in [ports(7)](https://man.openbsdhandbook.com/ports.7/)
.

### Installing and removing packages

Install prebuilt packages with [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
:

```sh
$ doas pkg_add nginx  # Install Nginx
```

Remove packages with [pkg\_delete(1)](https://man.openbsdhandbook.com/pkg_delete.1/)
:

```sh
$ doas pkg_delete nginx  # Remove Nginx
```

List installed packages with [pkg\_info(1)](https://man.openbsdhandbook.com/pkg_info.1/)
:

```sh
$ pkg_info  # List installed packages
```

### Updating (same release)

OpenBSD ships binary patches for the **base system** via [syspatch(8)](https://man.openbsdhandbook.com/syspatch.8/)
. This is broadly analogous to macOS Software Update for the operating system, not third-party software.

```sh
$ doas syspatch -c  # Show available base patches
$ doas syspatch     # Apply base patches (reboot if required)
```

Update installed packages to the latest for your release:

```sh
$ doas pkg_add -Uu  # Upgrade all packages within the current release
```

### Upgrading (to a new release)

Use [sysupgrade(8)](https://man.openbsdhandbook.com/sysupgrade.8/)
to fetch and perform a release upgrade. After the reboot, complete any configuration merges with [sysmerge(8)](https://man.openbsdhandbook.com/sysmerge.8/)
if prompted, then update packages.

```sh
$ doas sysupgrade  # Upgrade base to the next OpenBSD release
$ doas pkg_add -Uu # Update packages after the base upgrade
```

## Networking

On macOS, interfaces often appear as `en0`, `en1`, and so on. OpenBSD names interfaces by driver, for example `em0` (Intel), `re0` (Realtek), and `bge0` (Broadcom). See [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
for details.

### Interface configuration with hostname.if

Per-interface configuration lives in [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
, where `if` is the interface name (e.g., `/etc/hostname.em0`). This replaces macOS command-line tools such as `networksetup` and `scutil` for persistent settings.

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

Apply configuration from files using [netstart(8)](https://man.openbsdhandbook.com/netstart.8/)
:

```sh
$ doas sh /etc/netstart      # Reload all interfaces
$ doas sh /etc/netstart em0  # Reload a single interface
```

Temporary, runtime changes can be made with [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
:

```sh
$ doas ifconfig em0 10.0.0.100 255.255.255.0  # Set a temporary IPv4 address
```

### Hostname

Set the system’s fully qualified domain name in [myname(5)](https://man.openbsdhandbook.com/myname.5/)
(`/etc/myname`). This replaces `scutil --set HostName` on macOS.

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
, rather than using `scutil --dns` or the System Settings UI:

```
nameserver 192.0.2.1
lookup file bind
```

Reload networking with `sh /etc/netstart`.

## Daemons and startup

macOS uses `launchd` and `launchctl` for service management. OpenBSD uses the traditional BSD init and rc system; see [init(8)](https://man.openbsdhandbook.com/init.8/)
, [rc(8)](https://man.openbsdhandbook.com/rc.8/)
, and [rc.conf(8)](https://man.openbsdhandbook.com/rc.conf.8/)
. System defaults are in `/etc/rc.conf`. Do not edit it directly; override and localize settings in `/etc/rc.conf.local`.

Control and enable daemons with [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
. For example, to manage the base web server, [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
:

```sh
$ doas rcctl start httpd    # Start httpd now (analogous to `launchctl kickstart`)
$ doas rcctl stop httpd     # Stop httpd (analogous to unloading a launchd service)
$ doas rcctl reload httpd   # Reload httpd configuration
$ doas rcctl enable httpd   # Enable start at boot (persistent)
$ doas rcctl disable httpd  # Disable start at boot
```

## Common equivalents

| Task (macOS) | OpenBSD tool or file | Purpose |
| --- | --- | --- |
| `brew install nginx` / `port install nginx` | `pkg_add nginx` | Install a package from the repository |
| `brew upgrade` / `port upgrade outdated` | `pkg_add -Uu` | Upgrade all packages in the release |
| `softwareupdate -ia` | `syspatch` | Apply base system security/bug patches |
| Major macOS upgrade (Installer / MDM) | `sysupgrade` | Upgrade to the next OpenBSD release |
| `launchctl kickstart gui/UID/label` | `rcctl start svc` | Start a service |
| `launchctl bootout system/label` | `rcctl stop svc` | Stop a service |
| `launchctl enable system/label` | `rcctl enable svc` | Enable service at boot |
| `scutil --set HostName host.example.com` | `/etc/myname` | Set system hostname |
| `networksetup -setdhcp Wi-Fi` | `/etc/hostname.em0` with `dhcp` | Configure an interface for DHCP |
| View DNS: `scutil --dns` | `/etc/resolv.conf` | DNS resolver configuration |

For further details, consult the referenced manual pages on this site.
