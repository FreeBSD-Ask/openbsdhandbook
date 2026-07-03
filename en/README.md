# OpenBSD Handbook

> **“Linux people do what they do because they hate Microsoft. We do what we do because we love Unix.”**
>
> – Theo de Raadt

The OpenBSD Handbook is a practical reference for system administrators. It emphasizes clear explanations, minimal assumptions, and copy-pasteable examples. The aim is dependable installation, hardening, and operation of OpenBSD 7.8 on servers, appliances, and workstations.

## How OpenBSD Differs

OpenBSD is a complete, coherent Unix-like operating system maintained as an integrated whole. The project prioritizes simplicity, correctness, and security.

- **Coherent base system.** Kernel, userland, toolchain, and core daemons are developed together. Administrative interfaces such as [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
  are consistent across services.
- **Security engineering.** Proactive code audits, privilege separation, and exploit mitigations are standard practice. Application sandboxes [pledge(2)](https://man.openbsdhandbook.com/pledge.2/)
  and [unveil(2)](https://man.openbsdhandbook.com/unveil.2/)
  are part of the base system.
- **Practical defaults.** The PF firewall is configured via [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
  . A minimal web server ([httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
  ) uses [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)
  . Other base components include [relayd(8)](https://man.openbsdhandbook.com/relayd.8/)
  , [smtpd(8)](https://man.openbsdhandbook.com/smtpd.8/)
  , [bgpd(8)](https://man.openbsdhandbook.com/bgpd.8/)
  , and OpenSSH.
- **Documentation first.** Manual pages are authoritative and complete; start with [man(1)](https://man.openbsdhandbook.com/man.1/)
  . Package management begins with [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
  .
- **Predictable releases.** A new release appears roughly every six months. Binary patches are applied with [syspatch(8)](https://man.openbsdhandbook.com/syspatch.8/)
  . Upgrades between releases can be automated with [sysupgrade(8)](https://man.openbsdhandbook.com/sysupgrade.8/)
  .

## Strengths and Trade-offs

OpenBSD favors secure, comprehensible designs over feature accretion. The base system is small and intentionally conservative. Hardware support is broad but not universal; some specialized or proprietary devices may lack drivers. Ports and packages provide a wide range of third-party software, with emphasis on maintainability. Performance matters, but is not pursued at the expense of safety or clarity.

## How to Use This Handbook

This handbook supplements the manual pages with workflows, background notes, and cautions. Sections reference the relevant manual pages and include tested examples. Conventions:

- Shell prompts: `$` indicates an unprivileged user; `#` indicates the superuser (typically via `doas`; see [doas(1)](https://man.openbsdhandbook.com/doas.1/)
  ).
- Command output and configuration fragments are shown exactly as entered or produced.
- The first reference to a command, daemon, configuration file, or system call links to its manual page.

For orientation and scope, see the Introduction.

Read next -> [Introduction](install-and-configure/introduction.md)

## Start Here

Common tasks and entry points:

- **Install and bootstrap.** Disk layout, networking, users, first login.
  -> [/install/](install-and-configure/installation.md)
- **Secure the base.** Privilege model, `doas`, file permissions, cryptography, patching with syspatch, release upgrades with sysupgrade.
  -> [/security/](system-administration/security.md)
- **Networking.** Interfaces, autoconfiguration, bridging, VLANs, wireless, routing.
  -> [/networking/](install-and-configure/networking.md)
- **PF firewall.** Policy design, NAT, redirection, tables, anchors, testing with `pfctl` (see [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
  ).
  -> [/pf/](networking-and-daemons/pf/README.md)
- **Web and proxies.** httpd, TLS, OCSP, reverse proxying, health checks with [relayd(8)](https://man.openbsdhandbook.com/relayd.8/)
  .
  -> [/httpd/](networking-and-daemons/services/web-services/httpd.md)
  · [/relayd/](networking-and-daemons/services/web-services/relayd.md)
- **Mail transport.** OpenSMTPD basics (see [smtpd(8)](https://man.openbsdhandbook.com/smtpd.8/)
  ), local delivery, relaying.
  -> [/mail/](networking-and-daemons/services/mail-services/README.md)
- **Storage.** Disks, partitions, softraid, encryption, filesystems, backups.
  -> [/storage/](system-administration/storage.md)
- **Virtualization.** Native virtualization with [vmd(8)](https://man.openbsdhandbook.com/vmd.8/)
  and the [vmm(4)](https://man.openbsdhandbook.com/vmm.4/)
  device.
  -> [/vmm/](system-administration/virtualization.md)
- **Packages and ports.** Searching, installing, version locks, building from ports with [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
  .
  -> [/packages/](install-and-configure/ports.md)
- **Upgrades and maintenance.** Release planning, backups, post-upgrade checks.
  -> [/upgrading/](system-administration/upgrade.md)
- **Workstation topics.** X Window System, graphics, audio, daily tools.
  -> [/x11/](install-and-configure/x11.md)

For deeper reference, consult the manuals:
Manuals -> <https://man.openbsdhandbook.com/>

## Scope and Audience

The handbook targets technically proficient administrators who prefer explicit, reproducible procedures and conservative defaults. Content avoids unnecessary abstraction, calls out common pitfalls, and favors simple, auditable solutions over complex stacks.

For environments where predictability and security matter, OpenBSD offers a stable foundation with clear trade-offs. This handbook provides the procedures required to apply it effectively.
