# Samba

## 概述

**Samba** 是一组程序，实现了 SMB（Server Message Block）协议及其现代扩展 CIFS（Common Internet File System）。它让 Unix 系统能与 Windows 客户端共享文件和打印机，反之亦然。Samba 可作为独立文件服务器、Windows 域成员或 Active Directory 域控制器运行（后者在 OpenBSD 上较少见）。

OpenBSD 基本系统不包含 Samba。可通过 ports 与软件包系统获取。

## 安装

安装 Samba：

```sh
# pkg_add samba
```

此命令会安装 Samba 服务器二进制文件、配置文件与文档。该软件包包含用于文件与打印机共享的 `smbd(8)` 以及用于 NetBIOS 名称解析与浏览的 `nmbd(8)`。基本的服务器用途通常只需 `smbd`。

## 配置

Samba 通过 **/etc/samba/smb.conf** 配置。启动守护进程前必须创建此文件。导出公共共享的最小化配置如下：

### 配置

```ini
[global]
   workgroup = WORKGROUP
   server string = OpenBSD Samba Server
   security = user
   log file = /var/log/samba/log.%m
   max log size = 1000
   passdb backend = tdbsam
   load printers = no
   printing = bsd
   disable spoolss = yes

[public]
   path = /srv/smb/public
   public = yes
   writable = yes
   printable = no
   guest ok = yes
```

此配置定义了位于 **/srv/smb/public** 的共享目录，允许匿名用户写入。在更安全的环境中，应使用 `guest ok = no` 并通过用户认证控制访问。

Samba 使用自己的用户数据库。系统用户必须通过以下命令添加到其中：

```sh
# smbpasswd -a username
```

确保共享目录存在并具有合适权限：

```sh
# mkdir -p /srv/smb/public
# chown -R nobody:nogroup /srv/smb/public
# chmod 0777 /srv/smb/public
```

这些权限允许访客用户写入。根据认证共享的需要作适当调整。

## 服务管理

Samba 开箱即用，不直接集成 OpenBSD 的 `rc.d(8)` 框架。服务可通过 `rc.local` 或自定义 `rc.d` 脚本启动。

手动启动服务：

```sh
# /usr/local/sbin/smbd
# /usr/local/sbin/nmbd
```

要在启动时自动运行，追加到 **/etc/rc.local**：

```sh
if [ -x /usr/local/sbin/smbd ]; then
    echo -n ' smbd'; /usr/local/sbin/smbd
fi
if [ -x /usr/local/sbin/nmbd ]; then
    echo -n ' nmbd'; /usr/local/sbin/nmbd
fi
```

或者创建自定义 `rc.d` 脚本以集成 `rcctl(8)`。

## 客户端访问

在 Unix 客户端上，可使用 `samba-client` 软件包中的 `smbclient` 测试对 Samba 共享的访问：

```sh
$ pkg_add samba-client
$ smbclient //hostname/public -U username
```

在 Windows 上，打开文件资源管理器并输入：

```
\\hostname\public
```

将 `hostname` 替换为 Samba 服务器的 IP 地址或 NetBIOS 名称。

除非配置了访客访问，否则会出现认证提示。

## 打印机共享

打印机支持有限，通常需要与 `cupsd` 或 `lpd` 等打印后台程序集成。对于基本的文件服务器功能，通常会完全禁用打印机支持：

```conf
load printers = no
disable spoolss = yes
```

这能减小攻击面，并避免客户端环境中出现误导性的打印机列表。

## 防火墙注意事项

SMB 协议使用多个端口：

- TCP 139：NetBIOS 会话服务
- TCP 445：直接 SMB over TCP
- UDP 137：NetBIOS 名称服务
- UDP 138：NetBIOS 数据报服务

若使用 `pf(4)`，必须编写 pass 规则以允许来自客户端主机的流量访问这些端口：

```pf
pass in on $int_if proto tcp from any to (self) port { 139, 445 }
pass in on $int_if proto udp from any to (self) port { 137, 138 }
```

切勿将这些服务暴露到互联网。

## 安全注意事项

Samba 是庞大而复杂的守护进程，历史上出现过若干漏洞。强烈建议：

- 将访问限制在受信任的网络
- 禁用不必要的功能（如打印、访客访问）
- 定期更新软件包
- 用防火墙限制访问
- 避免在面向互联网的接口上运行 Samba

对于需要强访问控制并与集中认证集成的生产环境，考虑仅以独立模式运行 Samba，或根据客户端需求使用 `rsync` 或 `NFS` 等替代方案。

## 与 NFS 的差异

| 特性 | Samba | NFS |
| --- | --- | --- |
| 协议 | SMB/CIFS | NFSv3 |
| Windows 支持 | 原生 | 有限 |
| 认证 | 用户名/密码（tdbsam） | 基于 UID，`maproot` |
| 集成 | Windows 环境 | 类 Unix 系统 |
| 可移植性 | 与 Windows 配合良好 | 与 Unix/Linux 配合良好 |
| 所需软件包 | 不在基本系统中 | 内置于 OpenBSD |

Samba 适合有 Windows 客户端的异构环境。NFS 更适合纯 Unix 网络。
