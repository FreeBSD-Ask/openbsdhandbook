# Postfix

## 概述

Postfix 是一款功能齐全的邮件传输代理（MTA），以性能优异、配置便捷、安全性强著称。在 OpenBSD 上以软件包形式提供，需要更高级能力时可替代默认的 `smtpd(8)` 守护进程，例如兼容复杂邮件环境、精细的策略控制或详尽的日志。

本章详细介绍如何在 OpenBSD 上安装和配置 Postfix，涵盖收发邮件、配置 TLS 加密、通过 smarthost 中继、设置虚拟域与虚拟用户，以及集成垃圾邮件与防病毒过滤器等常见任务。

## 功能特性

- 安全且模块化的设计
- 支持 SMTP、SMTPS 与 submission 端口
- 通过 TLS 进行身份验证与加密
- 丰富的队列管理
- 通过 `main.cf` 与 `master.cf` 进行灵活配置
- 支持别名、虚拟域与 LDAP 集成
- 与 `dovecot`、`spamassassin`、`amavisd`、`rspamd` 集成

## 安装 Postfix

从软件包安装 Postfix：

```sh
# pkg_add postfix
```

为避免与 OpenSMTPD 冲突，若已启用 `smtpd(8)`，先将其禁用：

```sh
# rcctl disable smtpd
# rcctl stop smtpd
```

启用并启动 Postfix 服务：

```sh
# rcctl enable postfix
# rcctl start postfix
```

主配置目录为：

```sh
/etc/postfix/
```

## 配置

Postfix 使用两个主要配置文件：

- `/etc/postfix/main.cf`：主配置
- `/etc/postfix/master.cf`：服务定义与进程控制

修改后重新加载配置：

```sh
# postfix reload
```

检查语法与配置：

```sh
# postfix check
```

## 最小配置

以下基本设置足以实现本地投递与通过 DNS 中继外发邮件：

```conf
myhostname = mail.example.org
myorigin = /etc/mailname
mydestination = localhost
relayhost =
inet_interfaces = loopback-only
inet_protocols = all
```

此配置：

- 设置邮件主机名
- 通过 DNS 中继外发邮件
- 仅接受本机邮件

要启用对本地用户的投递，确保 `mailbox_command` 或 `mailbox_transport` 设置得当，例如：

```conf
home_mailbox = Maildir/
```

确保每个用户都有 `~/Maildir/` 目录。

## 接收互联网邮件

接收来自远程系统的邮件：

```conf
inet_interfaces = all
myhostname = mail.example.org
mydestination = mail.example.org, localhost.localdomain, localhost
```

将入站邮件限制为特定域：

```conf
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination
```

确保 `example.org` 的 MX 记录指向正确的公网 IP 地址。

## TLS 加密

为收发邮件启用 TLS：

```conf
smtpd_tls_cert_file = /etc/ssl/example.org.crt
smtpd_tls_key_file = /etc/ssl/private/example.org.key
smtpd_use_tls = yes
smtpd_tls_auth_only = yes

smtp_tls_security_level = may
smtp_tls_note_starttls_offer = yes
```

确保权限正确：

```sh
# chown root:_postfix /etc/ssl/private/example.org.key
# chmod 640 /etc/ssl/private/example.org.key
```

强制所有客户端使用加密：

```conf
smtpd_tls_security_level = encrypt
```

## 通过 Smarthost 发送外发邮件

通过服务商中继所有外发邮件：

```conf
relayhost = [smtp.example.net]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_use_tls = yes
```

创建 `/etc/postfix/sasl_passwd`：

```ini
[smtp.example.net]:587 user@example.net:password123
```

转换并保护该文件：

```sh
# postmap /etc/postfix/sasl_passwd
# chmod 600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
# chown root:_postfix /etc/postfix/sasl_passwd*
```

重新加载 Postfix：

```sh
# postfix reload
```

## 虚拟域与虚拟用户

为未映射到系统账户的域和用户托管邮件：

### 定义 `/etc/postfix/virtual`

```
admin@example.org    adminuser
sales@example.org    salesuser
info@example.org     infoaccount
```

创建并映射该文件：

```sh
# postmap /etc/postfix/virtual
```

更新 `main.cf`：

```conf
virtual_alias_domains = example.org
virtual_alias_maps = hash:/etc/postfix/virtual
```

重新加载 Postfix：

```sh
# postfix reload
```

## 邮件过滤

Postfix 通过内容检查与 milter 集成支持多种内容过滤器。

### 基本标头与正文检查

要阻止特定主题或正文，使用：

- `/etc/postfix/header_checks`
- `/etc/postfix/body_checks`

在 `main.cf` 中启用：

```conf
header_checks = regexp:/etc/postfix/header_checks
body_checks = regexp:/etc/postfix/body_checks
```

### 与 SpamAssassin 和 ClamAV 集成

安装 `amavisd`：

```sh
# pkg_add amavisd-new spamassassin clamav
```

配置 `main.cf`：

```conf
content_filter = smtp-amavis:[127.0.0.1]:10024
```

编辑 `/etc/postfix/master.cf`，加入：

```conf
smtp-amavis unix - - n - 2 smtp
  -o smtp_data_done_timeout=1200
  -o smtp_send_xforward_command=yes
  -o disable_dns_lookups=yes

127.0.0.1:10025 inet n - n - - smtpd
  -o content_filter=
  -o receive_override_options=no_unknown_recipient_checks
  -o smtpd_helo_restrictions=
  -o smtpd_client_restrictions=
  -o smtpd_sender_restrictions=
  -o smtpd_recipient_restrictions=permit_mynetworks,reject
  -o mynetworks=127.0.0.0/8
  -o smtpd_authorized_xforward_hosts=127.0.0.0/8
```

## 管理命令

检查队列：

```sh
# mailq
```

刷新队列：

```sh
# postfix flush
```

查看日志：

```sh
# tail -f /var/log/maillog
```

检查配置：

```sh
# postconf -n
```

## 测试

发送测试邮件：

```sh
$ mail -s "Test Postfix" user@example.org < /dev/null
```

检查 Postfix 是否在监听：

```sh
# netstat -an | grep 25
```

验证 TLS：

```sh
$ openssl s_client -starttls smtp -connect mail.example.org:25
```

## 关键配置文件一览

| 文件 | 用途 |
| --- | --- |
| `/etc/postfix/main.cf` | 主配置 |
| `/etc/postfix/master.cf` | 守护进程/服务进程配置 |
| `/etc/postfix/virtual` | 虚拟地址映射 |
| `/etc/postfix/sasl_passwd` | SMTP 身份验证凭据 |
| `/var/log/maillog` | Postfix 日志输出 |
| `/etc/postfix/header_checks` | 用于过滤邮件标头的正则 |
| `/etc/postfix/body_checks` | 用于过滤邮件正文的正则 |

```sh
# postfix reload
```

修改配置文件后务必重新加载。

## 替代方案

Postfix 适用于大多数场景。对于追求简洁与紧密 OpenBSD 集成的系统，可选用 `smtpd(8)`（OpenSMTPD）等更轻量的替代方案。
