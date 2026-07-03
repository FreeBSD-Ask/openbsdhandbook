# Updating and Upgrading

## Synopsis

Maintaining an OpenBSD system involves several distinct tasks: applying binary patches to the base system, updating installed packages, upgrading to a new release, and optionally tracking the development branch (`-current`). OpenBSD provides a set of reliable, clearly defined tools for each of these operations.

This chapter describes the procedures for:

- Applying base system security and stability patches using `syspatch(8)`
- Upgrading installed packages with `pkg_add -u`
- Applying firmware updates using `fw_update(8)`
- Performing in-place system upgrades with `sysupgrade(8)`
- Switching to and maintaining the `-current` development branch

Each tool addresses a specific layer of the system. Binary patches (`syspatch`) cover the base system only; they do not affect installed packages. Similarly, package upgrades do not modify the kernel or core system libraries. Full system upgrades are handled separately with `sysupgrade`.

Systems that track `-current` are updated through manual builds or frequent reinstallation, and are intended for developers and experienced users.

The following sections provide detailed procedures and examples for each update and upgrade method.

## Base System Patching with `syspatch`

The OpenBSD base system is distributed as a cohesive set of files compiled from audited source code. For released versions (e.g., 7.8), security and stability fixes are issued as binary patches. These are applied using the `syspatch(8)` utility.

The patch mechanism affects only the **base system**, not user-installed packages. Patches are provided for supported releases and are downloaded directly from the OpenBSD mirrors.

### Applying Patches

To fetch and apply all available binary patches for the current release:

```sh
# syspatch
```

This command:

- Downloads missing patches from the official mirror
- Applies them in the correct order
- Logs changes to `/var/syspatch/`

It is safe to run `syspatch` at any time; already-applied patches are skipped automatically.

### Checking for Available Patches

To check whether any patches are available **without applying them**, use the `-c` (check) flag:

```sh
# syspatch -c
```

This displays a list of unapplied patches, if any, and exits with a non-zero status if patches are needed. This mode is useful for scripted or automated maintenance tasks.

### Viewing Installed Patches

To list all currently applied patches:

```sh
# syspatch -l
```

Each entry corresponds to a binary patch matching an official erratum. Patches are stored under `/var/syspatch/`.

### Removing a Patch

To remove a previously applied patch (not generally recommended):

```sh
# syspatch -r patch_name
```

Use this only when advised, such as during troubleshooting. Removing patches may lead to an unsupported or vulnerable system state.

### Rebooting After Patches

If a patch modifies the kernel or other early-boot components, `syspatch` will display:

```
Relinking to create unique kernel... done; reboot to load the new kernel
```

This message indicates that a reboot is required for the patch to take effect. Other patches (e.g., userland binaries or daemons) may not require a reboot.

It is safe to reboot after any `syspatch` run if unsure:

```sh
# reboot
```

### Availability and Support

Patching with `syspatch` is supported only for release versions of OpenBSD. It is not used on systems running `-current` or `-stable`. Those systems must be updated via manual builds or full system upgrades.

## Upgrading Installed Packages with `pkg_add -u`

The OpenBSD package system is updated independently of the base system. Packages are updated continuously for each supported release and are distributed via the OpenBSD mirror network.

To upgrade all installed packages to their latest versions, use `pkg_add(1)` with the `-u` option. This process does not affect base system files installed from release sets such as `baseXX.tgz`.

### Upgrading All Packages

To upgrade all packages currently installed:

```sh
# pkg_add -u
```

This command:

- Compares installed package versions with the latest available on the mirror
- Downloads and installs updated packages
- Automatically removes or replaces obsolete dependencies

A working network connection and a valid package path are required.

### Confirming Package Path

If the `PKG_PATH` environment variable is not set, `pkg_add` will consult `/etc/pkg.conf`. To verify:

```sh
# grep '^installpath' /etc/pkg.conf
```

To manually set the path:

```sh
# export PKG_PATH=https://cdn.openbsd.org/pub/OpenBSD/$(uname -r)/packages/$(arch -s)/
```

