# 打印

## 概述

OpenBSD 既支持传统的 BSD 打印系统（`lpd`），也支持功能更丰富的 CUPS（Common UNIX Printing System）。本章介绍如何配置本地与网络打印机、安装打印机驱动与过滤器、打印到 PDF，以及将 OpenBSD 系统配置为打印服务器。

## OpenBSD 打印概述

基本系统包含 `lpd(8)`，这是一个轻量级的行式打印机守护进程，负责打印队列管理与基本过滤。涉及驱动、图形化管理或网络浏览等更复杂的场景，更适合用 CUPS 处理，CUPS 通过软件包提供。

支持的常见打印机类型：

- 通过 **/dev/ulpt0** 连接的 USB 打印机
- 支持 IPP、JetDirect（端口 9100）或 LPD 的网络打印机
- 无需额外驱动的 PostScript 打印机
- 通过驱动程序包支持的主机型打印机（如部分 HP、Samsung）

对于不支持纯文本或 PostScript 的打印机，常用 `a2ps(1)`、`enscript(1)` 和 `ghostscript` 等过滤器准备打印任务。

## 用 lpd 配置打印

`lpd(8)` 是 OpenBSD 基本系统内置的传统 Berkeley 行式打印机队列管理器，简单可靠，适合不需要自动驱动检测或图形化配置的本地或网络打印。打印机需在 **/etc/printcap** 中手动配置，格式转换（如纯文本转 PostScript）由用户提供的过滤器处理。`lpd` 轻量，常用于小型或专用系统。

### 启用 lpd

用 `rcctl` 启用基本打印系统：

```sh
# rcctl enable lpd
# rcctl start lpd
```

确保打印设备（如 **/dev/ulpt0**）对 `daemon` 用户可写：

```sh
# chown daemon /dev/ulpt0
# chmod 600 /dev/ulpt0
```

### 配置 /etc/printcap

`printcap(5)` 文件描述可用的打印机。

本地 USB 打印机示例：

```conf
lp|Local USB Printer:\
    :lp=/dev/ulpt0:\
    :sd=/var/spool/output/lpd:\
    :lf=/var/log/lpd-errs:
```

创建并准备队列目录：

```sh
# mkdir -p /var/spool/output/lpd
# chown daemon:daemon /var/spool/output/lpd
```

打印到远程 LPD 服务器：

```conf
remotehp:\
    :rm=192.0.2.10:\
    :rp=raw:\
    :sd=/var/spool/output/remotehp:\
    :lf=/var/log/lpd-errs:
```

### 添加打印过滤器

若打印机不支持纯文本，需要通过 `if=` 字段指定打印过滤器。

安装 `enscript` 和 `ghostscript`：

```sh
# pkg_add enscript ghostscript
```

创建过滤脚本：

```sh
# vi /usr/local/libexec/psfilter
```

内容示例：

```sh
#!/bin/sh
exec enscript -o - | gs -q -dNOPAUSE -dBATCH -sDEVICE=ijs -sOutputFile=- - 2>/dev/null
```

此管道流程如下：

- `enscript -o -` 将纯文本转为 PostScript，并发送到标准输出。
- `gs`（Ghostscript）读取 PostScript 输入，用 ijs 驱动光栅化，再输出到标准输出。
- `-q`、`-dNOPAUSE` 和 `-dBATCH` 抑制提示信息，处理完成后退出。
- `-sDEVICE=ijs` 选择 IJS 光栅输出接口；根据打印机不同，也可选用 pxlmono、ljet4、uniprint 等。
- `-sOutputFile=-` 写入标准输出；2>/dev/null 抑制错误信息。

这样 lpd 就能借助外部过滤器支持多种打印机。运行 gs -h 可列出支持的输出设备。

设为可执行：

```sh
# chmod +x /usr/local/libexec/psfilter
```

更新 **/etc/printcap**：

```conf
lp:\
    :lp=/dev/ulpt0:\
    :if=/usr/local/libexec/psfilter:\
    :sd=/var/spool/output/lpd:\
    :lf=/var/log/lpd-errs:
```

### 从命令行打印

提交打印任务：

```sh
$ lpr file.txt
```

查看队列：

```sh
$ lpq
```

删除任务：

```sh
$ lprm -
```

## 用 CUPS 打印

