# vsftpd

## 概述

**vsftpd（Very Secure FTP Daemon）** 是一款精简、高性能的 FTP 服务器，高度重视安全性。在资源占用低、正确性比灵活性更重要的环境中常被采用。

与 `ProFTPD` 不同，`vsftpd` 原生不支持丰富的模块系统或虚拟用户数据库。但能提供简单、稳健的 FTP 服务，支持：

- 主动与被动 FTP 模式
- 匿名与本地用户登录
- 用户 chroot
- FTPS 加密

vsftpd 不属于 OpenBSD 基本系统，需从软件包安装。

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

通过软件包系统安装 `vsftpd`：

```sh
# pkg_add vsftpd
```

此操作会安装守护进程 `vsftpd` 和默认配置文件：

```text
/etc/vsftpd.conf
```

## 基本配置

以下为支持匿名访问与系统用户登录的最小配置示例：

```conf
listen=YES
listen_ipv6=NO

anonymous_enable=YES
local_enable=YES
write_enable=NO
local_umask=022

chroot_local_user=YES

ftpd_banner=Welcome to OpenBSD vsftpd

# 限制被动端口范围，便于防火墙配置
pasv_min_port=49152
pasv_max_port=49200

# 使用非特权用户
nopriv_user=_vsftpd

# 安全的日志位置
xferlog_enable=YES
xferlog_file=/var/log/vsftpd.log
```

为匿名访问创建主目录：

```sh
# mkdir -p /var/ftp/pub
# chown root:wheel /var/ftp
# chown -R _vsftpd:_vsftpd /var/ftp/pub
```

若要允许系统用户登录（例如 `ftpuser`）：

```sh
# useradd -m -s /sbin/nologin ftpuser
# passwd ftpuser
# mkdir /home/ftpuser/incoming
# chown ftpuser:ftpuser /home/ftpuser/incoming
```

## 启动服务

手动启动 `vsftpd` 以测试：

```sh
# /usr/local/sbin/vsftpd /etc/vsftpd.conf
```

要在系统启动时自动运行，追加到 `/etc/rc.local`：

```sh
if [ -x /usr/local/sbin/vsftpd ]; then
    echo -n ' vsftpd'; /usr/local/sbin/vsftpd /etc/vsftpd.conf
fi
```

或者创建自定义 `rc.d` 脚本，配合 `rcctl` 使用。

## FTPS（TLS）支持

vsftpd 支持 FTPS（基于 SSL/TLS 的 FTP）。启用方法：

1. 生成或获取证书与密钥对。

```sh
# mkdir -p /etc/ssl/vsftpd
# openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout /etc/ssl/vsftpd/server.key \
  -out /etc/ssl/vsftpd/server.crt \
  -days 365
```

2. 设置属主与权限：

```sh
# chmod 600 /etc/ssl/vsftpd/server.key
# chmod 644 /etc/ssl/vsftpd/server.crt
```

3. 修改配置：

```conf
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES

rsa_cert_file=/etc/ssl/vsftpd/server.crt
rsa_private_key_file=/etc/ssl/vsftpd/server.key

ssl_tlsv1_2=YES
ssl_tlsv1_3=YES
```

重启 `vsftpd`。此后 lftp 或 FileZilla 等 FTPS 客户端即可安全连接。

## 主动与被动 FTP

默认情况下，vsftpd 支持**被动模式**，推荐用于大多数位于防火墙后的环境。

限制被动数据连接的端口范围：

```conf
pasv_min_port=49152
pasv_max_port=49200
```

OpenBSD 的 `pf.conf` 应更新以放行控制端口与数据端口：

```pf
pass in on $int_if proto tcp from any to (self) port 21
pass in on $int_if proto tcp from any to (self) port 49152:49200
```

主动 FTP 要求客户端防火墙允许入站连接，这通常难以实现。

## 日志与监控

若设置 `xferlog_enable=YES`，vsftpd 会以 xferlog 格式将日志写入 `/var/log/vsftpd.log`。

示例：

```sh
# tail -f /var/log/vsftpd.log
```

测试访问：

```sh
$ ftp localhost
$ lftp ftp://ftpuser@localhost
```

使用 `tcpdump` 或 `netstat -an` 验证连接尝试与端口占用。
