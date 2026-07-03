# rsync

## 概述

`rsync` 是快速而通用的文件复制工具，用于在本地与远程系统之间同步文件和目录。它采用基于增量的文件传输，在备份与大规模数据同步任务中尤为高效。`rsync` 虽未包含在 OpenBSD 基本系统中，但可通过软件包获取，并与 `ssh(1)` 集成实现安全传输。

本章介绍 `rsync` 在 OpenBSD 上的安装、配置与使用，涵盖客户端与服务器两种角色，并提供常见用法示例、安全配置以及独立 `rsyncd` 守护进程的运行方式。

## 安装

从 OpenBSD 软件包集合安装 `rsync`：

```sh
# pkg_add rsync
```

此命令将 `rsync` 二进制文件及相关文件安装到 **/usr/local/bin/rsync**。

## 客户端用法

`rsync` 最常见的用法是作为客户端，在本地或远程系统之间同步文件。它支持多种传输方式，远程操作默认使用 `ssh(1)`。

### 语法

```sh
rsync [options] SOURCE DEST
```

示例如下。

### 本地复制

在本地复制目录：

```sh
$ rsync -av /home/user/documents/ /backup/documents/
```

- `-a`：归档模式（保留权限、符号链接等）
- `-v`：详细输出
- 源路径末尾的斜杠确保复制的是目录内容，而非目录本身

### 通过 SSH 远程复制

通过 SSH 将文件复制到远程主机：

```sh
$ rsync -avz /home/user/data/ user@remote.example.com:/home/user/backup/
```

从远程主机复制：

```sh
$ rsync -avz user@remote.example.com:/var/log/ ./logs/
```

- `-z`：启用压缩
- `-e ssh`：可省略；`rsync` 默认使用 SSH 传输

指定非标准 SSH 端口：

```sh
$ rsync -av -e "ssh -p 2222" ./site/ user@remote.example.com:/var/www/
```

### 排除文件

排除临时文件：

```sh
$ rsync -av --exclude '*.tmp' ./ src/ user@host:/srv/
```

排除一组模式列表：

```sh
$ rsync -av --exclude-from=exclude.txt ./ src/ user@host:/srv/
```

### 删除已移除的文件

同步并删除目标端中源端已不存在的文件：

```sh
$ rsync -av --delete ./syncdir/ user@host:/srv/syncdir/
```

### 试运行

仅展示将执行的操作，不实际执行：

```sh
$ rsync -av --dry-run ./docs/ user@host:/backup/docs/
```

## 运行 rsync 服务器

`rsync` 可作为独立守护进程运行，通过 `rsync://` 协议提供文件。与典型的基于 SSH 的用法不同，此模式不使用 SSH，需要显式配置与激活。

### rsyncd 配置

作为守护进程运行时，`rsync` 从 **/etc/rsyncd.conf** 读取配置，或从通过 `--config` 选项指定的其他文件读取。格式类似 INI 文件，可定义一个或多个模块。

配置示例：

```ini
uid = _rsync
gid = _rsync
use chroot = yes
max connections = 4
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid

[public]
    path = /srv/rsync/public
    comment = Public files
    read only = yes
    list = yes

[backup]
    path = /srv/rsync/private
    comment = Backup files
    auth users = backupuser
    secrets file = /etc/rsyncd.secrets
    read only = no
    list = no
```

此示例定义了两个模块：

- `public`：只读的公共目录。
- `backup`：受密码保护的可写模块，仅限 `backupuser` 访问。

### 认证密钥

若任一模块使用了 `auth users`，须指定密钥文件。该文件将用户名映射到密码：

```
backupuser:password123
```

为密钥文件设置安全权限：

```sh
# chmod 600 /etc/rsyncd.secrets
```

### 启动守护进程

手动启动守护进程以进行测试：

```sh
# /usr/local/bin/rsync --daemon
```

要在开机时启用 `rsyncd`，推荐安装自定义 `rc.d` 脚本，详见下一节。

## rc.d 集成

OpenBSD 默认不为 `rsyncd` 提供 `rc.d` 脚本。要用 `rcctl(8)` 管理守护进程，在 **/etc/rc.d/rsyncd** 创建自定义脚本：

```sh
# vi /etc/rc.d/rsyncd
```

```sh
#!/bin/ksh
# PROVIDE: rsyncd
# REQUIRE: DAEMON
# KEYWORD: shutdown

. /etc/rc.d/rc.subr

rc_reload=NO
rc_cmd="/usr/local/bin/rsync --daemon"

rc_flags=""
rc_pre() {
    checkyesno rsyncd_flags
}

rc_start() {
    $rc_cmd
}

rc_stop() {
    pkill -xf "$rc_cmd"
}

run_rc_command "$1"
```

设置适当权限并启用服务：

```sh
# chmod +x /etc/rc.d/rsyncd
# rcctl enable rsyncd
# rcctl start rsyncd
```

## 连接到 rsync 守护进程

连接到 `rsync` 服务器上的公共模块：

```sh
$ rsync rsync://rsync.example.com/public/
```

带认证同步：

```sh
$ rsync rsync://backupuser@rsync.example.com/backup/ --password-file=/etc/rsync.pass
```

其中 **/etc/rsync.pass** 内容为：

```
password123
```

确保文件权限正确：

```sh
# chmod 600 /etc/rsync.pass
```

## 与替代工具对比

| 工具 | 加密 | 适用场景 | 备注 |
| --- | --- | --- | --- |
| `rsync` | 可选 | 大型文件集 | 增量传输；守护进程或通过 SSH |
| `scp` | 是 | 简单复制 | 始终加密；不支持断点续传 |
| `sftp` | 是 | 交互式使用 | 基于 SSH 的文件传输 |
| `ftp` | 否 | 匿名访问 | 传统方式；安全性较低 |
| `http` | 可选 | 公共下载 | 适合静态文件分发 |

## 安全注意事项

- 使用守护进程模式（`rsync://`）时**不进行加密**。
- 安全传输应始终优先使用基于 SSH 的 `rsync`。
- 以严格权限保护密钥文件（`rsyncd.secrets`、`--password-file`）。
- 暴露 `rsyncd` 时应使用 `chroot`、UID/GID 限制与适当的文件权限。

## 常用命令一览

| 命令 | 说明 |
| --- | --- |
| `pkg_add rsync` | 在 OpenBSD 上安装 `rsync` |
| `rsync -avz src/ user@host:/dest/` | 以归档模式与压缩方式复制 |
| `rsync -av --delete src/ user@host:/dest/` | 同步并删除多余文件 |
| `rsync -e "ssh -p 2222"` | 使用自定义 SSH 端口 |
| `rsync --daemon` | 启动 rsync 服务器守护进程 |
| `rsync rsync://host/module/` | 连接到远程 rsync 守护进程 |
