# Samba

## Synopsis

**Samba** is a suite of programs that implements the SMB (Server Message Block) protocol and its modern extension, CIFS (Common Internet File System). It allows Unix systems to share files and printers with Windows clients, and vice versa. Samba can function as a standalone file server, a member of a Windows domain, or as an Active Directory domain controller (though the latter roles are less common on OpenBSD).

The OpenBSD base system does not include Samba. It is available via the ports and packages system.

## Installation

To install Samba:

```sh
# pkg_add samba
```

This will install the Samba server binaries, configuration files, and documentation. The package includes `smbd(8)` for file and printer sharing and `nmbd(8)` for NetBIOS name resolution and browsing. For basic server use, `smbd` alone is often sufficient.

## Configuration

Samba is configured through `/etc/samba/smb.conf`. This file must be created before starting the daemons. A minimal configuration to export a public share might look as follows:

### Configuration

```ini
[global]
   workgroup = WORKGROUP
   server string = OpenBSD Samba Server
   security = user
   log file = /var/log/samba/log.%m
   max log size = 1000
   passdb backend = tdbsam
   load printers = no
   printing = bsd
   disable spoolss = yes

[public]
   path = /srv/smb/public
   public = yes
   writable = yes
   printable = no
   guest ok = yes
```

This defines a shared directory at `/srv/smb/public`, accessible to anonymous users with write access. In more secure environments, `guest ok = no` should be used and access controlled via user authentication.

Samba uses its own user database. System users must be added to it with:

```sh
# smbpasswd -a username
```

Ensure that the shared directory exists and has proper permissions:

```sh
# mkdir -p /srv/smb/public
# chown -R nobody:nogroup /srv/smb/public
# chmod 0777 /srv/smb/public
```

These permissions allow guest users to write. Adjust as appropriate for authenticated sharing.

## Service Management

Samba does not integrate directly with OpenBSD’s `rc.d(8)` framework out of the box. Services may be started via `rc.local` or custom `rc.d` scripts.

To launch the services manually:

```sh
# /usr/local/sbin/smbd
# /usr/local/sbin/nmbd
```

To launch automatically at boot, append to `/etc/rc.local`:

```sh
if [ -x /usr/local/sbin/smbd ]; then
    echo -n ' smbd'; /usr/local/sbin/smbd
fi
if [ -x /usr/local/sbin/nmbd ]; then
    echo -n ' nmbd'; /usr/local/sbin/nmbd
fi
```

Alternatively, create a custom `rc.d` script to integrate with `rcctl(8)`.

## Client Access

On Unix clients, access to Samba shares can be tested using `smbclient`, part of the `samba-client` package:

```sh
$ pkg_add samba-client
$ smbclient //hostname/public -U username
```

On Windows, open File Explorer and enter:

```
\\hostname\public
```

Replace `hostname` with the Samba server’s IP address or NetBIOS name.

Authentication prompts will appear unless guest access is configured.

## Printer Sharing

Printer support is limited and typically requires integration with a print spooler such as `cupsd` or `lpd`. For basic file server functionality, it is common to disable printer support entirely with:

```conf
load printers = no
disable spoolss = yes
```

This reduces the attack surface and prevents misleading printer listings in client environments.

## Firewall Considerations

The SMB protocol uses multiple ports:

- TCP 139: NetBIOS Session Service
- TCP 445: Direct SMB over TCP
- UDP 137: NetBIOS Name Service
- UDP 138: NetBIOS Datagram Service

If using `pf(4)`, pass rules must be written to allow traffic to these ports from client hosts:

```pf
pass in on $int_if proto tcp from any to (self) port { 139, 445 }
pass in on $int_if proto udp from any to (self) port { 137, 138 }
```

Do not expose these services to the Internet.

## Security Considerations

Samba is a large and complex daemon and has historically had several vulnerabilities. It is strongly recommended to:

- Limit access to trusted networks
- Disable unnecessary features (e.g., printing, guest access)
- Regularly update the package
- Use firewalls to restrict access
- Avoid running Samba on Internet-facing interfaces

For production environments requiring strong access control and integration with centralized authentication, consider running Samba in standalone mode only or using alternatives such as `rsync` or `NFS` depending on client needs.

## Differences Compared to NFS

| Feature | Samba | NFS |
| --- | --- | --- |
| Protocol | SMB/CIFS | NFSv3 |
| Windows Support | Native | Limited |
| Authentication | User/password (tdbsam) | UID-based, `maproot` |
| Integration | Windows environments | Unix-like systems |
| Portability | Works well with Windows | Works well with Unix/Linux |
| Required Packages | Not in base | Built into OpenBSD |

Samba is ideal for heterogeneous environments with Windows clients. NFS is better suited for Unix-only networks.
