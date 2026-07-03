# TFTP

## 概述

**TFTP（Trivial File Transfer Protocol）** 是一种基于 UDP 的轻量级文件传输协议，用于网络引导（PXE）、固件加载、简单设备 provisioning 等场景。

OpenBSD 基本系统内置了标准的 `tftpd(8)` 守护进程和 `tftp(1)` 客户端。TFTP 仅支持基本的文件读写，开销极小。它没有身份验证和加密，因此**仅可在受信任或隔离的环境中使用**。

本章介绍 OpenBSD 上 TFTP 的设置与使用，包括与 `inetd(8)` 集成、PXE 引导支持和防火墙配置。

## TFTP 使用场景

- 无盘系统的 PXE 引导
- 网络引导加载程序（如 `pxeboot`、`grub.efi`）
- 向嵌入式设备传输配置文件
- 远程刷写固件（如交换机、路由器）

## 通过 inetd 运行 tftpd

运行 TFTP 服务器最简单、最安全的方式是通过 OpenBSD 的超级服务器 `inetd(8)`。

编辑 `/etc/inetd.conf`，取消注释或添加以下行：

```
tftp    dgram   udp     wait    root    /usr/sbin/tftpd     tftpd -s /tftpboot
```

- `-s /tftpboot` 将文件访问限制在该目录内（`chroot`）
- `wait` 确保 `tftpd` 不会并发启动

创建目录并放入所需文件：

```sh
# mkdir -p /tftpboot
# chown root:wheel /tftpboot
# chmod 755 /tftpboot
```

启用并启动 `inetd`：

```sh
# rcctl enable inetd
# rcctl start inetd
```

编辑后重新加载配置：

```sh
# rcctl reload inetd
```

## 使用 tftp(1) 测试

使用基本系统的 `tftp` 客户端测试：

```sh
$ tftp localhost
tftp> get testfile
tftp> quit
```

文件 `testfile` 必须存在于 `/tftpboot/testfile`，且用户 `nobody`（或 `root`，取决于 `inetd` 配置）可读。

权限示例：

```sh
# cp /bsd.rd /tftpboot/bsd.rd
# chmod 644 /tftpboot/bsd.rd
```

## PXE 引导示例

支持 PXE 客户端：

1. 将引导加载程序文件放入 `/tftpboot`（如 `pxelinux.0`、`grubx64.efi` 或 `bsd.rd`）
2. 确保文件名与 DHCP 服务器配置匹配
3. 在 DHCP 响应中提供匹配的 `next-server` 和 `filename`

配合 OpenBSD `dhcpd(8)` 使用：

```
option domain-name-servers 192.0.2.1;
option routers 192.0.2.1;

next-server 192.0.2.1;
filename "pxelinux.0";
```

PXE 客户端随后会从运行于 `192.0.2.1` 的 TFTP 服务器获取 `pxelinux.0`。

## 防火墙配置

在 `pf.conf` 中放行 TFTP 访问：

```pf
pass in on $int_if proto udp from any to (self) port 69
```

TFTP 使用 **UDP 端口 69** 进行初始控制。数据端口按会话动态协商，均为临时端口。

在严格控制的环境中，可仅放行初始端口，并依赖可信网络。

## 日志

`tftpd` 默认不记录传输。要启用日志，在 `/etc/inetd.conf` 中使用 `-v`（详细）或 `-l`（记录所有请求）选项：

```
tftp    dgram   udp     wait    root    /usr/sbin/tftpd     tftpd -lv -s /tftpboot
```

请求将出现在 `/var/log/messages` 或通过 `syslogd` 输出。

示例：

```sh
# tail -f /var/log/messages
```

## 安全注意事项

TFTP 没有访问控制、身份验证或加密。为降低风险：

- 始终使用 `-s` 运行 `tftpd`，限制访问范围
- 尽可能使用只读文件
- 不要将 TFTP 暴露到不受信任的网络
- 使用防火墙按子网限制访问
- 除非确有必要，否则避免启用写访问（`-w`）

允许写访问：

```
tftp    dgram   udp     wait    root    /usr/sbin/tftpd     tftpd -sw /tftpboot
```

随后相应调整文件与目录权限。

## TFTP 客户端（tftp(1)）

OpenBSD 内置命令行客户端 `tftp(1)`：

```sh
$ tftp remote.host
tftp> get firmware.img
tftp> put config.txt
tftp> quit
```

使用 `-v` 获取详细输出，便于排错。

对于自动化传输（如 shell 脚本中），可使用 `ftp(1)` 或支持 TFTP 的外部工具。

## 替代方案与增强

- 可能时优先使用 `ftp(1)` 配合 HTTP/FTP，而非 TFTP
- 将 TFTP 与 HTTP 引导结合（iPXE 或 UEFI HTTP 引导）
- 使用 `relayd(8)` 进一步限制访问
- 对于需要身份验证的传输，可考虑 `scp` 或 `rsync` 等安全替代方案
