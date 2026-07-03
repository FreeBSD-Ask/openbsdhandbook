# 安全

本章介绍 OpenBSD 系统中常见的权限提升方式。日常管理应优先采用尽可能小的权限边界，仅在确实需要时才使用完整的 root shell。

OpenBSD 基本系统内置 [doas(1)](https://man.openbsdhandbook.com/doas.1/)
。软件包也提供 sudo 等替代方案，但默认不安装。通过 [su(1)](https://man.openbsdhandbook.com/su.1/)
直接以 root 登录的方式应慎用，因其需要 root 口令并授予完整 root shell。

## Doas

[doas(1)](https://man.openbsdhandbook.com/doas.1/)
工具允许用户以其他用户身份运行命令。管理员最常用它以 root 身份执行选定的命令。

配置文件为 [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/)
，位于 `/etc/doas.conf`。该文件默认不存在。OpenBSD 在 `/etc/examples/` 下提供了示例：

```
# cp /etc/examples/doas.conf /etc/
```

默认示例允许 `wheel` 组成员以 root 身份运行命令。与多数 sudo 配置不同，除非配置了 `persist` 选项，`doas` 每次执行命令都要求认证。

```
# 默认允许 wheel 组
permit persist keepenv :wheel
```

默认情况下，除非用 `-u` 指定其他目标用户，`doas` 以 root 身份运行命令。通过 `doas` 执行的命令记录到 `/var/log/secure`。

要验证 `doas` 是否对管理账户生效，可运行一条需要提权的命令，例如 [vipw(8)](https://man.openbsdhandbook.com/vipw.8/)
：

```sh
$ doas vipw
```

若账户未获授权或配置有误，命令将失败。可使用 [id(1)](https://man.openbsdhandbook.com/id.1/)
核对组成员身份，或用 [getent(1)](https://man.openbsdhandbook.com/getent.1/)
查看 `wheel` 组条目：

```sh
$ id
$ getent group wheel
```

更多示例参见 [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/)
。

## Sudo

sudo 工具在类 Unix 系统中应用广泛。OpenBSD 以软件包形式提供，但默认不安装，因基本系统已包含 `doas`。

使用 [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
安装 sudo：

```sh
# pkg_add sudo
```

默认仅 root 可使用 sudo。要允许 `wheel` 组成员使用，需用 `visudo` 编辑 `/etc/sudoers` 并启用相应规则。

**visudo** 命令在写入前检查 `/etc/sudoers` 语法，有助于避免配置错误。请勿直接编辑该文件。

```conf
# 取消注释以允许 wheel 组成员运行所有命令
# 并设置环境变量。
# %wheel        ALL=(ALL) SETENV: ALL
```

查看 sudo 权限：

```sh
$ sudo -l
```

获授权用户应看到类似如下的规则：

```
(ALL) SETENV: ALL
```

若用户不在 `wheel` 组，sudo 会提示该用户无法在主机上运行 sudo。使用 [usermod(8)](https://man.openbsdhandbook.com/usermod.8/)
将用户加入 `wheel`：

```
# usermod -G wheel USER
```

## Su

[su(1)](https://man.openbsdhandbook.com/su.1/)
工具以其他用户身份启动 shell。管理员常用它切换为 root，但此举会授予完整 root shell 并要求 root 口令。日常管理命令应优先使用 `doas`。

只有 `wheel` 组成员才能用 `su` 切换为 root：

```sh
$ su -
```