This is useful in non-default configurations or scripting environments.

### Interactive Prompts

If multiple versions of a package are available, `pkg_add` may prompt for a selection:

```
ambiguous: choose package for ImageMagick
 a 0: <ImageMagick-6.9.12p4>
 a 1: <ImageMagick-7.0.10p0>
Your choice:
```

Select the appropriate version, or press Enter to accept the default.

### Packages Requiring Restart

When upgrading packages that provide services or daemons, it may be necessary to restart them:

```sh
# rcctl restart unbound
```

You may review `/var/db/pkg` or system logs to identify upgraded services.

### Verifying Package State

To verify consistency of the installed package set:

```sh
# pkg_check
```

This command checks dependencies, shared library links, and recorded install states. It does not modify the system.

Upgrading packages with `pkg_add -u` is safe to perform on live systems and does not require a reboot unless a critical library or component is replaced. The base system must be updated separately using `syspatch(8)` or `sysupgrade(8)`.

## Firmware Updates with `fw_update`

Some devices require proprietary firmware files in order to function correctly. These files are not included in the OpenBSD installation sets due to licensing restrictions, but can be installed using `fw_update(8)` after the system is installed.

Firmware is typically required for wireless adapters, USB devices, and other hardware with embedded microcontrollers.

### Installing Firmware

To install all required firmware files for the currently running system:

```sh
# fw_update
```

This will:

- Determine which firmware files are needed
- Fetch them from the OpenBSD firmware mirror
- Install them in `/etc/firmware/`
- Update `/var/db/fw_update.status` to track the installed files

Firmware installation requires a working network connection.

### Updating Firmware After an Upgrade

After a system upgrade (via `sysupgrade` or manual installation), firmware files must be updated to match the new kernel. This is handled automatically at first boot: `fw_update` is invoked by the system if any required firmware is missing or incompatible.

To manually ensure that all firmware is current:

```sh
# fw_update
```

This step is safe to repeat at any time and ensures that devices such as wireless adapters and USB peripherals function correctly under the new system version.

### Verifying Installed Firmware

Installed firmware files are located in:

- `/etc/firmware/` — contains binary firmware blobs
- `/var/db/fw_update.status` — tracks installed firmware and versions

To list the installed firmware:

```sh
# ls /etc/firmware/
```

Firmware is loaded automatically by device drivers at boot time or when the device is attached.

### Removing Unused Firmware

To remove firmware that is no longer needed:

```sh
# fw_update -c
```

This command compares installed firmware against the current hardware configuration and offers to delete unused files.

Removal is optional and can be skipped if the system is used across multiple hardware profiles.

Firmware management is essential on systems that rely on devices such as wireless cards (`iwn(4)`, `iwx(4)`, `urtwn(4)`, etc.) or other peripherals requiring binary blobs to operate.

## System Upgrades with `sysupgrade`

The OpenBSD base system is released twice per year. Upgrading from one release to the next (e.g., from 7.7 to 7.8) can be performed automatically using the `sysupgrade(8)` utility. This tool handles all steps required to perform a safe, in-place upgrade to the next official release.

`sysupgrade` is designed for systems tracking the OpenBSD **release branch**. It is not intended for upgrading `-current` systems or for applying base system patches (use `syspatch` for that purpose).

### Performing an Upgrade

To upgrade to the next available release:

```sh
# sysupgrade
```

This command performs the following operations:

1. Downloads the install sets for the new release from the nearest mirror
2. Verifies their integrity
3. Reboots into a temporary upgrade kernel
4. Installs the new sets in place
5. Reboots into the upgraded system

During this process, no manual interaction is required unless custom disk layouts or system configurations are in use. A reboot is always required to complete the upgrade.

### Upgrading to a Specific Version

By default, `sysupgrade` installs the latest available release. To upgrade to a specific version:

```sh
# sysupgrade -R 7.8
```

This is useful when performing staged upgrades across multiple versions, or to align with site-wide release policies.

### Post-Upgrade Actions

