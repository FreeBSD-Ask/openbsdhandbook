# 安装 Z shell（zsh）

## 概览

Z shell 简称 **zsh**，是一款 Unix shell，既可作为交互式命令解释器，也可作为脚本语言。它兼容 Bourne shell，并提供可编程补全、扩展通配、灵活的提示符自定义等高级特性。本章介绍如何在 OpenBSD 上安装 zsh、将其设为登录 shell，并完成初始配置。

**登录 shell** 是用户登录系统时启动的命令解释器。**交互式 shell** 则是从终端读取命令的 shell。

## 安装

从 OpenBSD 软件包仓库安装 zsh。运行包管理命令需要 root 权限。

```sh
# doas pkg_add zsh
```

示例输出：

```
quirks-*-signed-on-*
zsh-*-*: ok
```

安装完成后，确认可执行文件路径，并确保系统允许其作为登录 shell。

```sh
$ command -v zsh
$ which zsh
```

OpenBSD 上 zsh 安装到 `/usr/local/bin/zsh`。合法登录 shell 列表存放在 `/etc/shells`。若该文件中没有 `/usr/local/bin/zsh`，将其追加进去。

```sh
# grep '^/usr/local/bin/zsh$' /etc/shells || echo /usr/local/bin/zsh | doas tee -a /etc/shells
```

## 修改登录 shell

可用 `chsh` 工具修改登录 shell，也可用 `vipw` 编辑用户数据库。`chsh` 会更新调用者自身的 shell，且要求目标 shell 已列入 `/etc/shells`。

### 使用 `chsh`

为当前用户将 zsh 设为登录 shell。使用绝对路径。

```sh
$ chsh -s /usr/local/bin/zsh
```

注销后重新登录即可启用新 shell。

### 使用 `vipw`

也可用 `vipw` 在加锁状态下编辑用户数据库。找到当前条目，替换其中的 shell 字段。

先查看 zsh 路径以便参照：

```sh
$ which zsh
/usr/local/bin/zsh
```

编辑密码数据库：

```sh
# vipw
```

修改前：

```
user:*:1000:1000:Firstname Lastname:/home/user:/bin/ksh
```

修改后：

```
user:*:1000:1000:Firstname Lastname:/home/user:/usr/local/bin/zsh
```

保存并退出编辑器，然后注销重新登录。

## 初始配置

首次启动时若没有用户启动文件，zsh 会显示 `zsh-newuser-install` 助手。它可帮助创建或定制 `.zshenv`、`.zprofile`、`.zshrc`、`.zlogin` 等基础配置文件。

示例提示：

```
This is the Z Shell configuration function for new users,
zsh-newuser-install.
You are seeing this message because you have no zsh startup files
(the files .zshenv, .zprofile, .zshrc, .zlogin in the directory
~). This function can help you with a few settings that should
make your use of the shell easier.

You can:

(q)  Quit and do nothing.  The function will be run again next time.

(0)  Exit, creating the file ~/.zshrc containing just a comment.
     That will prevent this function being run again.

(1)  Continue to the main menu.

--- Type one of the keys in parentheses ---
```

选择 `1` 进入引导式配置。向导可设置历史行为、补全、按键绑定以及常用 shell 选项。

示例菜单：

```
Please pick one of the following options:

(1)  Configure settings for history, i.e. command lines remembered
     and saved by the shell.  (Recommended.)

(2)  Configure the new completion system.  (Recommended.)

(3)  Configure how keys behave when editing command lines.  (Recommended.)

(4)  Pick some of the more common shell options.  These are simple "on"
     or "off" switches controlling the shell's features.

(0)  Exit, creating a blank ~/.zshrc file.

(a)  Abort all settings and start from scratch.  Note this will overwrite
     any settings from zsh-newuser-install already in the startup file.
     It will not alter any of your other settings, however.

(q)  Quit and do nothing else.  The function will be run again next time.
--- Type one of the keys in parentheses ---
```

若偏好手动控制，可退出向导自行创建 `~/.zshrc`。

```sh
$ : > ~/.zshrc
```

## 可选：使用 oh-my-zsh

**oh-my-zsh** 是第三方配置框架，为 zsh 提供主题与插件。它会把 `~/.zshrc` 以及相关资源维护在 `~/.oh-my-zsh` 目录下。由于该框架会修改 shell 启动文件，请先审阅安装脚本，并视需要将配置纳入版本控制。

安装所需工具后运行官方安装脚本。下列命令使用 `curl` 与 `git`。

```sh
# doas pkg_add curl git
$ sh -c 'REMOTE=https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh; \
          curl -fsSL "$REMOTE" -o /tmp/install.sh && sh /tmp/install.sh'
```

安装完成后，编辑 `~/.zshrc` 设置主题。以下示例选用 `bira` 主题。

```conf
# ~/.zshrc
ZSH_THEME="bira"
```

在当前 shell 会话中重新载入配置：

```sh
$ source ~/.zshrc
```

## 应用改动

`~/.zshrc` 中的配置改动对新启动的 shell 生效。为确保环境干净，可注销后重新登录。也可启动新的 zsh 实例以测试增量改动。

```sh
$ zsh -f
```

`-f` 选项让 zsh 启动时不读取启动文件，便于排查配置问题。
