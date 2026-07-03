# Introduction

## Synopsis

This chapter introduces the OpenBSD operating system. It provides an overview of the project’s origins, its guiding principles, development practices, licensing approach, and areas of technical focus. Readers will gain a foundational understanding of the system and its goals before proceeding to installation and configuration topics.

## Historical Background

OpenBSD began in 1995 as a fork of NetBSD. The project was founded by Theo de Raadt after his departure from the NetBSD core team. The initial motivation was to establish a development environment that was open to public discussion and scrutiny, with a particular emphasis on code quality and security.

From its inception, OpenBSD differentiated itself by focusing not only on correctness and performance, but also on proactive security measures and a permissive licensing model. It quickly gained a reputation for code auditing, consistent releases, and integrated cryptographic functionality. Over time, the project introduced many security technologies that were subsequently adopted by other operating systems.

The OpenBSD project is headquartered in Calgary, Alberta, Canada, but maintains a global developer community that collaborates through CVS and mailing lists. The project’s motto, “Secure by default,” reflects its mission to ship an operating system with secure defaults and minimal attack surface.

## Project Goals and Development Model

The OpenBSD project is governed by a set of core principles that shape its architecture, policies, and release process. It emphasizes clean and correct code, careful integration of new features, and a commitment to open development practices. Changes to the system are always reviewed for portability, correctness, security impact, and maintainability.

Development is coordinated through a central CVS repository. Patches are submitted by developers and reviewed prior to integration. There is no anonymous commit access. Daily snapshots and semi-annual releases are produced from this repository. Code changes are public immediately, and every aspect of the system is open for inspection.

Unlike distributions that assemble packages from external sources, OpenBSD provides a coherent base system with consistent interfaces and documentation. Users are expected to consult the manual pages for detailed technical behavior. These pages are maintained with the same care as the code.

## Release Process

OpenBSD follows a strict six-month release cycle. New versions are published in the months of May and November. Releases are named numerically, such as 7.7 or 7.8, and are considered stable for production use.

Each release is supported with binary security patches for approximately one year. Patches are distributed using `syspatch(8)` and are available for supported architectures only. Daily snapshots are also produced for those who wish to follow development more closely.

The release engineering process emphasizes stability and correctness. Features that are not ready are deferred rather than rushed. All supported architectures must build and boot successfully before a release is finalized.

## Licensing and Code Reuse

OpenBSD favors permissive software licenses, such as the ISC license and BSD license. The project explicitly avoids incorporating code under the GNU General Public License (GPL) whenever possible. This policy enables broader reuse of OpenBSD code by other systems and organizations without legal restrictions.

The base system is developed as a unified whole, with no differentiation between core and third-party components. This allows the project to maintain a high level of consistency and auditing across the entire codebase.

Code from OpenBSD is used widely in other systems. Notable examples include the adoption of `pf(4)` in FreeBSD, the use of OpenSSH across nearly all UNIX-like platforms, and the integration of OpenBSD’s malloc implementation in various environments.

## Supported Architectures

OpenBSD places a strong emphasis on portability and supports a wide range of hardware architectures. As of the current release, the officially supported platforms include amd64, arm64, macppc, riscv64, and several others. Each supported platform must successfully complete full system builds, regression tests, and boot validation prior to release.

Support for multiple architectures ensures that OpenBSD remains portable, cleanly structured, and applicable to a wide range of deployment environments, including embedded systems, desktops, and high-performance servers. Architectures are added or retired based on hardware availability, developer interest, and build quality.

## Security Philosophy

Security in OpenBSD is not treated as a reactive process, but as a design principle. The project proactively addresses potential classes of vulnerabilities through code auditing, compiler-based mitigation, and system call restrictions.

Several features have originated in OpenBSD, including:

- W^X memory protection, which enforces non-executable writable memory regions
- `pledge(2)` and `unveil(2)`, which restrict process capabilities and file system access
- Integrated cryptographic support in the kernel and userland
- Default denial network filter policies using `pf(4)`
- Secure memory allocation routines in libc

