# 系统配置

## Synopsis

OpenBSD 的系统配置依赖少数定义良好的工具与 **/etc** 下的文本文件。设置可在运行时用 `sysctl`、`wsconsctl` 与 `rcctl` 等工具应用，也可写入 **/etc/sysctl.conf**、**/etc/wsconsctl.conf**、**/etc/rc.conf.local** 等文件持久化。此外，系统维护还包括管理服务行为、登录环境限制、引导加载程序参数、root 邮件通知与计划更新。

## 用 `sysctl` 配置内核参数

`sysctl` 工具用于显示与修改内核状态变量。这些变量控制网络、安全、硬件行为、内存限制与进程约束等方面。

显示所有值：

```sh
$ sysctl
```

读取或设置单个值：

```sh
$ sysctl hw.smt
# sysctl hw.smt=0
```

要在重启后保持设置，将其写入 **/etc/sysctl.conf**：

```conf
hw.smt=0
net.inet.ip.forwarding=1
kern.maxfiles=16384
net.inet.tcp.recvspace=65536
net.inet.tcp.sendspace=65536
kern.nosuidcoredump=1
vm.swapencrypt.enable=1
fs.posix.setuid=0
```

### 常见示例

- `hw.smt=0` — 禁用超线程（SMT），出于安全考虑常推荐启用此项。
- `net.inet.ip.forwarding=1` — 启用 IP 包转发，用于路由。
- `kern.maxfiles` — 增加打开文件描述符的数量。
- `vm.swapencrypt.enable=1` — 加密换出到磁盘的数据。
- `kern.nosuidcoredump=1` — 阻止 set-user-ID 二进制文件产生核心转储。
- `fs.posix.setuid=0` — 禁用 POSIX 信号量上的 setuid 行为。

**/etc/sysctl.conf** 的改动由系统启动脚本在引导时应用。

## 用 `wsconsctl` 配置控制台输入与显示

`wsconsctl` 工具用于修改控制台键盘与显示行为。设置可在运行时应用，也可保存到 **/etc/wsconsctl.conf** 持久化。

### 键盘布局与行为

调整键盘布局：

```sh
# wsconsctl keyboard.layout=us
```

其他设置：

```sh
# wsconsctl keyboard.bell.volume=0
# wsconsctl keyboard.repeat.del1=400
# wsconsctl keyboard.repeat.deln=40
```

要将键盘设置持久化，加入 **/etc/wsconsctl.conf**：

```conf
keyboard.layout=us
keyboard.bell.volume=0
keyboard.repeat.del1=400
keyboard.repeat.deln=40
```

### 控制台字体配置

字体可用 `wsfontload` 动态加载。例如：

```sh
# wsfontload -N Spleen32 -n 10 -f /usr/share/wscons/fonts/spleen32x64.fnt
# wsconsctl display.font=Spleen32
```

要在引导时加载字体，将字体加载命令放入 **/etc/rc.local**：

```sh
# vi /etc/rc.local
```

添加以下内容：

```conf
wsfontload -N Spleen32 -n 10 -f /usr/share/wscons/fonts/spleen32x64.fnt
wsconsctl display.font=Spleen32
exit 0
```

字体的选择取决于屏幕分辨率与个人偏好。字体位于 **/usr/share/wscons/fonts**。

## 用 `rcctl` 管理服务

OpenBSD 用 `rcctl` 启用、禁用与控制服务。服务定义在 **/etc/rc.d/** 下。

开机时启用服务：

```sh
# rcctl enable ntpd
```

立即启动服务：

```sh
# rcctl start ntpd
```

查看状态：

```sh
# rcctl check ntpd
```

禁用服务：

```sh
# rcctl disable ntpd
```

重启服务：

```sh
# rcctl restart ntpd
```

为服务设置运行时标志：

```sh
# rcctl set ntpd flags "-s"
```

这些设置保存在 **/etc/rc.conf.local**。

## 持久化系统配置文件

系统行为主要由 **/etc** 下的配置文件定义。关键文件概述如下：

- **/etc/rc.conf** — 默认系统设置（不要编辑）
- **/etc/rc.conf.local** — 服务与变量的本地覆盖
- **/etc/hostname.if** — 接口配置（例如 `hostname.em0`）
- **/etc/myname** — 系统主机名
- **/etc/mygate** — 默认路由（网关）
- **/etc/resolv.conf** — DNS 解析器配置
- **/etc/fstab** — 文件系统挂载配置
- **/etc/doas.conf** — 提权权限配置
- **/etc/login.conf** — 用户登录类与限制
- **/etc/sysctl.conf** — 内核参数
- **/etc/wsconsctl.conf** — 控制台与键盘设置
- **/etc/ttys** — 终端与控制台定义
- **/etc/mail/aliases** — 邮件别名，包括 root
- **/etc/rc.local** — 启动命令

### 编辑用户与组数据库

安全编辑密码与用户数据库：

```sh
# vipw
```

这能保证更新是原子的，相关影子文件也保持同步。修改组则直接用文本编辑器编辑 **/etc/group**。

### 使用 **/etc/examples**

安装或启用新组件时，配置模板常位于 **/etc/examples**。例如：

```sh
# cp /etc/examples/ntpd.conf /etc/ntpd.conf
```

这些默认值由系统维护，可安全复制与自定义。

## 登录类与资源限制

**/etc/login.conf** 文件定义用户类的环境与资源限制。每个用户关联一个类（例如 `default`、`staff`、`daemon`）。

示例定义：

```conf
staff:\
  :openfiles-cur=1024:\
  :stacksize-cur=8M:\
  :tc=default:
```

编辑该文件后，重建能力数据库：

```sh
# cap_mkdb /etc/login.conf
```

改动在下次登录时生效。

## 引导加载程序配置

引导加载程序行为通过 **/etc/boot.conf** 控制。该文件可包含在系统启动早期自动传递给引导加载程序的命令。

示例配置：

```
set timeout 5
boot /bsd
```

这会设置 5 秒延迟，并指示加载程序引导 `/bsd`。引导时，用户可中断倒计时并手动输入命令。引导提示符语法参见 `boot(8)`。

## Root 邮件与内部通知

OpenBSD 用本地邮件投递通知管理员系统事件。邮件来源包括：

- `cron(8)` 作业输出
- `daily(8)`、`weekly(8)` 与 `monthly(8)` 报告
- 系统守护进程的错误或消息

查看 root 的邮件：

```sh
# mail
```

将 root 邮件转发到外部地址，编辑 **/etc/mail/aliases**：

```
root: admin@example.com
```

然后运行：

```sh
# newaliases
```

邮件存放在 **/var/mail/root**。

## 备用根分区（**/altroot**）

OpenBSD 支持备用根分区，通常挂载于 **/altroot**，用于：

- 托装备用根文件系统
- 存放系统转储
- 保存冗余内核镜像或引导配置

用 `dump` 创建备份：

```sh
# dump -0au -f /altroot/root.dump /
```

该分区尽量位于独立的物理设备上。

## 计划自动更新

建议用 `syspatch` 自动化安全补丁等周期性任务。一种方式是把命令加入 **/etc/daily.local**，由每日脚本执行。

示例：

文件：**/etc/daily.local**

```sh
#!/bin/sh
syspatch -c
exit 0
```

赋予可执行权限：

```sh
# chmod +x /etc/daily.local
```

系统脚本还会在 **/var/log/** 下生成日志：

- **/var/log/daily**
- **/var/log/weekly**
- **/var/log/monthly**

这些日志会通过邮件发送给 root。请定期查阅，或重定向到别处。
