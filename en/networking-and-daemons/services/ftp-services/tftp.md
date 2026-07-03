# TFTP

## Synopsis

**TFTP (Trivial File Transfer Protocol)** is a lightweight, UDP-based file transfer protocol used in scenarios such as network boot (PXE), firmware loading, and simple device provisioning.

OpenBSD includes the standard `tftpd(8)` daemon and `tftp(1)` client in the base system. TFTP supports only basic file reads and writes with minimal overhead. It lacks authentication and encryption, so it must be used in **trusted or isolated environments only**.

This chapter covers setup and usage of TFTP on OpenBSD, including integration with `inetd(8)`, PXE boot support, and firewall configuration.

## TFTP Use Cases

- PXE booting diskless systems
- Network bootloaders (e.g., `pxeboot`, `grub.efi`)
- Transferring configuration files to embedded devices
- Remote flashing of firmware (e.g., switches, routers)

## Running tftpd via inetd

The simplest and most secure way to run the TFTP server is through `inetd(8)`, OpenBSD’s super-server.

Edit `/etc/inetd.conf` and uncomment or add the following line:

```
tftp    dgram   udp     wait    root    /usr/sbin/tftpd     tftpd -s /tftpboot
```

- `-s /tftpboot` restricts file access to that directory (`chroot`)
- `wait` ensures `tftpd` is not spawned concurrently

Create the directory and populate it with required files:

```sh
# mkdir -p /tftpboot
# chown root:wheel /tftpboot
# chmod 755 /tftpboot
```

Enable and start `inetd`:

```sh
# rcctl enable inetd
# rcctl start inetd
```

To reload configuration after editing:

```sh
# rcctl reload inetd
```

## Testing with tftp(1)

Use the base system `tftp` client to test:

```sh
$ tftp localhost
tftp> get testfile
tftp> quit
```

The file `testfile` must exist in `/tftpboot/testfile` and be readable by user `nobody` (or `root`, depending on `inetd` configuration).

Permissions example:

```sh
# cp /bsd.rd /tftpboot/bsd.rd
# chmod 644 /tftpboot/bsd.rd
```

## PXE Boot Example

To support PXE clients:

1. Place bootloader files in `/tftpboot` (e.g., `pxelinux.0`, `grubx64.efi`, or `bsd.rd`)
2. Ensure filenames match DHCP server configuration
3. Provide matching `next-server` and `filename` in DHCP response

For use with OpenBSD `dhcpd(8)`:

```
option domain-name-servers 192.0.2.1;
option routers 192.0.2.1;

next-server 192.0.2.1;
filename "pxelinux.0";
```

PXE clients will then retrieve `pxelinux.0` from the TFTP server running on `192.0.2.1`.

## Firewall Configuration

Allow TFTP access in `pf.conf`:

```pf
pass in on $int_if proto udp from any to (self) port 69
```

TFTP uses **UDP port 69** for initial control. Data ports are dynamically negotiated per session and are ephemeral.

For tightly controlled environments, allow only the initial port and rely on trusted networks.

## Logging

`tftpd` does not log transfers by default. To enable logging, use the `-v` (verbose) or `-l` (log all requests) option in `/etc/inetd.conf`:

```
tftp    dgram   udp     wait    root    /usr/sbin/tftpd     tftpd -lv -s /tftpboot
```

Requests will appear in `/var/log/messages` or via `syslogd`.

Example:

```sh
# tail -f /var/log/messages
```

## Security Considerations

TFTP has no access control, authentication, or encryption. To reduce risk:

- Always run `tftpd` with `-s` to confine access
- Use read-only files where possible
- Do not expose TFTP to untrusted networks
- Use firewall filtering to restrict access by subnet
- Avoid write access unless explicitly needed (`-w`)

To allow write access:

```
tftp    dgram   udp     wait    root    /usr/sbin/tftpd     tftpd -sw /tftpboot
```

Then adjust file and directory permissions accordingly.

## TFTP Client (tftp(1))

OpenBSD includes the command-line `tftp(1)` client:

```sh
$ tftp remote.host
tftp> get firmware.img
tftp> put config.txt
tftp> quit
```

Use `-v` for verbose output and troubleshooting.

For automated transfers (e.g., in shell scripts), use `ftp(1)` or external tools that support TFTP.

## Alternatives and Enhancements

- Use `ftp(1)` with HTTP/FTP instead of TFTP when possible
- Combine TFTP with HTTP boot (iPXE or UEFI HTTP boot)
- Use `relayd(8)` to restrict access further
- Consider secure alternatives like `scp` or `rsync` for authenticated transfers
