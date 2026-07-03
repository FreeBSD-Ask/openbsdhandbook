# Rspamd

## 概述

Rspamd 是一款快速、模块化、可扩展的垃圾邮件过滤系统，设计用于高效处理大量邮件。它通过多种方法评估邮件，包括正则表达式、统计分析、SPF/DKIM/DMARC 检查与 URL 信誉。Rspamd 通过 milter 协议或代理 worker 与 `smtpd(8)`、Postfix、Exim 等 MTA 集成。

本章介绍在 OpenBSD 上安装和配置 Rspamd，重点说明与 OpenBSD `smtpd(8)` 及其他受支持邮件传输代理的集成。

## 功能特性

- 垃圾邮件分类与评分
- 内置 DKIM、SPF、DMARC 验证
- 基于 Redis 后端的统计过滤
- 支持 Milter 与 proxy 协议
- 基于 Web 的管理与状态界面
- 用于自定义规则的 Lua 脚本引擎
- 与防病毒、防钓鱼后端集成

## 安装

Rspamd 不包含在 OpenBSD 基本系统中。使用 `pkg_add(1)` 安装：

```sh
# pkg_add rspamd
```

Redis 等依赖与可选 Lua 模块会自动安装。

启用并启动 Rspamd：

```sh
# rcctl enable rspamd
# rcctl start rspamd
```

主配置目录位于：

```text
/etc/rspamd/
```

运行时数据位于：

```text
/var/db/rspamd/
```

## 架构概览

Rspamd 采用多进程架构，由以下组件构成：

- **controller**：处理配置重载、统计与 HTTP 请求
- **normal workers**：执行垃圾邮件检查与过滤
- **proxy workers**：在 MTA 后方以 milter 或 proxy 模式运行时使用
- **rspamc**：用于提交邮件进行分类或学习的客户端工具

## 配置结构

Rspamd 将配置分散到多个文件：

| 文件或目录 | 用途 |
| --- | --- |
| `/etc/rspamd/rspamd.conf` | 主配置文件 |
| `/etc/rspamd/worker-proxy.inc` | 代理 worker 设置 |
| `/etc/rspamd/worker-controller.inc` | 控制器（HTTP 管理界面）设置 |
| `/etc/rspamd/worker-normal.inc` | 标准过滤 worker 设置 |
| `/etc/rspamd/modules.d/` | 各模块配置 |

修改任何配置文件后，重启服务：

```sh
# rcctl restart rspamd
```

## 启用 Web 界面

控制器默认监听 `localhost:11334`。要启用远程访问：

编辑 `/etc/rspamd/worker-controller.inc`：

```
controller {
  bind_socket = "0.0.0.0:11334";
  password = "$2$myadminpasswordhash"; # 使用 rspamadm 生成
  enable_password = "$2$myenablepasswordhash";
  secure_ip = "127.0.0.1";
}
```

使用以下命令生成密码哈希：

```sh
# rspamadm pw
```

建议通过 `secure_ip` 或防火墙规则将访问限制在可信网络。

访问 Web UI：

```
http://localhost:11334/
```

## 与 smtpd(8) 集成

Rspamd 可通过其 proxy 模式与 `smtpd(8)` 的 `filter` 指令集成。

### 示例：/etc/mail/smtpd.conf

```pf
filter rspamd proc-exec "/usr/local/bin/rspamd-milter --socket /tmp/rspamd-milter.sock"

listen on egress tls pki mail.example.org filter "rspamd"
listen on localhost
match from any for domain "example.org" action "local"
```

确保 Rspamd 以 milter 模式运行并监听指定套接字。

编辑 `/etc/rspamd/rspamd.conf`，加入：

```
worker "proxy" {
  bind_socket = "/tmp/rspamd-milter.sock mode=0666";
  timeout = 120s;
  milter = yes;
}
```

然后重启两个服务：

```sh
# rcctl restart rspamd
# rcctl restart smtpd
```

## 与 Postfix 集成

将 Rspamd 作为 milter 与 Postfix 配合使用：

