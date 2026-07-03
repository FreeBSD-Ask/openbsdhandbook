# Linux Compatibility

## Synopsis

This chapter documents the current status of Linux binary compatibility on OpenBSD. It describes the historical `linux(4)` subsystem that once enabled Linux ELF binary execution, the reasons for its removal, and modern alternatives for accessing Linux-only software, including virtualization, remote execution, and native packaging.

## Historical Note

OpenBSD previously supported partial Linux compatibility through the `linux(4)` kernel subsystem. This layer enabled execution of select Linux ELF binaries, primarily for compatibility with older commercial or binary-only software. It required the presence of Linux-shared libraries and emulated system calls within a controlled environment.

The feature was rarely used and became increasingly difficult to maintain. It was formally removed from the kernel in OpenBSD 6.3.

## Rationale for Removal

The `linux(4)` implementation was removed for the following reasons:

- Maintaining binary compatibility with a foreign ABI conflicted with OpenBSD’s focus on correctness, simplicity, and security.
- Most software that once required Linux compatibility has become open source and is available natively.
- The compatibility layer required ongoing effort to track changes in Linux behavior and libc interfaces, with little user demand.

The subsystem was considered obsolete and unmaintainable in light of modern application and system design.

## Alternatives and Workarounds

### Native Packages and Ports

A large number of software packages initially developed for Linux are now available as native OpenBSD ports. These include:

- Desktop software: Firefox, Chromium, LibreOffice
- Development tools: GCC, Rust, Go
- Networking tools: WireGuard, Docker clients, Kubernetes utilities

To search for available software:

```sh
$ pkg_info -Q firefox
$ pkg_info -Q libreoffice
```

If a package is not available, it may be possible to build it from source, assuming it does not rely on Linux kernel-specific interfaces.

### Virtualization

Linux applications requiring a full Linux environment can be executed inside a virtual machine using OpenBSD’s built-in hypervisor framework (`vmm(4)` and `vmd(8)`).

Instructions for setting up virtual machines with Linux guests are provided in the [Virtualization](../system-administration/virtualization.md)
chapter.

### Remote Execution

For workflows involving Linux-only software, remote execution on a Linux host may be a practical solution. This can be done over SSH with optional X11 forwarding:

```sh
$ ssh -X user@linuxhost gimp
```

To run console-only applications:

```sh
$ ssh user@linuxhost ffmpeg -i input.avi -vf scale=640:360 output.mp4
```

Files may be transferred using `scp` or `rsync` as needed.

Remote Linux environments can also be provided using containers (e.g., `podman`, `docker`, `lxc`) or a full virtual machine.
