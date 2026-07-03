# ftpd

## 概述

OpenBSD 完整支持文件传输协议（FTP），既提供命令行客户端 `ftp(1)`，也提供简洁安全的 FTP 服务器 `ftpd(8)`。FTP 已基本被 `sftp`（通过 `sshd`）和 `httpd` 等安全方案取代，但在兼容旧系统、自动化任务和精简环境中仍有用武之地。

本章介绍 OpenBSD 上 FTP 客户端与服务器的用法，并对比 FTP 与替代方案，说明主动模式与被动模式的区别，给出常用操作示例。

## 功能特性

- `ftp(1)` 同时支持 FTP 与 HTTP 下载
- 支持主动模式和被动模式
- 支持匿名访问与身份验证访问
- `ftpd(8)` 默认配置安全，支持 chroot
- 适用于自动化与脚本
- 包含于 OpenBSD 基本系统

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

## FTP 客户端

基本系统附带 `ftp(1)`，是一款安全且适合脚本使用的命令行工具，可通过 FTP 与 HTTP 获取文件。它支持被动与主动传输模式、基于 URL 的调用、HTTP 重定向和代理。

### 基本用法

通过 FTP 下载文件：

```sh
$ ftp ftp://ftp.openbsd.org/pub/OpenBSD/7.8/README
```

通过 HTTP 下载文件：

```sh
$ ftp https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/install78.img
```

下载并重命名：

```sh
$ ftp -o obsd.img https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/install78.img
```

使用通配符下载多个文件：

```sh
$ ftp ftp://ftp.example.com/pub/*.txt
```

为防止 shell 进行通配符展开，需用引号包裹。

### 交互模式

不带 URL 调用时，`ftp(1)` 进入交互模式：

```sh
$ ftp
ftp> open ftp.openbsd.org
ftp> login anonymous
ftp> cd pub/OpenBSD/7.8/amd64
ftp> get install78.img
ftp> quit
```

此模式下可使用标准 FTP 命令，如 `cd`、`get`、`put`、`ls` 与 `bye`。

### 常用标志

- `-p`：强制使用被动模式（默认）
- `-A`：使用主动模式
- `-o <filename>`：将输出写入指定文件
- `-M`：镜像目录
- `-R`：自动重试失败的传输
- `-V`：详细模式

### 主动模式与被动模式

FTP 建立数据连接有**主动**与**被动**两种模式，选择何种模式会影响防火墙与 NAT 兼容性。

#### 主动模式

主动模式下，**客户端告知服务器**回连的 IP 地址和端口。该方式常被客户端防火墙或 NAT 设备拦截。

流程：

1. 客户端连接到服务器的 TCP 端口 21（控制连接）。
2. 客户端发送 `PORT` 命令，告知自己的 IP 与端口。
3. 服务器向客户端发起新连接（数据连接）。

```sh
$ ftp -A ftp://ftp.example.com
```

#### 被动模式

被动模式下，**服务器告知客户端**应连接的端口。客户端同时发起控制连接和数据连接。OpenBSD 默认采用此模式。

流程：

1. 客户端连接到服务器的端口 21。
2. 客户端发送 `PASV` 命令。
3. 服务器回复 IP 与端口。
4. 客户端连接到服务器的端口（数据连接）。

```sh
$ ftp -p ftp://ftp.example.com
```

除非特定服务器要求主动模式，否则推荐使用被动模式。

### 代理支持

在代理后使用 `ftp(1)`：

设置 `FTP_PROXY` 或 `HTTP_PROXY` 环境变量：

```sh
$ export FTP_PROXY=http://proxy.example.com:8080
$ ftp ftp://ftp.example.com
```

### 脚本化下载

下载脚本示例：

```sh
#!/bin/sh
URL="https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/install78.img"
OUT="install78.img"

ftp -o "$OUT" "$URL"
```

### HTTP 身份验证

对于需要密码的 HTTP URL：

```sh
$ ftp https://user:pass@example.com/private/file.tar.gz
```

凭据会出现在进程列表中，请谨慎使用。

## FTP 服务器

基本系统附带 `ftpd(8)`，这是一款安全的 FTP 守护进程，适用于可信网络或 chroot 环境中的基本文件服务。该服务器同时支持匿名会话与身份验证会话。

### 启用服务器

启用并启动 FTP 守护进程：

```sh
# rcctl enable ftpd
# rcctl start ftpd
```

### 配置选项

默认行为无需配置文件，通过 `rcctl(8)` 传入的标志控制。

查看当前设置：

```sh
# rcctl get ftpd flags
```

设置为允许匿名访问并启用 chroot：

```sh
# rcctl set ftpd flags -A
```

启动守护进程：

```sh
# rcctl restart ftpd
```

#### 常用标志

- `-A`：启用匿名 FTP
- `-D`：不脱离终端（便于调试）
- `-S`：记录所有会话活动
- `-l`：记录所有传输的文件

### 匿名 FTP

启用匿名访问：

1. 创建匿名用户主目录：

   ```sh
   # mkdir -p /home/ftp/pub
   # chown root:wheel /home/ftp
   # chmod 755 /home/ftp
   ```
2. 将文件放置在 `/home/ftp/pub` 下。
3. 使用 `-A` 标志启动 `ftpd`：

   ```sh
   # rcctl set ftpd flags -A
   # rcctl restart ftpd
   ```

用户现在可通过以下方式连接：

```sh
$ ftp ftp://hostname/pub/
```

匿名用户会被 chroot 到 `/home/ftp`。

### 身份验证 FTP

FTP 登录使用系统账户。将 shell 设为 `/sbin/nologin` 可授予无 shell 访问权限：

```sh
# useradd -s /sbin/nologin ftpuser
# passwd ftpuser
```

然后将文件放置在用户主目录下。

### 日志

FTP 日志出现在 `/var/log/messages`。通过 `-S` 标志启用会话日志：

```sh
# rcctl set ftpd flags "-A -S"
# rcctl restart ftpd
```

## FTP 的替代方案

| 协议 | 工具 | 是否加密 | 说明 |
| --- | --- | --- | --- |
| SFTP | `sftp(1)` | 是 | 基于 SSH；首选替代方案 |
| HTTP | `httpd(8)` | 可选 | 便于通过 HTTP 提供文件 |
| SCP | `scp(1)` | 是 | 简单的基于 SSH 的文件复制 |
| rsync | `rsync` | 可选 | 通过软件包提供，效率高 |

仅在需要兼容旧系统或特殊场景时使用 FTP。其他情况优先使用 `sftp` 等加密协议。

## 关键命令一览

| 命令 | 说明 |
| --- | --- |
| `ftp ftp://...` | 通过 FTP 获取文件（客户端） |
| `ftp -o file URL` | 保存为指定文件名 |
| `rcctl enable ftpd` | 启用 FTP 守护进程 |
| `rcctl set ftpd flags ...` | 设置服务器选项 |
| `rcctl start ftpd` | 启动 FTP 守护进程 |
| `rcctl restart ftpd` | 应用新配置 |

## 安全注意事项

FTP 协议本身不安全，凭据和数据均以明文传输。对外或不受信任的网络环境应使用 `sftp` 或其他加密传输方式。

以下场景可考虑使用 FTP：

- 内部或隔离网络
- 公共文件分发（仅匿名）
- 与旧系统的特定自动化任务
