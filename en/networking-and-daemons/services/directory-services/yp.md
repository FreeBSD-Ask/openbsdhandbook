# YP (NIS)

## Synopsis

**YP (Yellow Pages)**, also known as **NIS (Network Information Service)**, is a simple protocol for sharing configuration databases such as user accounts, group memberships, and hostnames across a trusted local network. OpenBSD includes support for both YP servers and clients in the base system.

YP is **not the same as LDAP**, and it is important to distinguish them:

- YP is an older, RPC-based system developed by Sun Microsystems.
- LDAP (Lightweight Directory Access Protocol) is a modern, extensible protocol standardized in RFC 4511.
- YP is included in OpenBSD by default; LDAP support requires installing `openldap-server` and related packages.

YP is best suited for small, trusted networks where simplicity is valued over flexibility or cryptographic security. LDAP offers a richer schema model, encryption, and fine-grained access control, and is generally preferred in larger or more security-sensitive environments.

This chapter explains how to configure OpenBSD as a YP master, YP slave, or YP client.

## YP Master Server Configuration

The YP master server maintains the authoritative copy of all YP maps (databases). These maps are generated from files in `/etc` and served to clients and slave servers.

The required daemons are:

- `portmap(8)`: RPC port mapper
- `ypserv(8)`: YP service daemon
- `ypbind(8)`: Required on the master to serve as a client to itself
- `rpc.ypxfrd(8)`: Optional map transfer accelerator (used by slaves)

### Enabling Required Services

Enable and start the necessary services:

```sh
# rcctl enable portmap ypserv ypbind
# rcctl start portmap
# rcctl start ypserv
# rcctl start ypbind
```

The YP master must also be able to generate and serve YP maps. Use `ypinit(8)` with the `-m` flag:

```sh
# ypinit -m
```

The script will prompt for the YP domain name and the list of slave servers. If there are no slaves, press Enter to continue.

The YP domain name (not to be confused with DNS domains) must also be set at boot. This is done using `domainname(1)` and `/etc/defaultdomain`:

```sh
# echo "exampledomain" > /etc/defaultdomain
```

To apply it immediately:

```sh
# domainname exampledomain
```

### Map Creation and Regeneration

The YP maps are built from standard system files using Makefiles. The source files are taken from `/var/yp/src`, which is typically a symlink to `/etc`.

To regenerate all maps:

```sh
# cd /var/yp
# make
```

The Makefile processes standard maps such as `passwd.byname`, `group.byname`, `hosts.byname`, etc.

To regenerate a single map (e.g., passwd):

```sh
# cd /var/yp
# make passwd
```

## YP Slave Server Configuration

Slave servers receive map updates from the master using `ypxfr(8)`.

To initialize a slave:

```sh
# ypinit -s yp-master-hostname
```

The slave must be part of the same YP domain and must have `/etc/defaultdomain` set accordingly.

Enable and start required daemons:

```sh
# rcctl enable portmap ypserv ypbind
# rcctl start portmap
# rcctl start ypserv
# rcctl start ypbind
```

The master server must allow the slave in `/var/yp/ypservers`. This file should list all slave hostnames that are permitted to pull map updates.

To fetch maps immediately:

```sh
# /usr/libexec/ypxfr passwd.byname
```

## YP Client Configuration

Clients use `ypbind(8)` to bind to a running YP server and request maps.

### Setting the Domain

Ensure `/etc/defaultdomain` contains the correct YP domain:

```sh
# echo "exampledomain" > /etc/defaultdomain
```

Apply immediately:

```sh
# domainname exampledomain
```

### Enabling the Client

Enable and start `portmap` and `ypbind`:

```sh
# rcctl enable portmap ypbind
# rcctl start portmap
# rcctl start ypbind
```

### Verifying Client Operation

Use `ypwhich` to confirm which server is being used:

```sh
$ ypwhich
yp-master.example.net
```

To list available maps:

```sh
$ ypcat -x
```

To dump specific maps:

```sh
$ ypcat passwd.byname
$ ypmatch root passwd.byname
```

### Enabling YP Lookups in `/etc/nsswitch.conf`

The `nsswitch.conf(5)` file determines which sources are used for system lookups.

To use YP for user, group, and host lookups, modify the relevant lines:

```
passwd:     files yp
group:      files yp
netid:      files yp
hosts:      files dns
```

This configuration causes the system to check local files first, then consult YP.

### Logging In with YP Accounts

Users from YP maps can log in normally. Ensure they have home directories and appropriate shells. If home directories are exported over NFS or provisioned on demand, this must be handled separately.

To list available users:

```sh
$ ypcat passwd.byname | cut -d: -f1
```

## Security Considerations

YP transmits all data in plaintext, including password hashes. For this reason:

- Use `passwd.adjunct.byname` to hide encrypted passwords (optional).
- Limit access to YP services using `pf(4)` and network segmentation.
- Do not run YP over untrusted or public networks.
- Avoid using `+::::::` in `/etc/passwd` unless necessary. Prefer `nsswitch.conf`.

In trusted LANs, where SSH is used for access control and physical or VLAN isolation applies, YP remains practical.

## Key Files and Tools

| Path | Purpose |
| --- | --- |
| `/etc/defaultdomain` | Stores the YP domain name |
| `/var/yp` | Contains YP maps |
| `/var/yp/ypservers` | Master’s list of slave servers |
| `/etc/nsswitch.conf` | Controls name lookup order |

| Command | Description |
| --- | --- |
| `ypinit -m` | Initialize YP master server |
| `ypinit -s` | Initialize YP slave server |
| `ypcat` | View contents of a map |
| `ypmatch` | Query a map for a key |
| `ypwhich` | Show the bound YP server |
| `make` (in `/var/yp`) | Rebuild YP maps |
