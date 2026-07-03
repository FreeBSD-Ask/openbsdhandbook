# Virtualization Cheat Sheet

## Overview

OpenBSD’s native hypervisor stack includes:

- **[vmm(4)](https://man.openbsdhandbook.com/vmm.4/)**: Kernel hypervisor.
- **[vmd(8)](https://man.openbsdhandbook.com/vmd.8/)**: Virtual machine daemon.
- **[vmctl(8)](https://man.openbsdhandbook.com/vmctl.8/)**: Command-line control utility.

For detailed guidance, see the Handbook’s [Virtualization chapter](https://www.openbsdhandbook.com/virtualization/)
.

## Directory Layout

Use `/var/vmm/` for VM assets to align with OpenBSD conventions.

| Purpose | Path | Notes |
| --- | --- | --- |
| Disk images | `/var/vmm/images/<name>.img` | Sparse raw files created by `vmctl`. |
| Installer ISOs | `/var/vmm/iso/<file>.iso` | Store official ISOs (OpenBSD, FreeBSD, etc.). |
| Configuration files | `/var/vmm/conf/` | Optional: store `install.conf`, kickstart, etc. |
| Logs (host) | `/var/log/vmd.log` | `vmd(8)` logs for troubleshooting. |

## Disk and Image Creation

| Task | Command | Notes |
| --- | --- | --- |
| Create 10G sparse disk image | `# vmctl create -s 10G /var/vmm/images/obsd.img` | Grows on write up to specified size. |
| Download OpenBSD ISO | `# ftp https://cdn.openbsd.org/pub/OpenBSD/78/amd64/install78.iso -o /var/vmm/iso/install78.iso` | Use `ftp(1)` or similar tool. |

## Single OpenBSD VM (Interactive Install)

```sh
# vmctl create -s 10G /var/vmm/images/obsd.img
# vmctl start -c -m 1G -r /var/vmm/iso/install78.iso -d /var/vmm/images/obsd.img obsd
```

- `-c`: Connect to console.
- `-m`: Memory (1G = 1024M).
- `-r`: ISO for installation.
- `-d`: Disk image.
- Last argument: VM name.

After installation, reboot without ISO:

```sh
# vmctl stop obsd
# vmctl start -c -m 1G -d /var/vmm/images/obsd.img obsd
```

## Networking Options

Use `-n` with `vmctl start` to configure networking.

| Mode | Description | Command Example |
| --- | --- | --- |
| `-n local` | Private NAT network; default, simple. | `-n local` |
| `-n <switch>` | Connect to a virtual switch (`vmd(8)`-managed). | Create switch, then `-n switch0`. |
| `-n <tapX>` | Attach to a `tap(4)` interface for custom setup. | `ifconfig tap0 create`, then `-n tap0`. |

### Example: Two VMs on Private NAT Network

```sh
# vmctl create -s 5G /var/vmm/images/vm1.img
# vmctl create -s 5G /var/vmm/images/vm2.img
# vmctl start -c -m 1G -d /var/vmm/images/vm1.img -n local vm1
# vmctl start -c -m 1G -d /var/vmm/images/vm2.img -n local vm2
```

### Example: Bridge VMs to Physical LAN

1. Create virtual switch and bridge:

```sh
# ifconfig vether0 create
# ifconfig bridge0 create
# ifconfig bridge0 add em0 add vether0 up
```

Replace `em0` with your physical NIC.

2. Start VMs:

```sh
# vmctl start -c -m 2G -d /var/vmm/images/srv.img -n switch0 srv
# vmctl start -c -m 1G -d /var/vmm/images/cli.img -n switch0 cli
```

> **Tip**: For VLANs, use `vether0.<vlan>` or configure VLAN tagging on the bridge. See [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
> and [vether(4)](https://man.openbsdhandbook.com/vether.4/)
> .

## Non-OpenBSD Guests

Use amd64 ISOs and allocate sufficient memory (2G+ recommended).

### FreeBSD

```sh
# vmctl create -s 20G /var/vmm/images/fbsd.img
# vmctl start -c -m 2G -r /var/vmm/iso/FreeBSD-14.1-RELEASE-amd64-dvd1.iso -d /var/vmm/images/fbsd.img fbsd
```

### Linux (Debian Example)

```sh
# vmctl create -s 10G /var/vmm/images/debian.img
# vmctl start -c -m 2G -r /var/vmm/iso/debian-12.7.0-amd64-netinst.iso -d /var/vmm/images/debian.img debian
```

### Windows (Limited Support)

Windows support is experimental; expect limited functionality.

```sh
# vmctl create -s 30G /var/vmm/images/win10.img
# vmctl start -c -m 4G -r /var/vmm/iso/Win10_22H2_English_x64.iso -d /var/vmm/images/win10.img win10
```

## VM Management

| Action | Command |
| --- | --- |
| List VMs | `# vmctl status` |
| Connect to console | `# vmctl console <name>` |
| Graceful shutdown | Guest: `halt` or OS-specific |
| Force stop | `# vmctl stop -f <name>` |
| Pause/resume | `# vmctl pause <name>` / `# vmctl unpause <name>` |

For non-root access, configure [doas(1)](https://man.openbsdhandbook.com/doas.1/)
.

## Troubleshooting

- Check `vmd` logs: `tail -f /var/log/vmd.log`.
- Verify network interfaces: `ifconfig -a`.
- Confirm paths: `/var/vmm/images/`, `/var/vmm/iso/`.
- Check `pf(4)` rules for custom networking issues.
