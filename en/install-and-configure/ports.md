# Managing Software: Packages and Ports

## Synopsis

This chapter offers a detailed guide to managing software on OpenBSD, focusing on installing and maintaining pre-built packages with `pkg_*` utilities and building custom applications from the Ports Collection using `ports(7)`. It also covers adding file sets after installation.

## Overview

OpenBSD provides two primary methods for installing software: pre-compiled binary packages and building from source using the Ports Collection. Packages are convenient, fast to install, and receive timely updates for critical software. The Ports Collection allows custom builds but requires more time, disk space, and attention to system resources.

## Using the Package System

OpenBSD packages are compressed and signed binary archives managed by the `pkg_*` suite of tools. They are easy to install and recommended for most use cases.

### Selecting a Mirror

The system uses the mirror listed in `/etc/installurl` by default. To configure it manually:

```
https://cdn.openbsd.org/pub/OpenBSD
```

Alternatively, use the `PKG_PATH` environment variable (not recommended unless scripting or debugging):

```sh
export PKG_PATH=https://cdn.openbsd.org/pub/OpenBSD/7.8/packages/amd64/
```

Replace `7.8` with the system release version. Use `uname -r` for verification.

See `installurl(5)` and `pkg_add(1)` for further options.

### Installing Packages

Use `pkg_add(1)` to install software:

```sh
doas pkg_add vim
```

If a package has multiple flavors or subpackages, the system will prompt:

```
Ambiguous: choose package for rsync
a       0: <None>
        1: rsync-3.1.2p0
        2: rsync-3.1.2p0-iconv
Your choice:
```

You may enter the number, or specify the full flavor inline:

```sh
doas pkg_add rsync--iconv
```

To install directly from a URL:

```sh
doas pkg_add https://mirror.example.org/packages/vim-9.0.1234p0.tgz
```

Some packages include notes in `/usr/local/share/doc/pkg-readmes/`. Always read these if prompted.

### Searching for Packages

To search by package name:

```sh
pkg_info -Q unzip
```

To locate files within packages:

```sh
pkglocate mutool
```

This requires the `pkglocatedb` package to be installed.

### Updating Packages

To upgrade all installed packages:

```sh
doas pkg_add -Uu
```

To update a specific package:

```sh
doas pkg_add -u unzip
```

OpenBSD rebuilds selected packages such as Firefox for supported releases. These updates are available using `pkg\_add -Uu`.

## Updating the System

### Base System Updates

OpenBSD provides signed binary patches for the base system via the `syspatch(8)` utility. These patches are available for supported `-release` versions and are typically issued for security vulnerabilities or other important fixes. To apply all available patches:

```sh
doas syspatch
```

It is recommended to run `syspatch` periodically to keep the base system up to date.

### Package Updates

Packages can be updated independently of the base system. The following command updates all installed packages:

```sh
doas pkg_add -Uu
```

This command contacts the configured package mirror and upgrades installed packages to newer versions if available. Use this command regularly to receive updates for supported packages, such as browsers or mail clients.

## Removing Packages

Use `pkg_delete(1)` to remove software:

```sh
doas pkg_delete screen
```

To remove unused dependencies:

```sh
doas pkg_delete -a
```

### Duplicating Package Lists

To duplicate a package set to another machine:

```sh
pkg_info -mz > list
doas pkg_add -l list
```

### Handling Incomplete or Broken Packages

If a package install fails and leaves a `partial-*` entry, clean it with:

```sh
doas pkg_delete partial-vim
```

If package metadata becomes corrupted:

```sh
doas pkg_check
```

## Using the Ports Collection

The Ports Collection allows building software from source with custom options.

### Installing the Ports Tree

To fetch the tree:

```sh
doas cvs -qd anoncvs@anoncvs.openbsd.org:/cvs get -rOPENBSD_7_6 -P ports
```

To update:

```sh
doas cd /usr/ports && cvs update -Pd
```

### Building a Port

```sh
cd /usr/ports/editors/vim
doas make install
```

For configuration options:

```sh
make config
```

### Managing Dependencies

List required packages:

```sh
make show-DEPENDS
```

Install them manually:

```sh
doas pkg_add $(make show-DEPENDS)
```

### Custom Builds

To build a version of `vim` without X11:

```sh
cd /usr/ports/editors/vim
env NO_X11=Yes doas make install
```

Building ports may take hours and consume significant disk space. Use binary packages unless custom compilation is necessary.

## Adding File Sets After Installation

File sets such as `comp{{ .Site.Params.short_version }}.tgz` or `xbase{{ .Site.Params.short_version }}.tgz` can be added after initial installation.

### Method 1: Using bsd.rd and Upgrade Mode

1. Boot `bsd.rd` from the existing disk or USB.
2. Choose `(U)pgrade`.
3. Select and install missing sets.
4. Reboot into the full system.

### Method 2: Manual Extraction

```sh
cd /tmp
ftp https://cdn.openbsd.org/pub/OpenBSD/{{ .Site.Params.full_version }}/amd64/comp{{ .Site.Params.short_version }}.tgz
cd /
doas tar xzvphf /tmp/comp{{ .Site.Params.short_version }}.tgz
```

The `-p` option is required to preserve correct permissions.
