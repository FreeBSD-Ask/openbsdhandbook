# YP (NIS)

## 概述

**YP（Yellow Pages）**，又称 **NIS（Network Information Service）**，是一种简单协议，用于在受信任的本地网络内共享用户账号、组成员关系、主机名等配置数据库。OpenBSD 基本系统同时支持 YP 服务器与客户端。

YP **不同于 LDAP**，二者需要区分：

- YP 是较老的、基于 RPC 的系统，由 Sun Microsystems 开发。
- LDAP（Lightweight Directory Access Protocol）是现代化的可扩展协议，由 RFC 4511 标准化。
- YP 默认包含在 OpenBSD 中；LDAP 支持需要安装 `openldap-server` 及相关软件包。

YP 最适合小型受信任网络，这类环境重视简洁胜过灵活性与加密安全。LDAP 提供更丰富的 schema 模型、加密与细粒度访问控制，在较大或对安全更敏感的环境中更为合适。

本章说明如何将 OpenBSD 配置为 YP 主服务器、YP 从服务器或 YP 客户端。

## YP 主服务器配置

YP 主服务器维护所有 YP 映射（数据库）的权威副本。这些映射根据 **/etc** 中的文件生成，并分发给客户端与从服务器。

所需的守护进程：

- `portmap(8)`：RPC 端口映射器
- `ypserv(8)`：YP 服务守护进程
- `ypbind(8)`：主服务器上也需要，用于作为自身的客户端
- `rpc.ypxfrd(8)`：可选的映射传输加速器（从服务器使用）

### 启用所需服务

启用并启动必要的服务：

```sh
# rcctl enable portmap ypserv ypbind
# rcctl start portmap
# rcctl start ypserv
# rcctl start ypbind
```

YP 主服务器还必须能够生成并提供 YP 映射。使用 `ypinit(8)` 并加 `-m` 标志：

```sh
# ypinit -m
```

该脚本会提示输入 YP 域名与从服务器列表。若没有从服务器，按回车继续。

YP 域名（不要与 DNS 域名混淆）还必须在启动时设置。这通过 `domainname(1)` 与 **/etc/defaultdomain** 完成：

```sh
# echo "exampledomain" > /etc/defaultdomain
```

立即生效：

```sh
# domainname exampledomain
```

### 映射的创建与重新生成

YP 映射由标准系统文件通过 Makefile 构建。源文件取自 **/var/yp/src**，通常是指向 **/etc** 的符号链接。

重新生成所有映射：

```sh
# cd /var/yp
# make
```

Makefile 处理 `passwd.byname`、`group.byname`、`hosts.byname` 等标准映射。

重新生成单个映射（如 passwd）：

```sh
# cd /var/yp
# make passwd
```

## YP 从服务器配置

从服务器通过 `ypxfr(8)` 从主服务器接收映射更新。

初始化从服务器：

```sh
# ypinit -s yp-master-hostname
```

从服务器必须属于同一 YP 域，并且 **/etc/defaultdomain** 要相应设置。

启用并启动所需守护进程：

```sh
# rcctl enable portmap ypserv ypbind
# rcctl start portmap
# rcctl start ypserv
# rcctl start ypbind
```

主服务器必须在 **/var/yp/ypservers** 中允许该从服务器。此文件应列出所有允许拉取映射更新的从服务器主机名。

立即获取映射：

```sh
# /usr/libexec/ypxfr passwd.byname
```

## YP 客户端配置

客户端使用 `ypbind(8)` 绑定到正在运行的 YP 服务器并请求映射。

### 设置域名

确保 **/etc/defaultdomain** 包含正确的 YP 域：

```sh
# echo "exampledomain" > /etc/defaultdomain
```

立即生效：

```sh
# domainname exampledomain
```

### 启用客户端

启用并启动 `portmap` 与 `ypbind`：

```sh
# rcctl enable portmap ypbind
# rcctl start portmap
# rcctl start ypbind
```

### 验证客户端运行

使用 `ypwhich` 确认正在使用哪台服务器：

```sh
$ ypwhich
yp-master.example.net
```

列出可用映射：

```sh
$ ypcat -x
```

转储特定映射：

```sh
$ ypcat passwd.byname
$ ypmatch root passwd.byname
```

### 在 /etc/nsswitch.conf 中启用 YP 查询

`nsswitch.conf(5)` 文件决定系统查询使用哪些来源。

要让用户、组与主机的查询使用 YP，修改相关行：

```
passwd:     files yp
group:      files yp
netid:      files yp
hosts:      files dns
```

此配置使系统先检查本地文件，再查询 YP。

### 使用 YP 账号登录

来自 YP 映射的用户可正常登录。需确保他们有主目录与合适的 shell。若主目录通过 NFS 导出或按需创建，则须单独处理。

列出可用用户：

```sh
$ ypcat passwd.byname | cut -d: -f1
```

## 安全注意事项

YP 以明文传输所有数据，包括密码哈希。因此：

- 使用 `passwd.adjunct.byname` 隐藏加密密码（可选）。
- 用 `pf(4)` 与网络隔离限制对 YP 服务的访问。
- 不要在不受信任或公共网络上运行 YP。
- 除非必要，否则避免在 **/etc/passwd** 中使用 `+::::::`。优先使用 `nsswitch.conf`。

在受信任的局域网中，若用 SSH 进行访问控制，并有物理或 VLAN 隔离，YP 仍然实用。

## 关键文件与工具

| 路径 | 用途 |
| --- | --- |
| **/etc/defaultdomain** | 存储 YP 域名 |
| **/var/yp** | 包含 YP 映射 |
| **/var/yp/ypservers** | 主服务器上的从服务器列表 |
| **/etc/nsswitch.conf** | 控制名称查询顺序 |

| 命令 | 说明 |
| --- | --- |
| `ypinit -m` | 初始化 YP 主服务器 |
| `ypinit -s` | 初始化 YP 从服务器 |
| `ypcat` | 查看映射内容 |
| `ypmatch` | 按键查询映射 |
| `ypwhich` | 显示已绑定的 YP 服务器 |
| `make`（在 **/var/yp** 中） | 重建 YP 映射 |