在 `/etc/rspamd/rspamd.conf` 中：

```
worker "proxy" {
  bind_socket = "/var/run/rspamd/rspamd.sock mode=0660 owner=_rspamd group=_postfix";
  milter = yes;
}
```

在 `/etc/postfix/main.cf` 中：

```
smtpd_milters = unix:/var/run/rspamd/rspamd.sock
non_smtpd_milters = unix:/var/run/rspamd/rspamd.sock
milter_protocol = 6
milter_mail_macros = i {mail_addr} {client_addr} {client_name} {auth_authen}
```

然后重启 Postfix：

```sh
# postfix reload
```

## 使用 rspamc 扫描邮件

`rspamc` 工具可用于手动扫描邮件、训练贝叶斯分类器或检查评分：

```sh
$ rspamc -h localhost:11333 < message.eml
$ rspamc learn_spam < spam.eml
$ rspamc learn_ham < ham.eml
```

## 使用 Redis 存储统计数据

Rspamd 使用 Redis 存储统计数据并进行速率限制。安装并启用 Redis：

```sh
# pkg_add redis
# rcctl enable redis
# rcctl start redis
```

在 `/etc/rspamd/local.d/classifier-bayes.conf` 中：

```
backend = "redis";
servers = "127.0.0.1";
```

重启 Rspamd：

```sh
# rcctl restart rspamd
```

## DKIM 签名与验证

### 为外发邮件签名

生成 DKIM 密钥：

```sh
# rspamadm dkim_keygen -d example.org -s mail > mail.dkim.key
# mv mail.dkim.key /etc/rspamd/dkim/example.org.mail.key
```

设置权限：

```sh
# chown _rspamd:_rspamd /etc/rspamd/dkim/example.org.mail.key
# chmod 0400 /etc/rspamd/dkim/example.org.mail.key
```

在 `/etc/rspamd/local.d/dkim_signing.conf` 中：

```
domain {
  example.org {
    selector = "mail";
    path = "/etc/rspamd/dkim/example.org.mail.key";
  }
}
```

在 DNS 中发布 DKIM 公钥（由 `dkim_keygen` 输出）。

### 验证 DKIM

若启用 `dkim.conf` 模块，Rspamd 会自动验证 DKIM。配置通常位于 `/etc/rspamd/modules.d/dkim.conf`。

## DMARC 与 SPF

确保以下模块已启用并配置：

- `/etc/rspamd/modules.d/dmarc.conf`
- `/etc/rspamd/modules.d/spf.conf`

使用公共 DNS 解析器，或配置 `/etc/resolv.conf` 以确保查询正确。

## 贝叶斯训练

Rspamd 从邮件中学习垃圾邮件与非垃圾邮件。通过 `rspamc` 提交：

```sh
$ rspamc learn_spam < spam.eml
$ rspamc learn_ham < ham.eml
```

要从邮件文件夹自动训练，可使用 `rspamadm` 或配置 `neural` 与 `learned` 模块。

## 日志与监控

日志写入：

```text
/var/log/rspamd/rspamd.log
```

使用 `rspamc stat` 查看实时统计：

```sh
$ rspamc stat
```

使用 `rspamadm configdump` 检查生效的配置：

```sh
# rspamadm configdump
```

## 安全注意事项

- 以 `_rspamd` 运行 Rspamd，权限受限
- 使用本地套接字或限制 TCP 监听器访问
- 用强密码与 IP 限制保护控制器界面
- 定期更新配置文件与规则集
- 对入站邮件验证 SPF、DKIM、DMARC 策略

## 文件位置

| 路径 | 说明 |
| --- | --- |
| `/etc/rspamd/` | 配置目录 |
| `/var/log/rspamd/rspamd.log` | 日志文件 |
| `/var/db/rspamd/` | 运行时与统计数据 |
| `/etc/rspamd/dkim/` | DKIM 私钥 |
| `/usr/local/bin/rspamd-milter` | smtpd(8) 的 milter 助手 |
