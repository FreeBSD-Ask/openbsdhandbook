# FreeBSD 用户视角的 OpenBSD

本快速入门面向 FreeBSD 管理员，通过将熟悉的概念映射到 OpenBSD 工具与惯例来介绍 OpenBSD。文中突出实际差异，并非详尽对比，也不涉及理念讨论。本指南假设 OpenBSD 7.8 已安装完毕，且你已有命令行访问权限。

## Shell

OpenBSD 上 root 与普通用户的默认 shell 都是 Korn shell [ksh(1)](https://man.openbsdhandbook.com/ksh.1/)
。这与 FreeBSD 不同——FreeBSD 的 root 账号默认使用 `tcsh`。OpenBSD 的 [ksh(1)](https://man.openbsdhandbook.com/ksh.1/)
实现了传统 Bourne shell 语言的一个超集。

其他 shell 以软件包形式提供，参见 [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
与 [chsh(1)](https://man.openbsdhandbook.com/chsh.1/)
。

**建议：** 不要把 root 的 shell 改成由软件包提供的 shell。非基本系统的 shell 位于 `/usr/local/bin`，在受限恢复场景下可能不可用。基本系统的 [ksh(1)](https://man.openbsdhandbook.com/ksh.1/)
位于 `/bin`。

```sh
# 以 root 身份安装常用替代 shell
# pkg_add bash zsh

$ doas pkg_add bash zsh
$ chsh -s /usr/local/bin/zsh
```

## 提权：doas

OpenBSD 基本系统自带 [doas(1)](https://man.openbsdhandbook.com/doas.1/)
用于提权。通过 [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/)
配置。`/etc/examples/` 下有已说明的示例。

```sh
$ doas cp /etc/examples/doas.conf /etc/doas.conf
  # 从示例配置开始
```

默认情况下，每次调用都会要求输入密码。要像某些 `sudo` 配置那样缓存认证，加入 `persist`：

```
permit persist keepenv :wheel
```

如确有需要，可安装 `sudo` 软件包，参见 sudo(8)。

## 软件管理

OpenBSD 把基本系统与软件包分开。优先使用预编译软件包，参见 [packages(7)](https://man.openbsdhandbook.com/packages.7/)
。从 ports 构建见 [ports(7)](https://man.openbsdhandbook.com/ports.7/)
。

### 安装与删除软件包

用 [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
安装预编译软件包：

```sh
$ doas pkg_add nginx
```

用 [pkg\_delete(1)](https://man.openbsdhandbook.com/pkg_delete.1/)
删除软件包：

```sh
$ doas pkg_delete nginx
```

用 [pkg\_info(1)](https://man.openbsdhandbook.com/pkg_info.1/)
列出已安装软件包：

```sh
$ pkg_info
```

### 更新（同一发行版内）

OpenBSD 通过 [syspatch(8)](https://man.openbsdhandbook.com/syspatch.8/)
为**基本系统**提供二进制补丁。

```sh
$ doas syspatch -c
  # 查看可用的基系统补丁
$ doas syspatch
  # 应用基系统补丁；按需重启
```

将已安装软件包升级到当前发行版的最新版本：

```sh
$ doas pkg_add -Uu
  # 在当前发行版内升级全部软件包
```

### 升级（到新发行版）

用 [sysupgrade(8)](https://man.openbsdhandbook.com/sysupgrade.8/)
拉取并执行发行版升级。重启后若收到提示，用 [sysmerge(8)](https://man.openbsdhandbook.com/sysmerge.8/)
合并配置改动，再更新软件包。

```sh
$ doas sysupgrade
  # 将基本系统升级到下一发行版

$ doas pkg_add -Uu
  # 基本系统升级后更新软件包
```

## 网络

两个系统的接口**命名**都基于驱动，因此 `em0`、`re0` 等名称并不陌生。**配置模型**则有所不同。

### 用 hostname.if 配置接口

每接口配置存放在 [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
中，`if` 为接口名。例如 `/etc/hostname.em0`：

静态 IPv4：

```
inet 10.0.0.100 255.255.255.0
```

静态 IPv6：

```
inet6 2001:db8:6000:9344::154 64
```

DHCP：

```
dhcp
```

临时性的运行时改动可用 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
完成：

```sh
$ doas ifconfig em0 10.0.0.100 255.255.255.0
```

从文件应用配置时，使用 [netstart(8)](https://man.openbsdhandbook.com/netstart.8/)
：

```sh
$ doas sh /etc/netstart
  # 重新载入全部接口
$ doas sh /etc/netstart em0
  # 重新载入单个接口
```

### 主机名

在 [myname(5)](https://man.openbsdhandbook.com/myname.5/)
（`/etc/myname`）中设置系统的全限定域名。该名称必须能通过 `/etc/hosts` 或 DNS 解析。

```
host.example.com
```

修改后用 `sh /etc/netstart` 重新载入网络。

### 默认网关

在 [mygate(5)](https://man.openbsdhandbook.com/mygate.5/)
（`/etc/mygate`）中设置默认网关。每行一个地址；每个协议族取第一条。

```
192.0.2.1
2001:db8:6000:9344::1
```

修改后用 `sh /etc/netstart` 重新载入网络。

### DNS 解析器

在 [resolv.conf(5)](https://man.openbsdhandbook.com/resolv.conf.5/)
中配置解析器：

```
nameserver 192.0.2.1
lookup file bind
```

修改后用 `sh /etc/netstart` 重新载入网络。

## 守护进程与启动

OpenBSD 采用传统 BSD init 与 rc 系统，参见 [init(8)](https://man.openbsdhandbook.com/init.8/)
、[rc(8)](https://man.openbsdhandbook.com/rc.8/)
与 [rc.conf(8)](https://man.openbsdhandbook.com/rc.conf.8/)
。系统默认值在 `/etc/rc.conf`。不要直接编辑该文件，应在 `/etc/rc.conf.local` 中覆盖并本地化设置。

FreeBSD 管理员常用 `service(8)` 与 `sysrc(8)`。在 OpenBSD 上，用 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
控制并启用守护进程。

要让基本系统的 Web 服务器 [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
开机自启，可用 `rcctl enable httpd`，或在 `rc.conf.local` 中写入空 flags 行：

```conf
httpd_flags=
```

用 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
控制守护进程：

```sh
$ doas rcctl start httpd # 立即启动 httpd 服务
$ doas rcctl stop httpd # 停止运行中的 httpd 服务
$ doas rcctl reload httpd # 重新载入 httpd 配置，无需完整重启
$ doas rcctl enable httpd # 让 httpd 开机自启
$ doas rcctl disable httpd # 取消开机自启
```

## 常用对照

| 任务（FreeBSD） | OpenBSD 工具或文件 | 用途 |
| --- | --- | --- |
| `pkg install nginx` | `pkg_add nginx` | 从仓库安装软件包 |
| `pkg delete nginx` | `pkg_delete nginx` | 删除软件包 |
| `pkg upgrade` | `pkg_add -Uu` | 在当前发行版内升级全部软件包 |
| `freebsd-update fetch install` | `syspatch` | 应用基本系统补丁 |
| `freebsd-update -r 14.1-RELEASE upgrade` | `sysupgrade` | 升级到下一 OpenBSD 发行版 |
| `service sshd start` | `rcctl start sshd` | 启动服务 |
| `sysrc sshd_enable=YES` | `rcctl enable sshd` | 让服务开机自启 |
| `sysrc ifconfig_em0="DHCP"` | `/etc/hostname.em0` 写入 `dhcp` | 将接口配置为 DHCP |
| `service netif restart` | `sh /etc/netstart [if]` | 重新载入网络配置 |
| `/etc/rc.conf` 中的 `hostname="host.example.com"` | `/etc/myname` | 设置系统主机名 |
| `/etc/rc.conf` 中的 `defaultrouter="192.0.2.1"` | `/etc/mygate` | 设置默认网关 |

更多详情请参阅本站引用的手册页。
