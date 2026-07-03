# The OpenBSD Boot Process

## Synopsis

OpenBSD follows a straightforward, well-documented boot process designed for reliability, clarity, and security. The boot mechanism consists of several distinct stages that begin with firmware (such as BIOS or UEFI) and conclude with the system transitioning to multi-user mode after executing startup scripts. This chapter describes each stage, including bootloaders, kernel initialization, and early userland processes.

## Boot Stages Overview

The boot process on OpenBSD systems typically proceeds through the following stages:

1. Firmware Initialization (BIOS or UEFI)
2. Bootloader Execution (`boot(8)`)
3. Kernel Loading and Initialization
4. Userland Initialization via `init(8)`
5. Multi-User Startup Scripts (`rc(8)`)

Each stage is modular, allowing flexibility in system recovery and troubleshooting.

## Firmware Initialization

When power is applied to the system, the firmware (BIOS or UEFI) initializes hardware and performs a POST (Power-On Self-Test). It then scans for bootable media and transfers control to the first-stage bootloader, which resides in the Master Boot Record (MBR) or in the EFI System Partition (ESP) depending on system architecture and configuration.

On x86 systems with UEFI, the loader is typically installed at:

```
EFI\BOOT\BOOTX64.EFI
```

On legacy BIOS systems, the bootloader is written to the MBR and the OpenBSD disklabel.

## The Bootloader (`boot(8)`)

OpenBSD uses a multi-stage bootloader composed of several programs:

- **`biosboot`**: First-stage bootloader for BIOS systems.
- **`pxeboot`**: Bootloader for PXE network booting.
- **`bootefi`**: EFI bootloader for UEFI systems.
- **`boot`**: Final stage bootloader, located in the root filesystem as `/boot`.

The `boot` program is responsible for locating and loading the OpenBSD kernel. It provides a user interface and a limited command environment that supports boot-time configuration.

At the boot prompt (`boot>`), users can specify a different kernel or pass options. For example:

```
boot> boot -s
```

This boots into single-user mode. Alternatively, to boot an older kernel:

```
boot> boot /bsd.old
```

If no input is provided, the default kernel (`/bsd`) is loaded after a short timeout.

## Kernel Initialization

Once the kernel is loaded, it initializes hardware, sets up device drivers, and mounts the root filesystem as read-only. Messages during this phase show kernel output from device detection and subsystem initialization.

The kernel then attempts to run the initial process, typically `/sbin/init`.

If the kernel cannot locate or execute `init(8)`, it will panic and halt or enter the debugger (`ddb(4)`), depending on boot settings.

## `init(8)` and Single-User Mode

The `init(8)` program is the first userland process. It performs the following steps:

- Reads the contents of `/etc/ttys` to configure terminals.
- If the `-s` option was provided, or if the system was not shut down cleanly, `init` enters **single-user mode**.
- In single-user mode, the root shell is invoked directly on the console without networking or multi-user services.

Single-user mode is used for maintenance, such as repairing filesystems or recovering from misconfiguration.

To proceed to multi-user mode from here:

```sh
# exit
```

## Multi-User Initialization

In multi-user mode, `init` executes the `/etc/rc` script. This script performs essential startup tasks, including:

- Checking and mounting filesystems (`fsck`, `mount`)
- Configuring the network (`hostname.if`, `dhcpleased`, etc.)
- Starting system daemons via `rcctl(8)`
- Applying `sysctl(8)` and `wsconsctl(8)` settings
- Running daily maintenance scripts (`daily`, `security`, etc.)

The output of `/etc/rc` is logged to `/var/run/rc.log`.

Terminal logins are enabled via `ttys(5)` by launching `getty(8)` on configured virtual consoles or serial ports.

## Boot-Time Configuration

Users can interrupt the bootloader to modify boot parameters, select an alternate kernel, or enter single-user mode.

### `-s`: Single-User Mode

Booting with `-s` starts the system in single-user mode. This is useful for performing repairs or entering a recovery shell.

```
boot> boot -s
```

### `-a`: Ask for Root Device

The `-a` flag causes the kernel to prompt for the root device, swap device, and filesystem type.

```
boot> boot -a
```

This is helpful when the root filesystem cannot be determined automatically.

### `-v`: Verbose Mode

The `-v` option enables verbose output from the kernel. This provides additional detail during the hardware initialization process.

```
boot> boot -v
```

### `-d`: Enter Debugger

When compiled with debugging support, `-d` drops into the kernel debugger (`ddb(4)`) early in the boot process.

```
boot> boot -d
```

### Specifying an Alternate Kernel

To load a kernel other than the default `/bsd`, the full path must be provided.

```
boot> boot /bsd.old
```

This is commonly used after a system upgrade to fall back to a known-good kernel.

## Emergency Recovery and Alternate Kernels

OpenBSD installs backup kernels as `/bsd.old` and `/bsd.sp`. These can be used if the primary kernel is misconfigured or incompatible with the system.

To boot into a previous kernel in single-user mode:

```
boot> boot /bsd.old -s
```

In the event of a boot failure, it is possible to boot from an installation medium such as `install78.img` and enter the shell mode for manual recovery.

## Bootloader Configuration and Timeout

The behavior of the bootloader may be customized using the `/etc/boot.conf` file. This file is read by the `boot` program and can be used to automate kernel selection or configure timeout settings.

Example `/etc/boot.conf`:

```
set timeout 5
boot /bsd
```

This configuration sets a five-second timeout and boots the default kernel.

## Key Files and Tools

| File | Purpose |
| --- | --- |
| `/bsd` | Default kernel |
| `/bsd.mp` | Multiprocessor kernel (default on SMP systems) |
| `/bsd.sp` | Single-processor kernel |
| `/bsd.old` | Backup kernel from previous upgrade |
| `/boot` | Final-stage bootloader |
| `/etc/boot.conf` | Bootloader configuration |
| `/etc/rc` | Startup script for multi-user mode |
| `/etc/ttys` | Terminal configuration |
| `/etc/fstab` | Filesystem mount configuration |

## Shutdown Sequence

The OpenBSD shutdown process is handled by `init(8)` and is initiated by using `shutdown(8)`, `reboot(8)`, or `halt(8)`. When `shutdown(8)` is called with appropriate arguments, the system follows a controlled sequence to terminate all userland activity safely.

Upon invocation of `shutdown(8)`, the following steps occur:

1. `init(8)` attempts to execute the script `/etc/rc.shutdown` if it exists. This script may be used to stop additional services or perform cleanup tasks before shutdown.
2. All processes are sent the `TERM` signal to request graceful termination.
3. After a short delay, any remaining processes are sent the `KILL` signal.
4. Filesystems are unmounted and system buffers are flushed.
5. The system either reboots or powers off, depending on the shutdown mode.

To **power off** the system immediately (if supported by the hardware), use:

```sh
# shutdown -p now
```

To **reboot** the system:

```sh
# shutdown -r now
```

Alternatively, one may use:

```sh
# halt
# reboot
```

Note that the user must be root or a member of the `operator` group in order to invoke these commands directly.

Membership in the `operator` group may be configured using `vipw(8)` and `chpass(1)`, as described in the chapter on account management.

The `shutdown(8)` utility supports scheduling future shutdowns and sending warnings to logged-in users. It is the preferred method for halting or rebooting systems in multi-user mode to avoid data loss and ensure a consistent system state.

The system logs shutdown and reboot events to `/var/log/messages`, and the kernel timestamp associated with the next boot will reflect a clean shutdown if this process completes properly.
