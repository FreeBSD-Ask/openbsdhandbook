# OpenBSD 速查表

本速查表以简洁命令汇总常见 OpenBSD 管理任务。命令中 `$` 表示普通用户，`#` 表示超级用户。用 [doas(1)](https://man.openbsdhandbook.com/doas.1/)
提权，并通过 [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/)
配置。

## 提权

首次配置可参考示例文件：

```sh
$ doas cp /etc/examples/doas.conf /etc/doas.conf  # 从示例复制配置
```

为 wheel 组启用持久认证：

```
permit persist keepenv :wheel
```

测试：

```sh
$ doas id  # 通过 doas 执行一条简单命令
```

## 软件包

包管理工具：[pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
、[pkg\_delete(1)](https://man.openbsdhandbook.com/pkg_delete.1/)
、[pkg\_info(1)](https://man.openbsdhandbook.com/pkg_info.1/)
，总览见 [packages(7)](https://man.openbsdhandbook.com/packages.7/)
。

```sh
$ doas pkg_add htop            # 安装软件包
$ doas pkg_delete htop         # 删除软件包
$ pkg_info | less              # 列出已安装软件包
$ doas pkg_add -Uu             # 升级当前发行版全部软件包
```

软件包镜像通过 [installurl(5)](https://man.openbsdhandbook.com/installurl.5/)
配置。

## 基本系统更新与版本升级

基本系统的安全/缺陷补丁：[syspatch(8)](https://man.openbsdhandbook.com/syspatch.8/)
。
版本升级：[sysupgrade(8)](https://man.openbsdhandbook.com/sysupgrade.8/)
。

```sh
$ doas syspatch -c  # 列出待应用的基系统补丁
$ doas syspatch     # 应用基系统补丁（按提示重启）

$ doas sysupgrade   # 拉取并升级到下一发行版
$ doas pkg_add -Uu  # 版本升级后升级软件包
```

## 服务（rc）

服务管理：[rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
。系统启动：[rc(8)](https://man.openbsdhandbook.com/rc.8/)
、[rc.conf(8)](https://man.openbsdhandbook.com/rc.conf.8/)
。

```sh
$ doas rcctl check sshd      # 查看状态
$ doas rcctl start sshd       # 立即启动
$ doas rcctl stop sshd        # 立即停止
$ doas rcctl reload sshd      # 重新载入配置
$ doas rcctl enable sshd      # 开机自启
$ doas rcctl disable sshd     # 取消开机自启
$ doas rcctl ls on            # 列出已启用服务
$ doas rcctl get sshd flags   # 查看守护进程标志
$ doas rcctl set sshd flags "-v"  # 持久设置守护进程标志
```

## 网络

网络接口：[ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
。持久化配置文件：[hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
、[myname(5)](https://man.openbsdhandbook.com/myname.5/)
、[mygate(5)](https://man.openbsdhandbook.com/mygate.5/)
、[resolv.conf(5)](https://man.openbsdhandbook.com/resolv.conf.5/)
。用 [netstart(8)](https://man.openbsdhandbook.com/netstart.8/)
应用配置。

### 查看与临时改动

```sh
$ ifconfig -A                 # 全部接口
$ doas ifconfig em0 10.0.0.10 255.255.255.0  # 临时 IPv4
$ doas ifconfig em0 inet6 2001:db8::10 64    # 临时 IPv6
$ doas sh /etc/netstart em0   # 重新载入单个接口
$ doas sh /etc/netstart       # 按文件重新载入全部接口
```

### 持久化示例

`/etc/hostname.em0`（从以下形式中任选一种）：

```
dhcp
```
```
inet 192.0.2.10 255.255.255.0
```
```
inet6 2001:db8:6000:1::10 64
```

主机名、默认网关与解析器：

```
# /etc/myname
host.example.com
```
```
# /etc/mygate
192.0.2.1
2001:db8:6000:1::1
```
```
# /etc/resolv.conf
nameserver 192.0.2.53
lookup file bind
```

### 常用网络诊断

```sh
$ ping -n 1.1.1.1            # ICMP 测试（不做 DNS 解析）
$ traceroute -n 1.1.1.1      # 不做 DNS 解析的路径追踪
$ netstat -rn                # 路由表
$ route show                 # 路由套接字视图
$ tcpdump -ni em0 port 53    # 在 em0 上抓取 DNS
```

## 防火墙（PF）

包过滤工具：[pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
。配置：[pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
。

### 快捷操作

```sh
$ doas pfctl -sr           # 查看规则
$ doas pfctl -sn           # 查看 NAT
$ doas pfctl -si           # 查看统计
$ doas pfctl -e            # 启用 PF
$ doas pfctl -d            # 禁用 PF
$ doas pfctl -f /etc/pf.conf  # 重新载入规则集
```

### 最小放行规则集

```pf
# /etc/pf.conf
set block-policy drop
set skip on lo

block all
pass out inet proto { tcp udp icmp } from (egress) to any modulate state
pass out inet6 proto { tcp udp icmp6 } from (egress) to any modulate state
```

激活：

```sh
$ doas pfctl -f /etc/pf.conf
$ doas pfctl -e
```

## Web 与 SSH 守护进程

HTTP 服务器：[httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
配合 [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)
。
安全 Shell：[sshd(8)](https://man.openbsdhandbook.com/sshd.8/)
、[sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)
。

```sh
$ doas rcctl enable httpd
$ doas rcctl start httpd
$ doas rcctl enable sshd
$ doas rcctl start sshd
```

## 用户与组

用户与组管理：[useradd(8)](https://man.openbsdhandbook.com/useradd.8/)
、[usermod(8)](https://man.openbsdhandbook.com/usermod.8/)
、[userdel(8)](https://man.openbsdhandbook.com/userdel.8/)
、[groupadd(8)](https://man.openbsdhandbook.com/groupadd.8/)
、[passwd(1)](https://man.openbsdhandbook.com/passwd.1/)
、chsh(1)。

```sh
$ doas useradd -m -G wheel -s /bin/ksh alice  # 添加管理员用户
$ doas passwd alice                            # 设置密码
$ doas usermod -G wheel,staff alice            # 调整所属组
$ doas userdel -r bob                          # 删除用户及其主目录
$ chsh -s /bin/ksh                             # 修改自己的登录 shell
```

## 文件系统与存储

文件系统：[fstab(5)](https://man.openbsdhandbook.com/fstab.5/)
、[mount(8)](https://man.openbsdhandbook.com/mount.8/)
、[umount(8)](https://man.openbsdhandbook.com/umount.8/)
、[df(1)](https://man.openbsdhandbook.com/df.1/)
。磁盘设置：[fdisk(8)](https://man.openbsdhandbook.com/fdisk.8/)
（MBR/GPT）、[disklabel(8)](https://man.openbsdhandbook.com/disklabel.8/)
。

```sh
$ df -h                          # 查看使用量
$ mount                          # 已挂载文件系统
$ doas mount /home               # 按 fstab 条目挂载
$ doas umount /home              # 卸载
$ dmesg | egrep '^sd|^wd|^cd'    # 列出启动时识别到的磁盘
$ doas disklabel -E sd0          # 编辑 OpenBSD disklabel
```

## 时间与 ntpd

网络时间：[ntpd(8)](https://man.openbsdhandbook.com/ntpd.8/)
。配置：[ntpd.conf(5)](https://man.openbsdhandbook.com/ntpd.conf.5/)
。

```sh
$ doas rcctl enable ntpd
$ doas rcctl start ntpd
$ rcctl check ntpd
```

## 日志与轮转

Syslog：[syslogd(8)](https://man.openbsdhandbook.com/syslogd.8/)
，通过 [syslog.conf(5)](https://man.openbsdhandbook.com/syslog.conf.5/)
配置。日志轮转：[newsyslog(8)](https://man.openbsdhandbook.com/newsyslog.8/)
。

```sh
$ doas tail -f /var/log/messages       # 主系统日志
$ doas tail -f /var/log/daemon         # 守护进程日志
$ doas newsyslog -n                    # 预览将轮转的文件
$ doas newsyslog                       # 立即轮转
```

## 系统信息与诊断

```sh
$ dmesg | less               # 内核消息与硬件探测
$ sysctl kern.version        # 内核版本
$ sysctl hw.model hw.ncpu    # 硬件信息
$ top                        # 交互式进程/CPU 视图
$ vmstat 1                   # 系统计数器
$ systat iostat              # 各设备 I/O（curses）
$ systat ifstat              # 各接口流量（curses）
$ ps auxww | less            # 进程列表
$ pgrep httpd; pkill httpd   # 查找/终止进程
```

## OpenSSH 客户端基础

客户端：[ssh(1)](https://man.openbsdhandbook.com/ssh.1/)
，密钥用 [ssh-keygen(1)](https://man.openbsdhandbook.com/ssh-keygen.1/)
，配置见 [ssh\_config(5)](https://man.openbsdhandbook.com/ssh_config.5/)
。

```sh
$ ssh -o StrictHostKeyChecking=accept-new user@host
$ ssh-keygen -t ed25519 -C "me@host"     # 生成密钥
$ ssh-copy-id user@host                  # 若已安装；否则把 ~/.ssh/id_ed25519.pub 追加到服务器
```

## 常用路径

```
/etc/rc.conf.local   # 本地守护进程标志与启用项
/etc/installurl      # 软件包/升级镜像
/var/db/pkg/         # 已安装软件包元数据
/var/log/            # 系统日志
/etc/pf.conf         # PF 规则集
/etc/hostname.*      # 各接口网络配置
```

如需深入了解，请参阅各节引用的手册页。
