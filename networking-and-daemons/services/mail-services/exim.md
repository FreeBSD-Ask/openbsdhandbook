# Exim

## 概述

**Exim** 是一款为 Unix 系统设计的高度可配置邮件传输代理（MTA）。最初在剑桥大学开发，功能丰富，对邮件路由、重写与身份验证提供精细控制。Exim 适用于需要深度定制的复杂邮件环境。

本章介绍如何在 OpenBSD 上安装和配置 Exim，涵盖本地投递、TLS 加密、通过 smarthost 中继以及基本内容过滤等场景。Exim 不属于 OpenBSD 基本系统，但以软件包形式提供，在需要更高级邮件逻辑时可替代 Postfix 或 `smtpd(8)`。

## 功能特性

- 完全兼容 Sendmail 风格的命令行
- 精细的路由与重写策略
- 支持 SMTP、submission 与 LMTP
- 支持 TLS、SASL 与 DKIM
- 虚拟域与虚拟用户处理
- 灵活的日志与队列管理

## 安装 Exim

安装 Exim 软件包：

```sh
# pkg_add exim
```

为避免与默认的 `smtpd(8)` 服务冲突，先将其禁用：

```sh
# rcctl disable smtpd
# rcctl stop smtpd
```

启用并启动 Exim：

```sh
# rcctl enable exim
# rcctl start exim
```

Exim 配置位于：

```text
/usr/local/exim/configure
```

若此文件不存在，复制默认模板：

```sh
# cp /usr/local/share/examples/exim/configure.default /usr/local/exim/configure
```

## 配置概览

与 Postfix 不同，Exim 使用**单一的整体配置文件**：`/usr/local/exim/configure`。

该文件分为多个节，包括：

- `MAIN CONFIGURATION SETTINGS`
- `ACL CONFIGURATION`
- `ROUTERS CONFIGURATION`
- `TRANSPORTS CONFIGURATION`
- `RETRY CONFIGURATION`
- `REWRITE CONFIGURATION`
- `AUTHENTICATION CONFIGURATION`

每节控制 Exim 行为的不同方面，指令对顺序敏感。

编辑配置后，测试：

```sh
# exim -bV
# exim -C /usr/local/exim/configure -bV
```

重新加载 Exim 以应用更改：

```sh
# rcctl reload exim
```

## 最小配置示例

以下简单配置适用于本地投递与通过 DNS 发送外发邮件：

### `/usr/local/exim/configure`

```
primary_hostname = mail.example.org
domainlist local_domains = @ : localhost
domainlist relay_to_domains =
hostlist   relay_from_hosts = 127.0.0.1

begin routers
  dnslookup:
    driver = dnslookup
    domains = ! +local_domains
    transport = remote_smtp

begin transports
  remote_smtp:
    driver = smtp

  local_delivery:
    driver = appendfile
    directory = $home/Maildir
    maildir_format

begin retry
*                      F,2h,10m; G,16h,1h,1.5; F,4d,6h

begin rewrite

begin authenticators
```

此配置：

- 对本地用户使用 Maildir 投递
- 通过 DNS MX 解析发送所有外发邮件
- 不为未经身份验证的远程用户中继

## 接收互联网邮件

允许来自外部系统的入站邮件：

1. 确保 `domainlist local_domains` 包含系统的 FQDN 或所需域名：

   ```conf
   domainlist local_domains = example.org : localhost
   ```
2. 确保系统监听外部接口（默认行为）：

   ```sh
   # netstat -an | grep 25
   ```
3. 必要时在 `pf.conf` 中开放 TCP 端口 25。
4. 确保配置了正确的 MX DNS 记录。

## TLS 配置

启用 TLS 前，创建或安装有效的证书与私钥，例如：

- `/etc/ssl/example.org.crt`
- `/etc/ssl/private/example.org.key`

在 `MAIN CONFIGURATION` 节中加入：

```conf
tls_advertise_hosts = *
tls_certificate = /etc/ssl/example.org.crt
tls_privatekey = /etc/ssl/private/example.org.key
```

强制 STARTTLS：

```conf
tls_require_ciphers = HIGH
```

检查 TLS 支持：

```sh
$ openssl s_client -starttls smtp -connect mail.example.org:25
```

## Smarthost（中继）配置

通过服务商中继所有外发邮件：

1. 将身份验证凭据加入 `/usr/local/exim/passwd.client`：

   ```
   mail.example.org:login@example.org:password123
   ```
2. 限制文件权限：

   ```sh
   # chmod 600 /usr/local/exim/passwd.client
   ```
3. 添加使用 smarthost 的路由器与传输：

### Routers 节

```conf
smarthost:
  driver = manualroute
  domains = ! +local_domains
  transport = remote_smtp_smarthost
  route_list = * smtp.provider.net
```

### Transports 节

```conf
remote_smtp_smarthost:
  driver = smtp
  hosts_require_auth = *
  hosts_require_tls = *
```

### Authentication 节

```conf
plain:
  driver = plaintext
  public_name = PLAIN
  client_send = : login@example.org : password123
```

重新加载 Exim：

```sh
# rcctl reload exim
```

## 虚拟域与虚拟用户

对于托管多域并使用虚拟用户的系统，Exim 支持多种查找机制。

使用扁平文件的基本配置：

### `/etc/exim/virtual.aliases`

```
admin@example.org    localuser
info@example.org     infoaccount
```

### Routers 节

```
virtual_aliases:
  driver = redirect
  domains = example.org
  data = ${lookup{$local_part@$domain}lsearch{/etc/exim/virtual.aliases}}
```

创建文件并设置权限：

```sh
# touch /etc/exim/virtual.aliases
# chmod 644 /etc/exim/virtual.aliases
```

## 过滤与 ACL

Exim 提供精细的 ACL，用于阻止垃圾邮件和执行策略。

要拒绝非 TLS 连接或未经身份验证的中继，修改 `acl_smtp_rcpt` 节：

```conf
accept  authenticated = *
accept  hosts = +relay_from_hosts
deny    message = Relay not permitted
```

基本的垃圾邮件标头过滤：

```
deny message = Spam detected
     condition = ${if match{$h_Subject:}{(?i)free money}}
```

Exim 还可通过 `transport_filter` 或 `av_scanner` 选项与 SpamAssassin、rspamd、ClamAV 等外部垃圾邮件过滤器集成。

## 邮件队列与日志

查看队列：

```sh
# exim -bp
```

刷新队列：

```sh
# exim -qff
```

查看日志：

```sh
# tail -f /var/log/exim/mainlog
```

查看邮件日志：

```sh
# exim -Mvh <message-id>
# exim -Mvb <message-id>
```

## 测试

从命令行发送邮件：

```sh
$ echo test | mail -s "Test from Exim" user@example.org
```

检查其是否出现在邮件队列或日志文件中。

## 关键配置文件一览

| 文件 | 用途 |
| --- | --- |
| `/usr/local/exim/configure` | 主配置文件 |
| `/etc/exim/virtual.aliases` | 虚拟别名映射 |
| `/usr/local/exim/passwd.client` | SMTP 身份验证凭据 |
| `/var/log/exim/mainlog` | 主日志文件 |

## 替代方案

Exim 最适合需要对邮件路由、重写与身份验证完全掌控的管理员。对于较简单的场景，[Postfix](postfix.md)
或基本系统的 [smtpd](smtpd.md)
守护进程配置更简便，与 OpenBSD 集成也更紧密。