CUPS（Common UNIX Printing System）是一款模块化打印系统，支持多种打印机协议、驱动和管理工具。与 `lpd` 不同，CUPS 提供 Web 界面、自动打印机发现，并与许多现代打印流程集成。CUPS 不属于 OpenBSD 基本系统，但可通过软件包安装，适合 USB 喷墨打印机、需要 PPD 文件的打印机或复杂网络打印环境使用。

### 安装与启用 CUPS

安装 CUPS 及相关驱动：

```sh
# pkg_add cups gutenprint foomatic-db splix hplip
```

启用并启动守护进程：

```sh
# rcctl enable cupsd
# rcctl start cupsd
```

### 访问 Web 界面

CUPS 的 Web 界面访问地址：

```
http://localhost:631
```

访问可能需要将用户加入 `operator` 组，并创建 `doas` 规则：

```
permit persist :operator as root cmd /usr/local/sbin/lpadmin
```

可通过 Web 界面或 `lpadmin` 添加打印机。

### 驱动与后端支持

查看后端：

```sh
$ lpinfo -v
```

列出可用型号与驱动：

```sh
$ lpinfo -m
```

CUPS 后端支持的协议包括：

- `socket://` — JetDirect
- `ipp://` 或 `http://` — IPP 打印机
- `usb://` — 本地打印机

### 用 CUPS 打印

打印：

```sh
$ lp -d PRINTER file.pdf
```

设置默认打印机：

```sh
$ lpoptions -d PRINTER
```

查看任务：

```sh
$ lpstat -o
```

取消任务：

```sh
$ cancel JOBID
```

## 搭建 OpenBSD 打印服务器

### 基于 lpd 的打印服务器

启用 `lpd` 并在 **/etc/printcap** 中配置对外提供的打印机。将信任的客户端主机加入 **/etc/hosts.lpd**：

```
192.0.2.15
client.example.org
```

客户端用法：

```sh
$ lpr -Pprinter@server file.txt
```

### 基于 CUPS 的打印服务器

编辑 **/etc/cups/cupsd.conf**：

```
Listen *:631
Browsing On
BrowseLocalProtocols all
<Location />
  Order allow,deny
  Allow all
</Location>
```

重启守护进程：

```sh
# rcctl restart cupsd
```

客户端系统配置：

```text
/etc/cups/client.conf
```

内容：

```
ServerName printserver.example.net
```

之后客户端无需其他配置即可使用 `lp`、`lpr` 或打印对话框。

## 打印到 PDF

### 从图形应用打印

多数支持 GTK、Qt 或 X11 打印对话框的应用都允许打印到文件。选择"打印到文件"或"打印到 PDF"，并选择目标位置。

### 使用命令行工具

用 `enscript` 和 `ghostscript` 将文本转为 PDF：

```sh
$ enscript file.txt -p - | ps2pdf - output.pdf
```

或直接使用 PostScript：

```sh
$ ps2pdf input.ps output.pdf
```

### 用 CUPS 配置虚拟 PDF 打印机

安装虚拟后端：

```sh
# pkg_add cups-pdf
```

在 CUPS Web 界面中添加名为"PDF"的新打印机。输出将写入：

```text
/var/spool/cups-pdf/USERNAME
```

输出目录可在 **/etc/cups/cups-pdf.conf** 中修改。

## 远程打印

### 打印到远程 lpd 服务器

配置 **/etc/printcap**：

```conf
remote:\
    :rm=printserver.example.net:\
    :rp=lp:\
    :sd=/var/spool/output/remote:\
    :lf=/var/log/lpd-errs:
```

### 打印到远程 CUPS 服务器

创建配置文件：

```text
/etc/cups/client.conf
```

内容：

```
ServerName cups.example.net
```

或设置环境变量：

```sh
$ export CUPS_SERVER=cups.example.net
```

### 打印到 Windows 共享打印机

安装 `samba`：

```sh
# pkg_add samba
```

使用如下设备 URI：

```
smb://WORKGROUP/username:password@host/sharename
```

CUPS 可以用 `smbspool` 作为 SMB 打印的后端。若需无人值守打印，确保凭据已嵌入 URI。

## 故障排查

- 对于 `lpd`，检查 **/var/log/lpd-errs**
- 对于 CUPS，使用：

```sh
$ tail -f /var/log/cups/error_log
```

- 检查打印机设备权限：

```sh
# ls -l /dev/ulpt0
```

- 修改后重启服务：

```sh
# rcctl restart lpd
# rcctl restart cupsd
```

- 任务失败时，确认已安装正确的驱动或过滤器
- 检查队列目录是否存在，且对 `daemon`（用于 `lpd`）或 CUPS 用户可写
