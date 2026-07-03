# LDAP

## 概述

**LDAP（Lightweight Directory Access Protocol）** 是一种灵活的、可通过网络访问的目录服务，广泛用于集中管理身份、认证与配置信息。它支持细粒度访问控制、加密传输与可扩展的 schema。

需要明确的是，**LDAP 不同于 YP（NIS）**：

- **YP** 是较老的、基于 SunRPC 的协议，包含在 OpenBSD 基本系统中。
- **LDAP** 是可扩展的工业标准协议（RFC 4511），设计为通过 TCP 运行，可选 TLS。
- OpenBSD 基本系统中**未**包含 LDAP 服务器。可通过软件包获得 **OpenLDAP** 支持。

LDAP 常用于需要安全集中管理用户、组、主机及其他实体的环境。本章介绍如何在 OpenBSD 上搭建并运行 OpenLDAP 服务器、配置客户端工具，以及可选地与系统认证集成。

## 安装

安装 OpenLDAP 服务器与客户端工具：

```sh
# pkg_add openldap-server openldap-client
```

此命令会提供 `slapd(8)` 守护进程以及 `ldapadd`、`ldapmodify`、`ldapsearch`、`slappasswd` 等工具。

配置与数据库目录位于：

- **/etc/openldap/**：配置文件（若使用旧版 `slapd.conf`）
- **/var/openldap-data/**：后端数据库
- **/etc/openldap/slapd.d/**：动态配置树（现代默认方式）

## 配置

OpenLDAP 有两种配置方式：

1. **静态配置**，通过 **/etc/openldap/slapd.conf**（更简单，但已弃用）
2. **动态配置**，通过 `cn=config` 后端

本章为清晰起见使用静态方式。生产环境如有需要，应迁移到动态配置。

### 创建配置文件

**/etc/openldap/slapd.conf** 示例：

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

使用 `slappasswd` 生成安全的哈希密码：

```sh
# slappasswd
New password:
Re-enter new password:
{SSHA}pFhVhOLHbI4YXEbrr4AQF4r+fpdU6xgF
```

将结果填入 `rootpw`。

创建数据库目录：

```sh
# mkdir -p /var/openldap-data
# chown _openldap:_openldap /var/openldap-data
```

## 运行服务器

手动启动服务器进行测试：

```sh
# /usr/local/libexec/slapd -f /etc/openldap/slapd.conf -u _openldap -g _openldap
```

要在启动时启用，将以下内容添加到 **/etc/rc.local**：

```sh
if [ -x /usr/local/libexec/slapd ]; then
    echo -n ' slapd'; /usr/local/libexec/slapd -f /etc/openldap/slapd.conf -u _openldap -g _openldap
fi
```

或者使用自定义 `rc.d` 脚本以集成 `rcctl`。

## 添加初始条目

为基础域创建 LDIF 文件：

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

添加条目：

```sh
# ldapadd -x -D "cn=admin,dc=example,dc=org" -W -f base.ldif
```

按提示输入管理员密码。

## 搜索与查询

搜索目录：

```sh
$ ldapsearch -x -b "dc=example,dc=org"
```

在特定子树中查找用户：

```sh
$ ldapsearch -x -b "ou=people,dc=example,dc=org" "(uid=wouter)"
```

如有需要，用 `-D` 与 `-W` 以绑定用户身份认证。

## TLS 加密

LDAP 可在加密或非加密模式下运行。要保护连接：

1. 在 **/etc/ssl/openldap/** 下创建证书与私钥
2. 在 **slapd.conf** 中添加以下内容：

```
TLSCertificateFile     /etc/ssl/openldap/server.crt
TLSCertificateKeyFile  /etc/ssl/openldap/server.key
TLSCACertificateFile   /etc/ssl/openldap/ca.crt
```

3. 重启 `slapd` 并使用 `ldaps://` 或 StartTLS 连接：

```sh
$ ldapsearch -H ldaps://localhost -x -b "dc=example,dc=org"
```

确保防火墙放行 636 端口（LDAPS）或 389 端口（StartTLS）。

## 与 PAM 和 NSS 集成

LDAP 认证可通过 `nsswitch.conf` 与 `login.conf` 集成，但 OpenBSD 默认**不支持** PAM。

可考虑通过 `nsswitch` 使用 `nss_ldap` 等外部工具，或运行 `nslcd` 等本地辅助程序。

此集成在 OpenBSD 上**并不简单**，应谨慎处理。

OpenBSD 标准的 `login(1)` 不支持直接 LDAP 认证。对更高级的集成，在非 OpenBSD 系统上可能需要代理认证服务（如 Linux 上的 `sssd` 或 `pam_ldap`）。

## 防火墙注意事项

仅在内网接口上允许访问所需 LDAP 端口的 TCP 流量：

```pf
pass in on $int_if proto tcp to port { 389, 636 }
```

在未配备加密、严格 ACL 与认证机制的情况下，**切勿**将 LDAP 服务暴露到公共互联网。

## YP 与 LDAP 对比

| 特性 | YP (NIS) | LDAP |
| --- | --- | --- |
| 协议 | SunRPC | TCP（RFC 4511） |
| 基本系统支持 | 包含在 OpenBSD 中 | 需要软件包 |
| 传输 | 不加密 | 支持 TLS 与 StartTLS |
| 访问控制 | 无（基本映射过滤） | ACL、认证、TLS |
| schema 支持 | 扁平的类 Unix 映射 | 可扩展的对象 schema |
| 推荐用途 | 小型受信任局域网 | 安全的身份基础设施 |