Security features are typically enabled by default. For example, userland binaries are compiled with stack protection, ASLR is enabled across supported platforms, and the default install enables only essential services.

## Documentation and Manual Pages

The primary source of documentation in OpenBSD is the manual page system. Every utility, system call, configuration file, and daemon is documented in a consistent and well-maintained format.

Manual pages are categorized by section. For instance, `rcctl(8)` documents a system administration tool, while `vm.conf(5)` describes a configuration file format. When conflicting information arises, the manual pages are considered the authoritative reference.

The OpenBSD Handbook complements these manual pages by offering higher-level explanations, structured guidance, and contextual information that would not be appropriate for a reference manual. Users are encouraged to read the handbook alongside the man pages, and to verify commands and options using the source.

## Community and Support

OpenBSD maintains a small but technically proficient community. The project does not operate official forums or user-facing support infrastructure. Instead, communication takes place on mailing lists such as `misc@openbsd.org` and in real-time on IRC channels like `#openbsd` on Libera.Chat.

Questions are expected to be specific, well-researched, and accompanied by relevant system output. The culture favors self-reliance, thorough documentation, and reproducible bug reports.

Bug tracking is conducted via mailing lists, not web-based trackers. Patches and problem reports should be submitted in plain text, and developers may respond directly with requests for clarification or additional diagnostics.

OpenBSD encourages users to audit, understand, and contribute to the system. All code is publicly accessible, and contributions are welcomed if they align with the project’s goals.

## Use Cases

OpenBSD is a general-purpose operating system designed with an emphasis on correctness, code clarity, and security. It is suitable for a wide range of applications where transparency, auditability, and predictable behavior are valued. With its permissive licensing model and complete source availability, OpenBSD is particularly well suited to long-term infrastructure, education, research, and development environments.

Many organizations deploy OpenBSD in security-sensitive and network-facing roles. The operating system includes a high-quality TCP/IP stack, robust daemon implementations, and native support for features commonly required in modern infrastructure.

Typical use cases include:

- **Internet Services**: OpenBSD is often used to provide core Internet-facing services. These include:

  - Web servers using `httpd(8)` or `relayd(8)`
  - Mail servers using `smtpd(8)`
  - DNS resolvers and authoritative name servers with `nsd(8)` and `unbound(8)`
  - Secure file transfers using `sftp`, `scp`, or `rsync` over SSH
  - VPN endpoints using `ipsecctl(8)` or `openiked(8)`
- **Network Infrastructure**: OpenBSD is used as a foundation for reliable and maintainable routing systems:

  - BGP routing using `OpenBGPD`
  - OSPF and RIP routing via `OpenOSPFD` and `OpenRIPD`
  - High-availability clusters using `carp(4)`
  - Load balancing and proxying with `relayd(8)`
  - NAT gateways and packet filtering using `pf(4)`
- **Firewalls and Gateways**: Many deploy OpenBSD as a hardened perimeter system:

  - Stateful firewalling with `pf(4)`
  - NAT and port redirection
  - DHCP and DNS relay services
  - Traffic shaping and filtering
- **Development and Testing**: OpenBSD is frequently used by software developers who require a reproducible and minimal environment. It offers:

  - A predictable base system with no hidden dependencies
  - A clean C-based userland and compiler toolchain
  - Safe defaults and strong user-space isolation using `pledge(2)` and `unveil(2)`
- **Research and Education**: OpenBSD provides a fully auditable and cohesive system for:

  - Operating systems research and kernel instrumentation
  - Secure software design studies
  - Network protocol experimentation
  - Systems programming and C language education
- **Embedded and Appliance Systems**: Thanks to its minimalism and consistent behavior, OpenBSD is often chosen as a platform for embedded systems, custom appliances, and headless deployments.

While OpenBSD includes support for X11, sound devices, and wireless networking, its strengths lie in simplicity, reliability, and verifiability. Systems built on OpenBSD are easier to reason about, test, and maintain over long periods without dependency drift or configuration ambiguity. This makes it a preferred choice for engineers, administrators, and researchers who demand a system that behaves exactly as documented.
