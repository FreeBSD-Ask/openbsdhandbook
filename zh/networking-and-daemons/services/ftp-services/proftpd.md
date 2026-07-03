# ProFTPD

## 概述

**ProFTPD** 是一款高度可配置且符合标准的 FTP 服务器，支持虚拟主机、TLS 加密、chroot 会话、匿名与身份验证访问，以及与外部用户数据库集成。

OpenBSD 内置的 `ftpd(8)` 提供精简且安全的 FTP 服务，而 **ProFTPD** 通过软件包提供，适合需要更高级功能或与旧 FTP 客户端兼容的管理员。

本章介绍如何在 OpenBSD 上安装和配置 ProFTPD，包括 TLS 加密与可选的虚拟用户支持。

## FTP 服务器对比

OpenBSD Handbook 文档记录了三种 FTP 服务器实现：

| 特性 | [ftpd](ftpd.md)（基本系统） | [vsftpd](vsftpd.md)（软件包） | [ProFTPD](proftpd.md)（软件包） |
| --- | --- | --- | --- |
| 来源 | 基本系统内置 | 通过 `pkg_add` 安装 | 通过 `pkg_add` 安装 |
| TLS（FTPS）支持 | 否 | 是 | 是 |
| Chroot | 全局 `/ftp` | 按用户（`chroot_local_user`） | 按用户（`DefaultRoot`） |
| 匿名 FTP | 是 | 是 | 是 |
| 虚拟用户 | 否 | 否 | 是（`AuthUserFile` 等） |
| 配置方式 | 仅内置标志 | 扁平配置文件 | 模块化，Apache 风格 |
| 日志 | syslog | xferlog 兼容文件 | syslog 或自定义文件 |
| FTPS 模式 | 不支持 | 显式 FTPS（TLS） | 显式 FTPS（TLS） |
| 资源占用 | 极低 | 低 | 中等 |
| 访问控制 | 极简 | 中等 | 丰富 |
| 适用场景 | 最小安装集 | 安全的公共/私有 FTP | 需精细控制的高级 FTP |

- **[ftpd](ftpd.md)** 适用于可信网络上的简单、仅匿名 FTP。
- **[vsftpd](vsftpd.md)** 适用于需要 TLS 与严格隔离且开销较小的场景。
- **[ProFTPD](proftpd.md)** 适用于需要灵活性、虚拟用户支持与复杂策略实施的环境。

## 安装

ProFTPD 可从 OpenBSD 软件包仓库安装。安装方式如下：

```sh
# pkg_add proftpd
```

此操作会安装主守护进程 `proftpd`、默认配置文件和可选模块。

默认配置文件位于：

```text
/etc/proftpd/proftpd.conf
```

## 基本配置

默认配置启用单个 FTP 站点并采用基础设置。先创建或编辑 `/etc/proftpd/proftpd.conf`：

```
ServerName                      "ProFTPD on OpenBSD"
ServerType                      standalone
DefaultServer                   on

Port                            21
Umask                           022
MaxInstances                    30

User                            _proftpd
Group                           _proftpd

DefaultRoot                    ~
RequireValidShell              off

<Limit LOGIN>
  AllowAll
</Limit>
```

- 服务监听 21 端口。
- `User` 与 `Group` 指定守护进程运行所用的凭据。
- `DefaultRoot ~` 将用户 chroot 到其主目录。
- `RequireValidShell off` 允许无有效 shell（如 `/sbin/nologin`）的用户登录。

创建主目录，并使用非特权系统用户测试：

```sh
# useradd -m -s /sbin/nologin ftpuser
# passwd ftpuser
```

确保属主正确：

```sh
# chown ftpuser:_proftpd /home/ftpuser
```

## 启动服务

手动启动守护进程以测试：

```sh
# /usr/local/sbin/proftpd -c /etc/proftpd/proftpd.conf
```

要在系统启动时自动启动 ProFTPD，在 `/etc/rc.local` 中添加：

```sh
if [ -x /usr/local/sbin/proftpd ]; then
    echo -n ' proftpd'; /usr/local/sbin/proftpd
fi
```

或者创建 `rc.d` 脚本，配合 `rcctl(8)` 使用。

停止服务：

```sh
# pkill proftpd
```

## TLS 支持

ProFTPD 通过 **FTPS** 支持加密 FTP（注意不要与基于 SSH 的 `sftp` 混淆）。

创建 TLS 密钥与证书（自签名或通过 ACME）：

```sh
# mkdir -p /etc/ssl/proftpd
# openssl req -x509 -newkey rsa:4096 -keyout /etc/ssl/proftpd/server.key -out /etc/ssl/proftpd/server.crt -days 365 -nodes
# chmod 600 /etc/ssl/proftpd/server.key
```

然后在配置中启用 TLS：

```
<IfModule mod_tls.c>
  TLSEngine                  on
  TLSLog                     /var/log/proftpd_tls.log
  TLSProtocol                TLSv1.2 TLSv1.3
  TLSRSACertificateFile      /etc/ssl/proftpd/server.crt
  TLSRSACertificateKeyFile   /etc/ssl/proftpd/server.key
  TLSOptions                 NoCertRequest
  TLSVerifyClient            off
  TLSRequired                on
</IfModule>
```

重启守护进程，并使用支持 FTPS 的客户端（如 FileZilla 或 lftp）连接：

```sh
$ lftp ftps://ftpuser@hostname
```

## 匿名 FTP

启用匿名访问：

```
<Anonymous /home/ftp/pub>
  User                          _proftpd
  Group                         _proftpd
  UserAlias                     anonymous ftp
  RequireValidShell             off
  <Limit WRITE>
    DenyAll
  </Limit>
</Anonymous>
```

创建目录：

```sh
# mkdir -p /home/ftp/pub
# chown _proftpd:_proftpd /home/ftp/pub
# chmod 755 /home/ftp/pub
```

匿名用户可从此目录获取文件，但无法上传。

## 虚拟用户

要创建仅存在于 ProFTPD 自身用户数据库（而非系统用户）中的用户，可使用 `mod_auth_file` 模块和 AuthUserFile。

在配置中定义该文件：

```
AuthUserFile /etc/proftpd/ftpd.passwd
AuthGroupFile /etc/proftpd/ftpd.group
AuthOrder mod_auth_file.c
```

使用标准格式生成条目：

```sh
# ftpasswd --passwd --name=vuser1 --home=/srv/ftp/vuser1 --shell=/sbin/nologin --file=/etc/proftpd/ftpd.passwd
# ftpasswd --group --name=ftpgroup --gid=2001 --member=vuser1 --file=/etc/proftpd/ftpd.group
```

创建主目录并设置权限：

```sh
# mkdir -p /srv/ftp/vuser1
# chown _proftpd:_proftpd /srv/ftp/vuser1
```

## 日志

默认情况下，日志写入 `/var/log/proftpd.log` 与 `/var/log/proftpd_tls.log`。可通过 `SystemLog` 与 `TLSLog` 指令调整路径。

示例：

```sh
tail -f /var/log/proftpd.log
```

## 防火墙注意事项

ProFTPD 使用：

- TCP 端口 **21** 用于命令连接
- 一段动态端口范围用于**被动模式数据连接**

为避免防火墙问题，显式定义被动端口范围：

```
PassivePorts 49152 49200
```

然后在 `pf.conf` 中放行这些端口：

```pf
pass in on $int_if proto tcp from any to (self) port 21
pass in on $int_if proto tcp from any to (self) port 49152:49200
```

除非已启用 TLS 且防火墙策略严格，否则不要将 FTP 暴露到互联网。
