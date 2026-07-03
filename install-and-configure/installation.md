# 安装 OpenBSD

## 概述

本章详细介绍 OpenBSD 操作系统的安装。内容涵盖获取与准备安装介质、安装前任务、运行安装程序、安装后配置，以及无人值守与无状态部署等高级安装选项。

## 获取安装介质

OpenBSD 官方安装镜像通过镜像站网络分发。镜像主列表维护于：

- <https://www.openbsd.org/ftp.html>

选择地理位置相近的镜像以获得最佳速度。当前版本的安装文件集位于：

```text
/pub/OpenBSD/7.8/ARCH/
```

将 `ARCH` 替换为硬件架构（例如 `amd64`、`arm64`、`i386`）。

例如，OpenBSD 7.8 的 amd64 目录为：

```
https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/
```

### 安装镜像

常见安装镜像包括：

- `install78.img` - USB 安装镜像（推荐多数用户使用）。
- `install78.iso` - 用于 CD/DVD 介质的 ISO 镜像。
- `miniroot78.img` - 适用于自定义或网络安装的小型 USB/PXE 镜像。

### 通过命令行下载

在 OpenBSD 系统上，使用 [ftp(1)](https://man.openbsdhandbook.com/ftp.1/)：

```sh
$ ftp https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/install78.img
$ ftp https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/SHA256.sig
```

在 Linux 或 macOS 上，可使用 curl(1) 或 wget(1)：

```sh
$ curl -O https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/install78.img
$ wget https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/SHA256.sig
```

务必同时获取对应的 `SHA256.sig` 文件，其中包含所有发布文件的签名。

### 校验签名

校验可确保镜像未被篡改。使用 [signify(1)](https://man.openbsdhandbook.com/signify.1/) 配合发布公钥（OpenBSD 系统已安装在 **/etc/signify/**）：

```sh
$ signify -C -p /etc/signify/openbsd-78-base.pub -x SHA256.sig install78.img
```

在非 OpenBSD 系统上，从 <https://ftp.openbsd.org/pub/OpenBSD/> 对应版本目录下下载正确的公钥（例如 `openbsd-78-base.pub`）再行校验。

校验成功会输出 `OK`。若失败，请勿使用该镜像。

## 将安装镜像写入 USB

下载得到的 `.img` 文件必须以**原始**方式写入 USB 闪存盘。写错磁盘会毁掉现有数据，操作前请仔细确认目标设备。

### 识别目标磁盘

**在 OpenBSD 上**，插入 USB 闪存盘后立即运行 [dmesg(8)](https://man.openbsdhandbook.com/dmesg.8/)：

```sh
$ dmesg | tail
sd6 at scsibus3 targ 1 lun 0: <Generic, Flash Disk, 8.07> removable
sd6: 30528MB, 512 bytes/sector, 62537728 sectors
```

此处显示设备为 `sd6`。使用 [dd(1)](https://man.openbsdhandbook.com/dd.1/) 写入时务必使用**原始设备**（`rsd6c`）。

**在 Linux 上**，用 lsblk(8) 查看：

```sh
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0 477G  0 disk
├─sda1   8:1    0 512M  0 part /boot
├─sda2   8:2    0 476G  0 part /
sdb      8:16   1 29.3G  0 disk
└─sdb1   8:17   1 29.3G  0 part /media/usb
```

此处 USB 闪存盘为 **/dev/sdb**。写入前先卸载所有分区。

**在 macOS 上**，使用 diskutil(8)：

```sh
$ diskutil list
/dev/disk2 (external, physical):
   #:                       TYPE NAME           SIZE       IDENTIFIER
   0:     FDisk_partition_scheme             *31.9 GB    disk2
   1:                 DOS_FAT_32 UNTITLED     31.9 GB    disk2s1
```

目标设备为 **/dev/disk2**。原始写入用 **/dev/rdisk2**。

---

### 在 OpenBSD 上写入

```sh
# dd if=install78.img of=/dev/rsd6c bs=1m
```

### 在 Linux 上写入

```sh
$ sudo dd if=install78.img of=/dev/sdX bs=1M status=progress conv=fsync
```

将 **/dev/sdX** 替换为目标 USB 磁盘（例如 **/dev/sdb**）。

### 在 macOS 上写入

```sh
$ diskutil unmountDisk /dev/disk2
$ sudo dd if=install78.img of=/dev/rdisk2 bs=1m
$ sync
```

### 图形化工具

若偏好图形工具，下列应用可直接写入 `.img` 文件：

- [balenaEtcher](https://www.balena.io/etcher/)
- [Fedora Media Writer](https://docs.fedoraproject.org/en-US/fedora/latest/preparing-boot-media/)
- [Rufus](https://rufus.ie)
  （仅限 Windows）

## 安装前任务

### 最低要求

绝对最低配置为 512 MB 内存与 1 GB 磁盘空间，但实际可用的系统至少需要 2 GB 内存与 8 GB 磁盘。强烈建议使用更大容量的磁盘。

### 备份与准备

若目标系统存有重要数据，或要用于多系统引导，请先做完整备份。备份应存放于外部介质。

### 待收集信息

- 主机名
- 时区
- root 密码
- 用户账户信息
- 静态 IP 配置（若不使用 DHCP）
- 磁盘布局与加密选择

### 固件设置

- 在 UEFI 中禁用 Secure Boot
- 视硬件启用 UEFI 或 BIOS 引导
- 将 USB 或 CD/DVD 驱动器设为第一启动设备

## 运行安装程序

从准备好的 USB 闪存盘启动系统。安装程序提示如下：

```
(I)nstall, (U)pgrade, (A)utoinstall or (S)hell?
```

选择 `I` 开始全新安装。

### 安装程序提示（带注释）

#### 终端与系统设置

- **Terminal type?**
  默认 `vt220` 适用于大多数系统。

#### 网络

- **System hostname?**
  示例：`myrouter`
- **Which network interface to configure?**
  选择接口或输入 `done`。
- **IPv4 address?**
  `dhcp`、`none` 或静态地址。
- **IPv6 address?**
  `autoconf`、`none` 或静态地址。
- **Default IPv4 route?**
  仅静态 IP 时填写。示例：**192.168.1.1**。
- **DNS domain name? / DNS nameservers?**
  通常由 DHCP 提供。

#### 用户与访问

- **Password for root?**
  输入时不显示。
- **Start [sshd(8)](https://man.openbsdhandbook.com/sshd.8/)
  ?**
  需要远程登录时选 `yes`。
- **Setup a user?**
  推荐。输入小写用户名。
- **Allow root ssh login?**
  出于安全考虑，选 `no` 或 `prohibit-password`。

#### 时区

- **What timezone are you in?**
  输入 `?` 列出选项。

#### 磁盘设置

- **Which disk is the root disk?**
  示例：`sd0`。
- **Partitioning scheme?**
  UEFI 系统推荐 GPT。
- **Layout? (A)uto / (E)dit / (C)ustom**
  Auto 会创建标准分区（`/`、**/var**、**/tmp**、**/home**）。

#### 加密

若选择加密，[bioctl(8)](https://man.openbsdhandbook.com/bioctl.8/) 会设置全盘加密并提示输入口令。

#### 文件集

- **Location of sets?**
  通常为 `http`。
- **Mirror and path?**
  示例：

  ```
  cdn.openbsd.org
  pub/OpenBSD/7.8/amd64/
  ```
- **Select/deselect sets**
  默认即可。用 `-game*` 跳过游戏。

结束时显示：

```
CONGRATULATIONS! Your OpenBSD install has been successfully completed!
Exit to (S)hell, (H)alt or (R)eboot? [reboot]
```

## 安装后配置

系统重启后，执行以下步骤：

### 登录

以 root 或创建的用户登录。需要提权时使用 [doas(1)](https://man.openbsdhandbook.com/doas.1/)：

```sh
$ doas -s
```

### 应用二进制补丁

运行 [syspatch(8)](https://man.openbsdhandbook.com/syspatch.8/)：

```sh
# syspatch
```

### 软件包更新

更新已安装的软件包：

```sh
# pkg_add -Uu
```

确保 **/etc/installurl** 指向某个镜像：

```sh
# echo "https://cdn.openbsd.org/pub/OpenBSD" > /etc/installurl
```

### 合并配置文件

更新后运行 [sysmerge(8)](https://man.openbsdhandbook.com/sysmerge.8/)：

```sh
# sysmerge
```

### 配置时区

运行 tzsetup(8)：

```sh
# tzsetup
```

### 启用服务

使用 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)：

```sh
# rcctl enable ntpd
# rcctl start ntpd
# rcctl ls on
```

### 配置 doas(1)

编辑 **/etc/doas.conf**：

```
permit persist keepenv :wheel
```

收紧权限：

```sh
# chmod 600 /etc/doas.conf
```

### 查阅系统日志

```sh
# tail -n 60 /var/log/messages
# sysctl hw.sensors
# ifconfig -A
```

### 配置远程访问

确保已启用 [sshd(8)](https://man.openbsdhandbook.com/sshd.8/)，并复制密钥：

```sh
$ mkdir -m 700 ~/.ssh
$ chmod 600 ~/.ssh/authorized_keys
```

### 验证磁盘加密

若启用加密，系统启动时会提示输入口令。交换分区加密是自动的。

```sh
# bioctl softraid0
```

## UEFI 引导说明

OpenBSD 将引导程序安装于：

```text
/EFI/BOOT/BOOTX64.EFI
```

多数固件会自动识别。Secure Boot 必须禁用。

## 用 `siteXX.tgz` 自定义安装

OpenBSD 支持额外的文件集 `site78.tgz`，会在**所有基本系统文件集之后**解包。管理员借此可以干净、受支持的方式，向新安装的系统注入自定义配置、脚本与文件。

### 何时应用

- 交互式安装时：在文件集安装完成后自动应用。
- 无人值守安装时：始终应用，不提示。
- 无状态引导时：若存在，则解包到内存。

安装程序会自动查找通用归档：

- `site78.tgz`

以及主机专属变体：

- `site78-HOSTNAME.tgz`

### 典型内容

示例：

```
etc/rc.conf.local
etc/pf.conf
etc/hostname.em0
root/.ssh/authorized_keys
install.site
usr/local/bin/custom-script
```

### 创建示例

```sh
# tar -C /path/to/custom/root -czphf site78.tgz .
```

确保保留文件模式与属主。所含的 `install.site` 脚本（若有）会在安装结束时执行。

### `install.site` 示例

```sh
#!/bin/sh
echo "Provisioning $(hostname)" >> /var/log/install.log
pkg_add rsync htop
rcctl enable sshd
```

借此可实现安装后自动化。

### 示例用例

#### 网络配置

```
etc/hostname.em0
```
```
inet 192.168.1.10 255.255.255.0
up
```

#### 防火墙规则

```
etc/pf.conf
```
```pf
set block-policy drop
block all
pass in on egress proto tcp to port ssh
```

#### 启用服务

```
etc/rc.conf.local
```
```conf
sshd_flags=
ntpd_flags=
smtpq=YES
```

#### 添加用户 SSH 密钥

```
root/.ssh/authorized_keys
```
```
ssh-ed25519 AAAA... root@admin
```

#### 预配置软件包镜像

```
etc/installurl
```
```
https://cdn.openbsd.org/pub/OpenBSD
```

## 无人值守安装

OpenBSD 可借助配置文件与可选的自定义文件集自动完成安装。

### 工作原理

1. 以 `auto_install` 启动安装程序（在引导提示符处按 `a`）。
2. 安装程序获取名为 `install.conf` 的配置文件。
   - 从 USB 闪存盘（FAT32）获取。
   - 从 DHCP 选项指定的 HTTP 服务器获取。
3. 可选地应用 `siteXX.tgz` 归档并运行 `install.site`。

### `install.conf` 示例

```conf
System hostname = server1
Password for root = *************
Setup a user = alice
Public ssh key for user = ssh-ed25519 AAAA... alice@laptop
Location of sets = http
HTTP Server = cdn.openbsd.org
Set name(s) = -game* +xbase +xshare
```

### 部署方式

- **PXE 引导 + DHCP**：自动分发 `bsd.rd` 与 `install.conf`。
- **USB 闪存盘**：将 `install.conf` 与 `siteXX.tgz` 放在闪存盘根目录。

借此可对大量机器做全自动配置，配置可统一或按主机区分。

## 无状态部署

无状态部署中，OpenBSD 不安装到持久存储，而是用 `bsd.rd` 与可选的 `siteXX.tgz` 引导进内存。所有改动在重启时消失，除非显式写到别处。

### 特点

- 所有文件系统（`/`、**/var**、**/tmp**）都驻留在内存。
- 从 PXE、USB 或 CD-ROM 引导。
- 适合信息亭、设备、临时节点。

### 构建无状态环境

1. **准备 `bsd.rd`**
   从发布目录获取，通过 TFTP 分发或放到 USB。
2. **创建 `siteXX.tgz`**
   加入 **/etc** 配置、脚本与 `install.site`。
3. **用 `install.site` 自动化**
   示例：

   ```sh
   #!/bin/sh
   echo "Stateless boot at $(date)" >> /var/log/stateless.log
   ftp -o /etc/runtime.conf https://config.example.org/host.conf
   mount_mfs -s 64m swap /var
   mount_mfs -s 64m swap /tmp
   ```
4. **启动系统**
   OpenBSD 解包 `siteXX.tgz` 并执行 `install.site`。

### 注意事项

- 每增加一个可写文件系统，内存占用随之上升。
- 日志与已安装的软件包若不导出，重启即丢失。
- 数据应在关机时上传到远程系统。

## 故障排查

- 检查 [dmesg(8)](https://man.openbsdhandbook.com/dmesg.8/) 与安装程序输出。
- 确认 UEFI/BIOS 设置。
- 务必查阅发布目录中的 `INSTALL.arch` 文件
