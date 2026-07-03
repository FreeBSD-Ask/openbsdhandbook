# Windows Compatibility

## Synopsis

This chapter describes the current state of Windows compatibility on OpenBSD. Wine is not available on OpenBSD due to architectural, security, and kernel-related incompatibilities. This chapter explains why Wine has not been successfully ported, documents past efforts, and offers practical alternatives such as virtualization, remote access, and native software.

## Wine and OpenBSD

Wine (Wine Is Not an Emulator) is a compatibility layer that allows execution of Microsoft Windows applications on Unix-like operating systems. It is well-supported on FreeBSD, NetBSD, and most Linux distributions.

However, **Wine is not available on OpenBSD** in the official ports tree or package collection. Historical attempts to port Wine have failed or been abandoned due to a number of fundamental incompatibilities with OpenBSD’s design.

## Technical Barriers

Efforts to port Wine to OpenBSD have encountered the following technical obstacles:

- **Lack of 32-bit Binary Support**: OpenBSD/amd64 does not support running 32-bit binaries. This breaks compatibility with many Windows applications and subsystems, as Wine relies heavily on 32-bit execution even when running 64-bit Windows programs. Many installers, DLLs, and system components expect a 32-bit runtime environment.
- **Security Restrictions**: OpenBSD’s strict memory protection policies, such as W^X (write xor execute) and the prohibition on mapping address page 0, conflict with Wine’s need to emulate Windows memory layouts and execution models. These restrictions interfere with Wine’s Portable Executable (PE) loader, signal handling, and Just-In-Time code execution.
- **Kernel ABI Divergence**: OpenBSD’s kernel differs significantly from Linux and other BSD systems. Wine assumes a Linux-like or FreeBSD-like process model and signal behavior, making low-level emulation of Windows APIs impractical without kernel changes.
- **Dependency and Build Failures**: Attempts to build Wine from source have encountered missing or incompatible libraries (e.g., `lcms`, `hal`, `alsa`) and errors related to process cloning and signal delivery. These issues are unresolved and not actively maintained.

The last known efforts to port Wine were discontinued, and the port has since been removed from the OpenBSD ports tree.

## Historical Attempts

Porting efforts for Wine on OpenBSD date back as far as 2008 and continued intermittently until around 2018. These efforts never reached a usable or maintainable state. At one point, Wine existed in the ports tree under `emulators/wine`, but it was later marked as broken and placed in the attic.

Earlier versions of Wine were sometimes run using OpenBSD’s now-removed `linux(4)` compatibility layer on i386 systems. However, this method is no longer supported or recommended.

## Alternatives

### Virtualization

The most reliable approach to running Windows software on OpenBSD is virtualization. OpenBSD includes the `vmm(4)` hypervisor and `vmd(8)` management daemon, which allow running a full Windows virtual machine in user space.

This method avoids emulation entirely and provides full compatibility with Windows applications and system calls. While performance is limited compared to Linux (due to lack of KVM), it remains secure and functional.

See the [Virtualization](../system-administration/virtualization.md)
chapter for setup instructions.

### Remote Access

Another option is to run Windows software remotely on a Windows system and access it using standard protocols:

- `ssh` with X11 forwarding
- `rdesktop` or `xfreerdp` for RDP
- `vncviewer` for VNC
- `xrdp` or `x11vnc` on the remote host

Example:

```sh
$ rdesktop winhost.example.org
```

This is especially suitable for enterprise environments or where a dedicated Windows machine is available.

### Native Alternatives

Where possible, replacing Windows applications with native software is preferred. Many open-source and cross-platform programs are already packaged for OpenBSD.

Examples include:

| Windows Software | OpenBSD Alternative |
| --- | --- |
| Notepad++ | `mg`, `mousepad`, `vim` |
| PuTTY | `ssh` (in base system) |
| WinSCP | `sftp`, `rsync`, `scp` |
| MS Paint | `gimp`, `xpaint` |
| MS Word | `libreoffice` |

Search for available packages using:

```sh
$ pkg_info -Q libreoffice
$ pkg_info -Q gimp
```

Some older Windows software may also run under `dosbox`, which is available via packages and useful for running 16-bit or DOS-era Windows applications (e.g., SimCity 2000 with Windows 3.1 installed inside DOSBox).

### Alternate Operating Systems

For users who regularly require Wine, it may be appropriate to run a dual-boot setup or maintain a FreeBSD or Linux guest in a virtual machine. Both platforms support Wine natively and have extensive documentation for running Windows software.
