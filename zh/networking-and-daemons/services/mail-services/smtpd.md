# smtpd

## 概述

OpenSMTPD 是 OpenBSD 默认的邮件传输代理（MTA），以 `smtpd(8)` 守护进程形式实现，设计上安全、简洁，适用于多种场景，包括本地投递、中继、经过身份验证的提交以及为虚拟域接收邮件。

本章介绍如何配置 `smtpd(8)` 以应对常见和高级场景，包括 TLS、过滤以及通过 `opensmtpd-extras` 支持的附加模块。

## 功能特性

- 默认安全，采用权限分离与合理的默认设置
- 投递至系统用户的本地邮箱
- 通过身份验证的提交或 smarthost 中继
- 完整的 IPv6 与 TLS 支持
- 过滤与地址重写
- 虚拟用户/虚拟域支持
- 与 `filter-*` 插件及 extras 软件包集成

## 启用 smtpd

要在系统启动时启用并启动 `smtpd` 服务：

```sh
# rcctl enable smtpd
# rcctl start smtpd
```

守护进程将从 `/etc/mail/smtpd.conf` 读取配置。

## 基本配置

启用本地投递与外发邮件的最简 `smtpd.conf(5)` 文件如下：

```
listen on lo0

action "local" mbox
action "outbound" relay

match from local for local action "local"
match from local for any action "outbound"
```

此配置：

- 仅监听环回接口
- 定义一个本地邮箱投递动作和一个出站中继动作
- 允许本地用户向其他本地用户发信
- 通过 DNS 将本地用户的外发邮件中继到其他系统

应用更改：

```sh
# smtpctl reload
```

## 接收来自互联网的邮件

接收外部邮件：

```pf
pki example.org cert "/etc/mail/certs/example.org.crt"
pki example.org key "/etc/mail/certs/example.org.key"

listen on egress tls pki example.org

action "inbound" maildir
match from any for domain example.org action "inbound"
```

此示例：

- 定义监听器使用的证书与密钥
- 在外部接口上启用 TLS 监听
- 接收发往 `example.org` 域的互联网邮件
- 投递到收件人主目录下的 `~/Maildir`

使用 TLS 时，须将有效证书放置在 `/etc/mail/certs/` 下：

```sh
# mkdir -p /etc/mail/certs
# cp fullchain.pem /etc/mail/certs/example.org.crt
# cp privkey.pem /etc/mail/certs/example.org.key
```

确保权限正确：

```sh
# chown root:_smtpd /etc/mail/certs/*
# chmod 640 /etc/mail/certs/*
```

然后在 `smtpd.conf` 中通过 `pki` 块定义证书：

```
pki example.org cert "/etc/mail/certs/example.org.crt"
pki example.org key "/etc/mail/certs/example.org.key"
```

## 通过 Smarthost 发送外发邮件

通过远程邮件服务器（如 ISP 或上游中继）中继所有外发邮件：

```
action "relay" relay host smtp+auth://user@smtp.example.net auth <secrets>
match for any action "relay"
```

在 `/etc/mail/secrets` 中定义身份验证凭据：

```
user@smtp.example.net password123
```

保护该文件：

```sh
# chmod 600 /etc/mail/secrets
# chown root:_smtpd /etc/mail/secrets
```

## 虚拟域与虚拟用户

接收发往未映射到系统用户的域的邮件，可使用虚拟别名：

### 定义虚拟映射表

创建 `/etc/mail/virtuals`：

```
info@example.org      alice
admin@example.org     bob
contact@example.org   contactuser
```

将其转换为可用表：

```sh
# makemap -t aliases /etc/mail/virtuals > /etc/mail/virtuals.db
```

### 更新 smtpd.conf

```pf
table aliases file:/etc/mail/virtuals.db

action "virtuals" mbox virtual <aliases>
match from any for domain example.org action "virtuals"
```

## 邮件过滤

`filter` 块可对入站邮件进行简单而有效的过滤。示例包括：

### 灰名单

启用基本灰名单：

```
filter "greylist" phase connect match !auth
filter "greylist" phase connect match !relay rdns
filter "greylist" phase connect match !relay sender
filter "greylist" phase connect match !relay helo

listen on egress filter "greylist"
```

### 阻止特定发件人或地址

创建表：

```
table badsenders file:/etc/mail/badsenders.txt
```

内容如下：

```
spammer@example.com
junk@example.net
```

然后：

```
filter "blocklist" phase connect match sender <badsenders> reject
listen on egress filter "blocklist"
```

### 通过 opensmtpd-extras 过滤

安装附加插件：

```sh
# pkg_add opensmtpd-extras
```

此软件包包含：

- `filter-rspamd`：与 Rspamd 集成
- `filter-clamav`：与 ClamAV 集成
- `filter-dkimsign`：用 DKIM 为外发邮件签名
- `filter-dnsbl`：DNS 黑名单检查

DKIM 签名示例：

```
filter "dkim" proc-exec "filter-dkimsign -d example.org -s default -k /etc/mail/dkim.key"

listen on egress filter "dkim"
```

## 本地投递选项

OpenSMTPD 可投递到 `mbox`（传统邮箱格式）或 `maildir`（每封邮件一个文件格式）。在 `action` 中设置方式，并在 `match` 规则中选用：

```pf
action "inbound" maildir
match from any for domain example.org action "inbound"
```

确保主目录中包含相应的 `Maildir/` 或 `mbox`。

## 管理命令

使用 `smtpctl(8)` 管理邮件系统：

```sh
# smtpctl show stats
```

显示已处理邮件数、已处理会话数等统计信息。

```sh
# smtpctl show queue
```

列出邮件队列。

```sh
# smtpctl schedule all
```

立即尝试发送所有排队邮件。

```sh
# smtpctl remove <msg-id>
```

从队列中移除指定邮件。

```sh
# smtpctl log verbose
```

启用详细日志以便调试。

## 日志与排错

日志写入 `/var/log/maillog`。跟踪日志以获取实时更新：

```sh
# tail -f /var/log/maillog
```

获取更多诊断信息：

```sh
# smtpd -dv
```

以调试模式运行 `smtpd`，输出详细信息。

## 关键配置文件一览

| 文件 | 用途 |
| --- | --- |
| `/etc/mail/smtpd.conf` | smtpd 主配置文件 |
| `/etc/mail/secrets` | 中继主机的身份验证凭据 |
| `/etc/mail/virtuals` | 虚拟用户映射（若使用） |
| `/etc/mail/certs/` | TLS 证书与密钥存放处 |
| `/etc/mail/aliases` | 本地用户别名（通过 `newaliases`） |

对 `smtpd.conf` 或相关表的任何更改都需重新加载：

```sh
# smtpctl reload
```
