# Dovecot

## 概述

[Dovecot](https://dovecot.org/)
是一款安全且高性能的邮件投递代理（MDA）和 IMAP/POP3 服务器。常与 `smtpd(8)`、Postfix 或 Exim 等邮件传输代理（MTA）配合使用，通过 IMAP 或 POP3 协议访问本地邮箱。Dovecot 支持标准邮箱格式（`mbox`、`Maildir`）、安全身份验证（包括 PAM 与 SQL 后端）以及完善的 TLS 集成。

本章介绍如何在 OpenBSD 上安装、配置和启用 Dovecot，覆盖常见部署场景，包括本地投递、虚拟邮箱与安全的 IMAP 访问。

## 功能特性

- IMAP4rev1 与 POP3 协议，完整支持 TLS
- 邮箱格式支持：`mbox`、`Maildir`、`dbox`
- 通过 PAM、passwd、SQL 或静态 userdb 进行身份验证
- 内置 LMTP 服务器，用于本地投递
- 支持虚拟用户与 chroot 邮件存储
- 内置 Sieve 过滤（通过插件）
- 高效索引与快速邮件访问
- 与 `smtpd(8)`、Postfix、Exim 兼容

## 安装 Dovecot

Dovecot 不包含在 OpenBSD 基本系统中。使用 `pkg_add(1)` 安装：

```sh
# pkg_add dovecot
```

此操作会安装 Dovecot 守护进程及示例配置文件到 `/etc/dovecot/`。

## 启用服务

启用 Dovecot 在系统启动时运行：

```sh
# rcctl enable dovecot
# rcctl start dovecot
```

主配置文件为 `/etc/dovecot/dovecot.conf`。附加配置位于 `/etc/dovecot/conf.d/`。

## 配置

Dovecot 采用模块化配置系统。各主要子系统（如协议、身份验证、邮件存储）在 `conf.d/` 中均有独立配置文件，通过主 `dovecot.conf` 中的 `!include conf.d/*.conf` 引入。

### dovecot.conf 示例

```conf
log_path = /var/log/dovecot.log
protocols = imap pop3 lmtp
disable_plaintext_auth = yes
mail_privileged_group = _dovecot
!include conf.d/*.conf
```

### 邮件位置

定义邮箱存储格式与位置。系统用户在主目录下使用 `Maildir`：

文件：`/etc/dovecot/conf.d/10-mail.conf`

```conf
mail_location = maildir:~/Maildir
mail_access_groups = _dovecot
```

或者使用 `mbox`：

```conf
mail_location = mbox:~/mail:INBOX=/var/mail/%u
```

确保每个用户都初始化了 `Maildir/` 或 `mail/` 目录。

### 身份验证

文件：`/etc/dovecot/conf.d/10-auth.conf`

对系统用户（来自 `/etc/passwd`）进行身份验证：

```conf
disable_plaintext_auth = yes
auth_mechanisms = plain login
!include auth-system.conf.ext
```

`auth-system.conf.ext` 文件配置针对 `/etc/passwd` 与 `login.conf` 的身份验证。

对于虚拟用户（如 SQL 或 passwd 风格文件），参见 `auth-passwdfile.conf.ext`。

### TLS 配置

文件：`/etc/dovecot/conf.d/10-ssl.conf`

```conf
ssl = required
ssl_cert = </etc/ssl/mail.example.org.crt
ssl_key  = </etc/ssl/private/mail.example.org.key
```

证书可通过 [acme-client(1)] 或其他 ACME 实现获取。Dovecot 在放弃 root 权限之前打开证书与密钥，因此私钥仅 `root` 可读（例如模式 `0600`）。

### 监听器配置

文件：`/etc/dovecot/conf.d/10-master.conf`

定义 IMAP、POP3 与 LMTP 的套接字监听器。

监听所有接口：

```
service imap-login {
  inet_listener imap {
    port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}

service pop3-login {
  inet_listener pop3 {
    port = 110
  }
  inet_listener pop3s {
    port = 995
    ssl = yes
  }
}
```

禁用 POP3：

```conf
protocols = imap lmtp
```

### 从 smtpd 或 Postfix 通过 LMTP 投递

文件：`/etc/dovecot/conf.d/10-master.conf`

```
service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = _postfix
    group = _postfix
  }
}
```

在 `smtpd.conf` 中配置 LMTP 投递：

```
action "local" lmtp "/var/spool/postfix/private/dovecot-lmtp"
match for local action "local"
```

## 邮箱初始化

每个用户都须正确初始化邮箱目录。对于 `Maildir`：

```sh
$ maildirmake ~/Maildir
$ chmod -R go-rwx ~/Maildir
```

用户登录时，Dovecot 会自动创建缺失的子目录。

## 测试配置

检查 Dovecot 配置是否存在语法错误：

```sh
# dovecot -n
# dovecot -a
```

跟踪日志以发现身份验证或邮件访问问题：

```sh
# tail -f /var/log/dovecot.log
```

## 使用 TLS

使用 `openssl` 验证基于 TLS 的 IMAP：

```sh
$ openssl s_client -connect mail.example.org:993
```

连接与身份验证成功后，Dovecot 会记录相应日志。

## 用户身份验证示例

### 系统用户

若用户已存在于 `/etc/passwd` 且主目录下有 Maildir，则无需额外配置。

### 通过 passwd-file 实现虚拟用户

创建 `/etc/dovecot/users`：

```
user1@example.org:{PLAIN}password1:1001:1001::/var/vmail/user1::userdb_mail=maildir:/var/vmail/user1/Maildir
```

文件：`/etc/dovecot/conf.d/auth-passwdfile.conf.ext`

```
passdb {
  driver = passwd-file
  args = scheme=PLAIN /etc/dovecot/users
}

userdb {
  driver = static
  args = uid=vmail gid=vmail home=/var/vmail/%n
}
```

设置属主与权限：

```sh
# chown _dovecot:_dovecot /etc/dovecot/users
# chmod 0600 /etc/dovecot/users
```

## Dovecot Sieve 支持

通过插件安装 Sieve 支持：

```sh
# pkg_add dovecot-pigeonhole
```

编辑 `/etc/dovecot/conf.d/20-managesieve.conf`：

```
protocols = $protocols sieve

service managesieve-login {
  inet_listener sieve {
    port = 4190
  }
}

plugin {
  sieve = ~/.dovecot.sieve
  sieve_dir = ~/sieve
}
```

重启 Dovecot 以应用更改。

## 管理工具

- `doveadm user`：列出已知用户
- `doveadm mailbox list`：显示用户邮箱
- `doveadm auth test`：验证登录身份验证
- `doveadm log errors`：显示最近的错误
- `doveadm stats`：显示性能指标
- `doveadm reload`：重新加载配置
- `doveadm kick`：断开用户连接

示例：

```sh
# doveadm auth test alice@example.org password
passdb: alice@example.org auth succeeded
```

## 文件位置

| 文件 | 用途 |
| --- | --- |
| `/etc/dovecot/dovecot.conf` | 主配置文件 |
| `/etc/dovecot/conf.d/` | 模块化子系统配置 |
| `/var/log/dovecot.log` | 主日志文件 |
| `/etc/dovecot/users` | 虚拟用户的 passwd 风格文件 |
| `/etc/ssl/mail.example.org.*` | TLS 证书与密钥 |
| `/var/vmail/` | 虚拟用户主目录根 |

## 安全与加固

- 除非在 TLS 下，否则不允许明文登录（`disable_plaintext_auth = yes`）
- 以专用用户（`_dovecot`）运行 Dovecot
- 使用强 TLS 证书
- 尽可能为虚拟邮件存储启用 `auth_chroot`
- 限制 `/etc/dovecot/users` 等敏感文件的权限