After the system boots into the new release, the following maintenance tasks are triggered automatically:

- `fw_update(8)` runs to install or update firmware matching the new kernel
- `syspatch(8)` applies any binary patches released since the new version’s publication
- `sysmerge(8)` is run to reconcile configuration files in `/etc`

During `sysmerge`, the system may prompt for review of modified files. Local customizations can be preserved or merged with updated defaults as needed.

Third-party packages installed via the ports system are **not** upgraded automatically. To bring packages in line with the new release, run:

```sh
# pkg_add -u
```

It is strongly recommended to upgrade packages after a system upgrade to avoid mismatches with shared libraries or kernel interfaces.

### Verifying Upgrade Status

To confirm that the system is running the new release:

```sh
$ uname -a
OpenBSD myhost.example.org 7.8 GENERIC#123 amd64
```

To fetch and verify the upgrade files without rebooting immediately:

```sh
# sysupgrade -n
```

This is **not** a dry run. The `-n` option downloads and verifies the install sets, creates `/bsd.upgrade`, and then stops before rebooting. A later boot from `/bsd.upgrade` continues the upgrade. Use this option only to stage an upgrade for a controlled reboot window.

### Cleaning Up

Downloaded install sets are stored in `/home/_sysupgrade` by default. These files are not removed automatically. To free disk space:

```sh
# rm -rf /home/_sysupgrade
```

This step is optional and can be postponed if a rollback or reinstall might be necessary.

### Recommendations

Always back up critical data before upgrading, especially on production systems. While `sysupgrade` is designed to be safe and fully automated, unanticipated configurations or hardware issues may require manual recovery.

## Tracking the Development Branch (`-current`)

The OpenBSD `-current` branch represents active development. It includes new features, ongoing security enhancements, and architectural changes that will appear in the next formal release. Users running `-current` are expected to keep their systems up to date frequently and be prepared to troubleshoot issues. This environment is intended for developers, testers, and technically experienced users.

### Installing or Upgrading to `-current`

To switch to `-current`, download and install the latest snapshot from an OpenBSD mirror:

```sh
# ftp https://cdn.openbsd.org/pub/OpenBSD/snapshots/amd64/installXX.img
```

Replace `XX` with the latest available version (e.g., `install78.img`). After writing the image to a USB stick or other bootable media, reboot the system and perform a full install. The installer will recognize the snapshot and configure the system as `-current`.

Snapshots are also available in other formats (e.g., PXE-bootable kernels and install sets) and are updated frequently—often daily.

### Keeping `-current` Up to Date

Systems running `-current` do **not** use `syspatch(8)` or `sysupgrade(8)`.

To keep a `-current` system updated, either:

- Reinstall from a newer snapshot periodically
- Or build the base system from source

The recommended method is to reinstall using snapshots at regular intervals. This ensures that the base system, packages, and configuration files remain in sync.

To confirm the current version:

```sh
$ sysctl kern.version
OpenBSD 7.8-current (GENERIC.MP) #123: ...
```

Compare the build date with the timestamp of the latest snapshot on the OpenBSD mirror.

### Updating Packages on `-current`

Binary packages for `-current` are rebuilt frequently to match the current kernel and libraries. Because the base system is in flux, package and library versions must remain synchronized.

To update packages:

```sh
# export PKG_PATH=https://cdn.openbsd.org/pub/OpenBSD/snapshots/packages/$(arch -s)/
# pkg_add -u
```

If packages are temporarily unavailable or out of sync, wait a few hours and try again. Mirror lag is normal during active development.

### Rebuilding from Source

Advanced users may choose to build the base system from source using the official `release(8)` and `build(7)` procedures. This is not required when using snapshots, but may be necessary for testing kernel changes, contributing patches, or validating unreleased features.

Building from source requires a clean environment, up-to-date source tree, and attention to published changes in the system architecture. It is documented in the OpenBSD FAQ and source tree.

Running `-current` provides an opportunity to contribute bug reports and test new code. It requires regular maintenance and familiarity with system internals.
