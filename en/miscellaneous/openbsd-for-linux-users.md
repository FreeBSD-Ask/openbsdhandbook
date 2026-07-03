# OpenBSD for Linux Users

This quickstart introduces Linux administrators to OpenBSD by mapping familiar concepts to OpenBSD tooling and conventions. It highlights practical differences; it is not an exhaustive comparison nor a discussion of philosophy. The guide assumes OpenBSD is already installed and you have command-line access. For installation, see the site’s installation chapter.

## Shells

The default shell for both root and regular users is the Korn shell, [ksh(1)](https://man.openbsdhandbook.com/ksh.1/)
. Its command language is a superset of the traditional Bourne shell. Shells such as Bash and Zsh are available as packages; see [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
and [chsh(1)](https://man.openbsdhandbook.com/chsh.1/)
.

**Recommendation:** Do not change root’s shell to a package-provided shell. Non-base shells live under `/usr/local/bin`, which may be unavailable in a limited-recovery scenario. The base [ksh(1)](https://man.openbsdhandbook.com/ksh.1/)
resides in `/bin`.

```sh
# install common alternative shells (as root)
# use doas(1) from a regular account; see the next section
# pkg_add bash zsh

$ doas pkg_add bash zsh
$ chsh -s /usr/local/bin/bash
```

## Privilege escalation: doas (instead of sudo)

OpenBSD provides [doas(1)](https://man.openbsdhandbook.com/doas.1/)
for privilege escalation. Configure it via [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/)
. A sample file exists in `/etc/examples/`.

```sh
$ doas cp /etc/examples/doas.conf /etc/doas.conf
  # Start from the documented example
```

By default, each invocation prompts for a password. To cache authentication similarly to many `sudo` setups, use the `persist` option:

```
permit persist keepenv :wheel
```

A `sudo` package is available if required; see sudo(8). In most workflows, `doas` fulfills the same role with simpler configuration.

## Software management

OpenBSD separates the base system from packages. Package tools are documented in [packages(7)](https://man.openbsdhandbook.com/packages.7/)
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
. Use it regularly:

```sh
$ doas syspatch -c
  # Show available base patches
$ doas syspatch
  # Apply base patches and reboot if required
```

Packages are updated independently within a release. To update installed packages to the latest for your release, use:

```sh
$ doas pkg_add -Uu
  # Upgrade all packages within the current release
```

### Upgrading (to a new release)

OpenBSD targets approximately two releases per year. Use [sysupgrade(8)](https://man.openbsdhandbook.com/sysupgrade.8/)
to fetch and perform a release upgrade:

```sh
$ doas sysupgrade
```

After the base upgrade and reboot, update packages:

```sh
$ doas pkg_add -Uu
```

## Networking

OpenBSD names network interfaces by driver name rather than `eth0`, `enp0s1`, and so on. Examples include `re0` (Realtek), `bge0` (Broadcom), and `em0` (Intel). See [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
for details.

### Interface configuration with hostname.if

Per-interface configuration lives in [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
, where `if` is the interface name. For example, `/etc/hostname.re0`:

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
$ doas ifconfig re0 10.0.0.100 255.255.255.0
```

Apply configuration from files using the same script used at boot, [netstart(8)](https://man.openbsdhandbook.com/netstart.8/)
:

```sh
$ doas sh /etc/netstart
  # Reload all interfaces
$ doas sh /etc/netstart re0
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

OpenBSD uses the traditional BSD init and rc system: see [init(8)](https://man.openbsdhandbook.com/init.8/)
, [rc(8)](https://man.openbsdhandbook.com/rc.8/)
, and [rc.conf(8)](https://man.openbsdhandbook.com/rc.conf.8/)
. There are no SysV-style run levels.

System defaults are in `/etc/rc.conf`. Do not edit it directly; override and localize settings in `/etc/rc.conf.local`.

To enable the base web server, [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
, at boot, either use [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
or set an empty flags line in `rc.conf.local`:

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

## Common command equivalents

| Linux command (RPM/DPKG) | OpenBSD tool | Purpose |
| --- | --- | --- |
| `yum install` / `apt-get install pkg` | `pkg_add pkg` | Install package from repository |
| `rpm -i pkg.rpm` / `dpkg -i pkg.deb` | `pkg_add pkg.tgz` | Install local package |
| `rpm -qa` / `dpkg -l` | `pkg_info` | List installed packages |
| `lspci` | `pcidump` | List PCI devices |
| `lsusb` | `usbdevs` | List USB devices |

See [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
, [pkg\_info(1)](https://man.openbsdhandbook.com/pkg_info.1/)
, [pcidump(8)](https://man.openbsdhandbook.com/pcidump.8/)
, and [usbdevs(8)](https://man.openbsdhandbook.com/usbdevs.8/)
for details.
