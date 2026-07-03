# LDAP

## Synopsis

**LDAP (Lightweight Directory Access Protocol)** is a flexible, network-accessible directory service widely used to centralize identity, authentication, and configuration information. It supports fine-grained access control, encrypted transport, and extensible schemas.

It is important to clarify that **LDAP is not the same as YP (NIS)**:

- **YP** is an older, SunRPC-based protocol included in the OpenBSD base system.
- **LDAP** is an extensible, industry-standard protocol (RFC 4511), designed to operate over TCP and optionally with TLS.
- OpenBSD does **not** include an LDAP server in base. Support for **OpenLDAP** is available via packages.

LDAP is commonly used in environments requiring secure and centralized management of users, groups, hosts, and other entities. This chapter documents how to set up and run an OpenLDAP server on OpenBSD, configure client tools, and optionally integrate with system authentication.

## Installation

Install the OpenLDAP server and client tools:

```sh
# pkg_add openldap-server openldap-client
```

This provides the `slapd(8)` daemon and tools such as `ldapadd`, `ldapmodify`, `ldapsearch`, and `slappasswd`.

The configuration and database directories will be located under:

- `/etc/openldap/`: Configuration files (if using legacy `slapd.conf`)
- `/var/openldap-data/`: Backend database
- `/etc/openldap/slapd.d/`: Dynamic configuration tree (modern default)

## Configuration

OpenLDAP can be configured in two ways:

1. **Static configuration** via `/etc/openldap/slapd.conf` (simpler, but deprecated)
2. **Dynamic configuration** via the `cn=config` backend

This chapter uses the static method for clarity. For production use, migrate to dynamic configuration where required.

### Creating the Configuration File

Example `/etc/openldap/slapd.conf`:

```
include         /etc/openldap/schema/core.schema

pidfile         /var/run/slapd.pid
argsfile        /var/run/slapd.args

loglevel        stats

modulepath      /usr/local/lib/openldap
moduleload      back_mdb.la

backend         mdb
database        mdb
maxsize         1073741824
suffix          "dc=example,dc=org"
rootdn          "cn=admin,dc=example,dc=org"
rootpw          {SSHA}xxxxxxxxxxxxxxxxxxxxxxx
directory       /var/openldap-data
```

Use `slappasswd` to generate a secure hashed password:

```sh
# slappasswd
New password:
Re-enter new password:
{SSHA}pFhVhOLHbI4YXEbrr4AQF4r+fpdU6xgF
```

Insert the result as `rootpw`.

Create the database directory:

```sh
# mkdir -p /var/openldap-data
# chown _openldap:_openldap /var/openldap-data
```

## Running the Server

To start the server manually for testing:

```sh
# /usr/local/libexec/slapd -f /etc/openldap/slapd.conf -u _openldap -g _openldap
```

To enable it at boot, add the following to `/etc/rc.local`:

```sh
if [ -x /usr/local/libexec/slapd ]; then
    echo -n ' slapd'; /usr/local/libexec/slapd -f /etc/openldap/slapd.conf -u _openldap -g _openldap
fi
```

Or use a custom `rc.d` script for `rcctl` integration.

## Adding Initial Entries

Create an LDIF file for the base domain:

```ldif
dn: dc=example,dc=org
objectClass: top
objectClass: dcObject
objectClass: organization
o: Example Organization
dc: example

dn: cn=admin,dc=example,dc=org
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: Directory administrator
userPassword: {SSHA}xxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Add entries with:

```sh
# ldapadd -x -D "cn=admin,dc=example,dc=org" -W -f base.ldif
```

Enter the admin password when prompted.

## Searching and Querying

To search the directory:

```sh
$ ldapsearch -x -b "dc=example,dc=org"
```

To look for users in a specific subtree:

```sh
$ ldapsearch -x -b "ou=people,dc=example,dc=org" "(uid=wouter)"
```

Use `-D` and `-W` to authenticate as a bind user if necessary.

## TLS Encryption

LDAP can operate with or without encryption. To secure connections:

1. Create a certificate and private key under `/etc/ssl/openldap/`
2. Add the following to `slapd.conf`:

```
TLSCertificateFile     /etc/ssl/openldap/server.crt
TLSCertificateKeyFile  /etc/ssl/openldap/server.key
TLSCACertificateFile   /etc/ssl/openldap/ca.crt
```

3. Restart `slapd` and connect using `ldaps://` or StartTLS:

```sh
$ ldapsearch -H ldaps://localhost -x -b "dc=example,dc=org"
```

Ensure port 636 (LDAPS) or port 389 (StartTLS) is allowed through any firewalls.

## Integration with PAM and NSS

LDAP authentication can be integrated via `nsswitch.conf` and `login.conf`, but OpenBSD does **not** support PAM by default.

Instead, consider external tools such as `nss_ldap` via `nsswitch`, or run a local helper such as `nslcd`.

This integration is **non-trivial** on OpenBSD and should be handled with care.

OpenBSD’s standard `login(1)` does not support direct LDAP authentication. For more advanced integration, a proxy authentication service (such as `sssd` or `pam_ldap` on Linux) may be needed on non-OpenBSD systems.

## Firewall Notes

Allow TCP access to the desired LDAP port(s) on internal interfaces only:

```pf
pass in on $int_if proto tcp to port { 389, 636 }
```

Do **not** expose LDAP services to the public internet without encryption, strict ACLs, and authentication mechanisms in place.

## YP vs LDAP Recap

| Feature | YP (NIS) | LDAP |
| --- | --- | --- |
| Protocol | SunRPC | TCP (RFC 4511) |
| Base Support | Included in OpenBSD | Requires package |
| Transport | Unencrypted | TLS and StartTLS supported |
| Access Control | None (basic map filtering) | ACLs, authentication, TLS |
| Schema Support | Flat Unix-like maps | Extensible object schema |
| Recommended Use | Small trusted LANs | Secure identity infrastructure |
