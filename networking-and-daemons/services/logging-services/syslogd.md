# syslogd

## 概述

`syslogd(8)` 守护进程负责收集并分发 OpenBSD 系统及其服务产生的日志消息。它通过 `syslog(3)` 接口接收来自内核与用户进程的日志条目，并按照配置文件 **/etc/syslog.conf** 中定义的规则进行转发。

默认情况下，`syslogd` 将消息写入 **/var/log/** 下的文本文件，也可通过 UDP 或 TLS 加密的 TCP 将消息转发至远程日志主机。在 OpenBSD 上，`syslogd` 采用严格的权限分离与文件系统访问限制，并默认启用。

本章介绍如何自定义日志记录行为、过滤与重定向日志、启用远程日志，以及运行时控制日志行为。

## 默认行为

OpenBSD 启动时会自动启动 `syslogd`。守护进程从 **/etc/syslog.conf** 读取配置，并通过 Unix 域套接字 **/dev/log** 开始监听本地日志消息。

`newsyslog(8)` 会按照 **/etc/newsyslog.conf** 中的设置定期轮转日志文件。

确认 `syslogd` 是否在运行：

```sh
$ ps -aux | grep syslogd
```

通过 `rcctl(8)` 检查状态：

```sh
# rcctl check syslogd
```

## 配置：/etc/syslog.conf

**/etc/syslog.conf** 文件定义了各类消息的写入位置。每行包含一个 *选择器*（facility.level）与一个 *动作*。例如：

```
auth.info               /var/log/authlog
cron.*                  /var/log/cron
mail.err                /dev/console
*.notice;auth,authpriv.none  /var/log/messages
```

上述配置完成以下工作：

- 将 info 级别及以上的认证日志写入 **/var/log/authlog**
- 将所有 `cron` 消息记录到 **/var/log/cron**
- 将邮件错误发送至控制台
- 将除 `auth` 与 `authpriv` 以外的所有 notice 消息记录到 **/var/log/messages**

应用更改：

```sh
# kill -HUP $(cat /var/run/syslog.pid)
```

### 设施与级别

设施（facility）将相关服务归类（如 `auth`、`cron`、`mail`）。级别（level）表示严重程度，从最严重到最轻微：

- `emerg`——系统不可用
- `alert`——必须立即采取措施
- `crit`——严重状况
- `err`——错误状况
- `warning`——警告
- `notice`——正常但重要
- `info`——信息性
- `debug`——调试级消息

特殊级别 `none` 用于排除某个设施。

## 远程日志

要将日志转发至远程 syslog 服务器，在 **/etc/syslog.conf** 中添加如下行：

```
*.*    @loghost.example.net
```

若需 TLS 加密的日志传输，使用 `tls` 前缀：

```
*.*    tls://loghost.example.net
```

然后重启 `syslogd`：

```sh
# rcctl restart syslogd
```

远程日志需要 DNS 解析与网络访问。若启用了包过滤，需在 `pf.conf` 中配置相应规则。

### 接收远程日志

要接收来自其他主机的日志，通过 `rcctl` 为 `syslogd` 添加 `-u` 或 `-U` 标志：

```sh
# rcctl set syslogd flags -u
# rcctl restart syslogd
```

`-u` 标志启用未加密的 UDP 接收。若要使用加密的 TCP 日志（TLS），使用 `-T` 并按 `syslogd(8)` 的说明配置证书。

## 运行时控制

在 chroot 或隔离环境等场景下需要禁用 `syslogd`：

```sh
# rcctl disable syslogd
# rcctl stop syslogd
```

之后若要重新启用：

```sh
# rcctl enable syslogd
# rcctl start syslogd
```

## 查看与管理日志

多数日志文件存放在 **/var/log/** 下。常见文件包括：

- **/var/log/messages**——常规系统活动
- **/var/log/authlog**——认证事件
- **/var/log/daemon**——长期运行服务的消息
- **/var/log/cron**——计划任务输出
- **/var/log/maillog**——邮件子系统消息

实时跟踪日志输出：

```sh
# tail -f /var/log/messages
```

## 日志轮转与保留

日志轮转由 `newsyslog(8)` 处理，每日由 **/etc/daily** 调用。配置文件 **/etc/newsyslog.conf** 决定哪些日志需要轮转、保留多少归档以及何时压缩。

典型条目：

```
/var/log/messages     600  5     100  *     Z
```

此设置保留 5 个 **/var/log/messages** 的压缩归档，并在日志超过 100 KB 时进行轮转。

## 故障排查

日志文件为空或缺失时：

- 确认 `syslogd` 正在运行
- 确认 `pf(4)` 未阻断出入站的 syslog 流量
- 检查 **/etc/syslog.conf** 中是否存在相关选择器
- 确认 **/var** 下有足够的磁盘空间

测试日志消息是否能送达守护进程：

```sh
$ logger -p user.info "This is a test log entry"
```

然后查看相应日志文件（如 **/var/log/messages**）中是否出现该测试条目。
