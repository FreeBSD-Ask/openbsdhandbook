# Linux 用户视角的 OpenBSD

本快速入门面向 Linux 管理员，通过将熟悉的概念映射到 OpenBSD 工具与惯例来介绍 OpenBSD。文中突出实际差异，并非详尽对比，也不涉及理念讨论。本指南假设 OpenBSD 已安装完毕，且你已有命令行访问权限。安装步骤请参阅本站安装章节。

## Shell

root 与普通用户的默认 shell 都是 Korn shell [ksh(1)](https://man.openbsdhandbook.com/ksh.1/)
。其命令语言是传统 Bourne shell 的超集。Bash、Zsh 等 shell 以软件包形式提供，参见 [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
与 [chsh(1)](https://man.openbsdhandbook.com/chsh.1/)
。

**建议：** 不要把 root 的 shell 改成由软件包提供的 shell。非基本系统的 shell 位于 `/usr/local/bin`，在受限恢复场景下可能不可用。基本系统的 [ksh(1)](https://man.openbsdhandbook.com/ksh.1/)
位于 `/bin`。

```sh
# 以 root 身份安装常用替代 shell
# 普通账号请使用 doas(1)；见下一节
# pkg_add bash zsh

$ doas pkg_add bash zsh
$ chsh -s /usr/local/bin/bash
```

## 提权：doas（取代 sudo）

OpenBSD 提供 [doas(1)](https://man.openbsdhandbook.com/doas.1/)
用于提权。通过 [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/)
配置。`/etc/examples/` 下有示例文件。

```sh
$ doas cp /etc/examples/doas.conf /etc/doas.conf
  # 从已说明的示例开始
```

默认情况下，每次调用都会要求输入密码。要像许多 `sudo` 配置那样缓存认证，使用 `persist` 选项：

```
permit persist keepenv :wheel
```

如确有需要，可安装 `sudo` 软件包，参见 sudo(8)。多数工作流中 `doas` 即可胜任同等角色，配置也更简单。

## 软件管理

OpenBSD 把基本系统与软件包分开。包管理工具见 [packages(7)](https://man.openbsdhandbook.com/packages.7/)
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
为**基本系统**提供二进制补丁。请定期使用：

```sh
$ doas syspatch -c
  # 查看可用的基系统补丁
$ doas syspatch
  # 应用基系统补丁，按需重启
```

软件包在同一发行版内独立更新。将已安装软件包升级到当前发行版的最新版本：

```sh
$ doas pkg_add -Uu
  # 在当前发行版内升级全部软件包
```

### 升级（到新发行版）

OpenBSD 大约每年发布两个发行版。用 [sysupgrade(8)](https://man.openbsdhandbook.com/sysupgrade.8/)
拉取并执行发行版升级：

```sh
$ doas sysupgrade
```

基本系统升级并重启后，更新软件包：

```sh
$ doas pkg_add -Uu
```

## 网络

OpenBSD 按驱动名命名网络接口，而非 `eth0`、`enp0s1` 之类。例如 `re0`（Realtek）、`bge0`（Broadcom）、`em0`（Intel）。详见 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
。

### 用 hostname.if 配置接口

每接口配置存放在 [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
中，`if` 为接口名。例如 `/etc/hostname.re0`：

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
$ doas ifconfig re0 10.0.0.100 255.255.255.0
```

从文件应用配置时，使用开机时用到的同一脚本 [netstart(8)](https://man.openbsdhandbook.com/netstart.8/)
：

```sh
$ doas sh /etc/netstart
  # 重新载入全部接口
$ doas sh /etc/netstart re0
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
。没有 SysV 风格的运行级别。

系统默认值在 `/etc/rc.conf`。不要直接编辑该文件，应在 `/etc/rc.conf.local` 中覆盖并本地化设置。

要让基本系统的 Web 服务器 [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
开机自启，可用 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
，或在 `rc.conf.local` 中写入空 flags 行：

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

## 常用命令对照

| Linux 命令（RPM/DPKG） | OpenBSD 工具 | 用途 |
| --- | --- | --- |
| `yum install` / `apt-get install pkg` | `pkg_add pkg` | 从仓库安装软件包 |
| `rpm -i pkg.rpm` / `dpkg -i pkg.deb` | `pkg_add pkg.tgz` | 安装本地软件包 |
| `rpm -qa` / `dpkg -l` | `pkg_info` | 列出已安装软件包 |
| `lspci` | `pcidump` | 列出 PCI 设备 |
| `lsusb` | `usbdevs` | 列出 USB 设备 |

详情参见 [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
、[pkg\_info(1)](https://man.openbsdhandbook.com/pkg_info.1/)
、[pcidump(8)](https://man.openbsdhandbook.com/pcidump.8/)
与 [usbdevs(8)](https://man.openbsdhandbook.com/usbdevs.8/)
。
