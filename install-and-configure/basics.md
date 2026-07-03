# OpenBSD 基础

## Synopsis

本章介绍 OpenBSD 的基本概念与管理工具，讲解如何管理用户与组、控制文件权限、配置 shell 环境、操作进程、管理服务以及处理已安装的软件。本章还描述重要的约定，并概述 OpenBSD 独有的关键安全设计原则。

## 用户与组管理

OpenBSD 中的用户与组账户为系统资源与服务定义访问控制边界。

### 账户类型

- **Root**（`UID 0`）：超级用户账户，拥有不受限制的访问权限。
- **普通用户**：交互式账户，通常属于 `users` 或 `staff` 等组。
- **系统账户**：非登录账户（例如 `_ntpd`、`_smtp`），使用 **/sbin/nologin** 等受限 shell，由守护进程与系统服务使用。

### 创建用户

交互式创建用户：

```sh
# adduser
```

`adduser(8)` 脚本会提示输入登录名、全名、shell、组与密码。

非交互式创建用户：

```sh
# useradd -m -s /bin/ksh -G wheel alice
# passwd alice
```

- `-m`：创建主目录。
- `-s`：设置 shell（默认为 **/bin/ksh**）。
- `-G`：将用户加入附加组（例如 `wheel`）。

### `wheel` 组

要用 `doas(1)` 或 `su(1)` 执行特权操作，必须属于 `wheel` 组。

**/etc/doas.conf** 示例条目：

```
permit persist :wheel
```

只有 `wheel` 组的用户才能以 root 身份执行命令。

### 查看用户信息

下列命令显示用户账户数据：

```sh
$ id
$ whoami
$ groups
$ getent passwd alice
```

- `id`：显示 UID 与 GID 信息。
- `whoami`：显示当前用户名。
- `groups`：列出附加组。
- `getent`：从系统数据库获取账户详情。

### 修改用户

添加附加组：

```sh
# usermod -G wheel,staff alice
```

更改主组：

```sh
# usermod -g staff alice
```

交互式编辑用户属性：

```sh
# chpass alice
```

### 删除用户

删除用户并删除其主目录：

```sh
# userdel -r alice
```

## 文件权限与属主

OpenBSD 采用传统 UNIX 风格的文件权限定义访问控制。每个文件与目录对三类用户关联权限：

- **属主（user）** — 文件创建者或指定属主
- **组（group）** — 属于文件所属组的用户
- **其他人（others）** — 其余所有用户

每类用户可有读（`r`）、写（`w`）与执行（`x`）权限。

### 查看权限

用 `ls -l` 查看权限：

```sh
$ ls -l /etc/passwd
```

示例输出：

```
-rw-r--r--  1 root  wheel  1234 Jul 10 12:00 /etc/passwd
```

含义如下：

- `-`：普通文件（目录显示 `d`）
- `rw-`：属主（`root`）可读写
- `r--`：组（`wheel`）只读
- `r--`：其他人只读

### 理解数字与符号权限

每类权限（用户、组、其他人）用 0 到 7 之间的数字表示，由以下值相加：

- `4` = 读（`r`）
- `2` = 写（`w`）
- `1` = 执行（`x`）

下表列出所有组合：

| 值 | 符号 | 含义 | 目录列表 |
| --- | --- | --- | --- |
| 0 | — | 无权限 | — |
| 1 | –x | 仅执行 | –x |
| 2 | -w- | 仅写 | -w- |
| 3 | -wx | 写与执行 | -wx |
| 4 | r– | 仅读 | r– |
| 5 | r-x | 读与执行 | r-x |
| 6 | rw- | 读与写 | rw- |
| 7 | rwx | 读、写与执行 | rwx |

完整权限模式如 `755` 拆解如下：

- `7` = `rwx`，属主
- `5` = `r-x`，组
- `5` = `r-x`，其他人

应用该模式：

```sh
# chmod 755 script.sh
```

