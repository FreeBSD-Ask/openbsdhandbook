# Install Z shell (zsh)

## Overview

The Z shell, abbreviated **zsh**, is a Unix shell that functions as both an interactive command interpreter and a scripting language. It is compatible with the Bourne shell while providing advanced features such as programmable completion, extended globbing, and flexible prompt customization. This chapter describes how to install zsh on OpenBSD, set it as the login shell, and perform initial configuration.

A **login shell** is the command interpreter that starts when a user logs in to the system. An **interactive shell** is a shell that reads commands from a terminal.

## Installation

Install the zsh package from the OpenBSD packages collection. Run package management commands with root privileges.

```sh
# doas pkg_add zsh
```

Example output:

```
quirks-*-signed-on-*
zsh-*-*: ok
```

After installation, confirm the binary path and ensure it is permitted as a login shell.

```sh
$ command -v zsh
$ which zsh
```

On OpenBSD, zsh installs to `/usr/local/bin/zsh`. The list of valid login shells resides in `/etc/shells`. If `/usr/local/bin/zsh` does not appear in that file, add it.

```sh
# grep '^/usr/local/bin/zsh$' /etc/shells || echo /usr/local/bin/zsh | doas tee -a /etc/shells
```

## Changing the Login Shell

You can change the login shell with the `chsh` utility or by editing the user database with `vipw`. The `chsh` utility updates the shell for the invoking user and requires the target shell to be listed in `/etc/shells`.

### Using `chsh`

Set zsh as the login shell for the current user. Use the absolute path.

```sh
$ chsh -s /usr/local/bin/zsh
```

Log out and log in again to start the new shell.

### Using `vipw`

Alternatively, edit the user database with record locking using `vipw`. Locate the current entry and replace the shell field.

Display the zsh path for reference.

```sh
$ which zsh
/usr/local/bin/zsh
```

Edit the password database.

```sh
# vipw
```

Before:

```
user:*:1000:1000:Firstname Lastname:/home/user:/bin/ksh
```

After:

```
user:*:1000:1000:Firstname Lastname:/home/user:/usr/local/bin/zsh
```

Write the changes and exit the editor, then log out and log back in.

## Initial Configuration

On first start without user startup files, zsh displays the `zsh-newuser-install` helper. It offers to create or customize basic configuration files such as `.zshenv`, `.zprofile`, `.zshrc`, and `.zlogin`.

Example prompt:

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

Choose option `1` to proceed with guided configuration. The wizard can set history behavior, completion, key bindings, and common shell options.

Example menu:

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

If you prefer manual control, you can exit the wizard and create `~/.zshrc` yourself.

```sh
$ : > ~/.zshrc
```

## Optional: Using oh-my-zsh

**oh-my-zsh** is a third-party configuration framework that provides themes and plugins for zsh. It maintains `~/.zshrc` and related assets in `~/.oh-my-zsh`. Because it modifies your shell startup files, review the install script and version control your configuration as needed.

Install the required tools and run the official installer. The commands below use `curl` and `git`.

```sh
# doas pkg_add curl git
$ sh -c 'REMOTE=https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh; \
          curl -fsSL "$REMOTE" -o /tmp/install.sh && sh /tmp/install.sh'
```

After installation, set a theme by editing `~/.zshrc`. The following example selects the `bira` theme.

```conf
# ~/.zshrc
ZSH_THEME="bira"
```

Reload the configuration in the current shell session.

```sh
$ source ~/.zshrc
```

## Applying Changes

Configuration changes in `~/.zshrc` take effect for new shells. To ensure a clean environment, log out and log back in. You can also start a new zsh instance to test incremental changes.

```sh
$ zsh -f
```

The `-f` option starts zsh without reading startup files, which is useful for troubleshooting configuration issues.