### 用符号模式更改权限

符号模式可对单个权限位做显式更改：

```sh
# chmod u+x script.sh
# chmod g-w file.txt
# chmod o= file.txt
```

- `u`、`g`、`o`、`a`：用户、组、其他人、全部
- `+`、`-`、`=`：添加、移除、精确设置
- `r`、`w`、`x`：权限类型

精细调整用符号模式，三类权限一次性设定用数字模式。

---

### 特殊权限位

除标准权限位外，OpenBSD 还支持三种**特殊**模式：

| 特殊位 | 八进制前缀 | 符号（ls -l） | 含义 |
| --- | --- | --- | --- |
| setuid | 4000 | 用户执行位为 `s` | 可执行文件以属主 UID 运行 |
| setgid | 2000 | 组执行位为 `s` | 可执行文件以组 GID 运行；目录继承组 |
| sticky 位 | 1000 | 其他人执行位为 `t` | 目录中只有文件属主或 root 可删除文件 |

#### setuid 示例

```sh
# chmod 4755 /usr/local/bin/somescript
```

设置为：

- `4`（setuid）+ `755` → `4755`
- 属主执行位变为 `s`：`-rwsr-xr-x`

用 `ls -l` 查看 `s`：

```sh
$ ls -l /usr/local/bin/somescript
```

#### setgid 示例

```sh
# chmod 2755 /usr/local/bin/teamcmd
```

- 组执行位变为 `s`：`-rwxr-sr-x`

在目录上设置 `setgid`，新文件会继承目录的组：

```sh
# chmod 2775 /var/shared
```

#### sticky 位示例

在 **/tmp** 等共享目录上，sticky 位阻止用户删除他人的文件：

```sh
# chmod 1777 /tmp
```

**/tmp** 中的文件只能由其属主或 `root` 删除，即便目录对所有人可写。

sticky 位显示为 `t`：

```sh
$ ls -ld /tmp
```

```
drwxrwxrwt  10 root  wheel  5120 Jul 11 15:10 /tmp
```

---

特殊位与标准权限组合时，相应地在数字模式前加前缀：

- `chmod 4755` → setuid + `755`
- `chmod 2755` → setgid + `755`
- `chmod 1777` → sticky + `777`

请谨慎使用这些位：这些位会影响安全与进程行为。

## Shell 配置

OpenBSD 默认 shell 为 `ksh(1)`（基于 Public Domain Korn Shell）。

### Shell 配置文件

用户 shell 初始化文件包括：

- `~/.profile`：登录 shell 执行
- `~/.kshrc`：用于交互式 `ksh` 会话（在 `.profile` 中引用）

最小化的 `.profile`：

```sh
export ENV="$HOME/.kshrc"
export PATH="/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin"
umask 022
```

## 进程管理

进程可用标准工具列出、控制与监视。

### 查看进程

```sh
$ ps aux
$ top
```

- `ps aux`：显示所有运行中的进程。
- `top`：提供交互式实时视图。

### 控制进程

```sh
$ kill -TERM 1234    # 优雅停止进程
$ kill -KILL 1234    # 强制终止
$ pkill httpd        # 按名称终止
```

## 服务管理

OpenBSD 服务用 `rcctl(8)` 管理。

### 启用与启动服务

```sh
# rcctl enable ntpd
# rcctl start ntpd
```

### 停止与禁用服务

```sh
# rcctl stop ntpd
# rcctl disable ntpd
```

### 重启与配置

```sh
# rcctl restart httpd
# rcctl set httpd flags "-d"
```

查看服务状态：

```sh
# rcctl check ntpd
```

## 已安装软件与版本信息

OpenBSD 包含最小化的基本系统，其他软件用 `pkg_add(1)` 安装。

查看系统版本：

```sh
$ uname -a
```

示例输出：

```
OpenBSD myhost.example.org 7.8 GENERIC.MP#123 amd64
```
