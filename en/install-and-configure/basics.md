# OpenBSD Basics

## Synopsis

This chapter introduces fundamental concepts and administrative tools in OpenBSD. It explains how to manage users and groups, control file permissions, configure shell environments, manipulate processes, manage services, and handle installed software. The chapter also describes important conventions and outlines key security design principles unique to OpenBSD.

## User and Group Management

User and group accounts in OpenBSD define access control boundaries for system resources and services.

### Account Types

- **Root** (`UID 0`): The superuser account with unrestricted access.
- **Regular Users**: Interactive accounts typically part of groups like `users` or `staff`.
- **System Accounts**: Non-login accounts (e.g., `_ntpd`, `_smtp`) with restricted shells like `/sbin/nologin`, used by daemons and system services.

### Creating Users

To create a user interactively:

```sh
# adduser
```

The `adduser(8)` script will prompt for login name, full name, shell, groups, and password.

To create a user non-interactively:

```sh
# useradd -m -s /bin/ksh -G wheel alice
# passwd alice
```

- `-m`: Creates the home directory.
- `-s`: Sets the shell (`/bin/ksh` is default).
- `-G`: Adds the user to supplementary groups (e.g., `wheel`).

### The `wheel` Group

Membership in the `wheel` group is required to perform privileged operations using `doas(1)` or `su(1)`.

Example `/etc/doas.conf` entry:

```
permit persist :wheel
```

Only users in the `wheel` group may execute commands as root.

### Inspecting User Information

The following commands show user account data:

```sh
$ id
$ whoami
$ groups
$ getent passwd alice
```

- `id`: Shows UID and GID info.
- `whoami`: Displays the current username.
- `groups`: Lists secondary groups.
- `getent`: Retrieves account details from the system database.

### Modifying Users

To add supplementary groups:

```sh
# usermod -G wheel,staff alice
```

To change the primary group:

```sh
# usermod -g staff alice
```

To edit user attributes interactively:

```sh
# chpass alice
```

### Removing Users

To remove a user and delete their home directory:

```sh
# userdel -r alice
```

## File Permissions and Ownership

OpenBSD uses traditional UNIX-style file permissions to define access control. Each file and directory has associated permissions for three categories of users:

- **Owner (user)** — the file’s creator or designated owner
- **Group** — users belonging to the file’s group
- **Others** — all remaining users

Each category can have read (`r`), write (`w`), and execute (`x`) permissions.

### Viewing Permissions

Permissions can be viewed with `ls -l`:

```sh
$ ls -l /etc/passwd
```

Example output:

```
-rw-r--r--  1 root  wheel  1234 Jul 10 12:00 /etc/passwd
```

This indicates:

- `-`: a regular file (directories show `d`)
- `rw-`: the owner (`root`) can read and write
- `r--`: the group (`wheel`) can read only
- `r--`: others can read only

### Understanding Numeric and Symbolic Permissions

Each permission class (user, group, others) is represented by a digit between 0 and 7, based on the sum of:

- `4` = read (`r`)
- `2` = write (`w`)
- `1` = execute (`x`)

The following table shows all possible combinations:

| Value | Symbolic | Meaning | Directory Listing |
| --- | --- | --- | --- |
| 0 | — | No permissions | — |
| 1 | –x | Execute only | –x |
| 2 | -w- | Write only | -w- |
| 3 | -wx | Write and execute | -wx |
| 4 | r– | Read only | r– |
| 5 | r-x | Read and execute | r-x |
| 6 | rw- | Read and write | rw- |
| 7 | rwx | Read, write, and execute | rwx |

A full permission mode such as `755` breaks down as:

- `7` = `rwx` for the owner
- `5` = `r-x` for the group
- `5` = `r-x` for others

To apply it:

```sh
# chmod 755 script.sh
```

### Changing Permissions with Symbolic Mode

Symbolic mode allows explicit changes to individual permission bits:

```sh
# chmod u+x script.sh
# chmod g-w file.txt
# chmod o= file.txt
```

- `u`, `g`, `o`, `a`: user, group, others, all
- `+`, `-`, `=`: add, remove, set exactly
- `r`, `w`, `x`: permission types

Use symbolic mode for fine-grained adjustments and numeric mode for explicitly setting all three categories.

---

### Special Permission Bits

In addition to the standard permission bits, OpenBSD supports three **special** modes:

| Special Bit | Octal Prefix | Symbol (ls -l) | Meaning |
| --- | --- | --- | --- |
| setuid | 4000 | `s` in user execute | Executable runs with the owner’s UID |
| setgid | 2000 | `s` in group execute | Executable runs with the group’s GID; directories inherit group |
| sticky bit | 1000 | `t` in others execute | Only file owner or root may delete files in the directory |

#### setuid Example

```sh
# chmod 4755 /usr/local/bin/somescript
```

This sets:

- `4` (setuid) + `755` → `4755`
- Owner execute becomes `s`: `-rwsr-xr-x`

Use `ls -l` to see the `s`:

```sh
$ ls -l /usr/local/bin/somescript
```

#### setgid Example

```sh
# chmod 2755 /usr/local/bin/teamcmd
```

- Group execute becomes `s`: `-rwxr-sr-x`

On directories, `setgid` causes new files to inherit the directory’s group:

```sh
# chmod 2775 /var/shared
```

#### sticky Bit Example

On shared directories like `/tmp`, the sticky bit prevents users from deleting each other’s files:

```sh
# chmod 1777 /tmp
```

Files in `/tmp` can only be deleted by their owner or by `root`, even though the directory is world-writable.

The sticky bit appears as `t`:

```sh
$ ls -ld /tmp
```

```
drwxrwxrwt  10 root  wheel  5120 Jul 11 15:10 /tmp
```

---

To combine special bits with standard permissions, prefix the numeric mode accordingly:

- `chmod 4755` → setuid + `755`
- `chmod 2755` → setgid + `755`
- `chmod 1777` → sticky + `777`

Use these bits with care: they affect security and process behavior.

## Shell Configuration

The default shell in OpenBSD is `ksh(1)` (based on the Public Domain Korn Shell).

### Shell Profiles

User shell initialization files include:

- `~/.profile`: Executed by login shells
- `~/.kshrc`: Used for interactive `ksh` sessions (referenced in `.profile`)

A minimal `.profile`:

```sh
export ENV="$HOME/.kshrc"
export PATH="/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin"
umask 022
```

## Process Management

Processes can be listed, controlled, or monitored using standard utilities.

### Viewing Processes

```sh
$ ps aux
$ top
```

- `ps aux`: Shows all running processes.
- `top`: Provides an interactive, real-time view.

### Controlling Processes

```sh
$ kill -TERM 1234    # Gracefully stop process
$ kill -KILL 1234    # Forcefully terminate
$ pkill httpd        # Terminate by name
```

## Service Management

OpenBSD services are managed using `rcctl(8)`.

### Enabling and Starting Services

```sh
# rcctl enable ntpd
# rcctl start ntpd
```

### Disabling and Stopping Services

```sh
# rcctl stop ntpd
# rcctl disable ntpd
```

### Restarting and Configuring

```sh
# rcctl restart httpd
# rcctl set httpd flags "-d"
```

To check a service’s status:

```sh
# rcctl check ntpd
```

## Installed Software and Version Info

OpenBSD includes a minimal base system. Additional software is installed using `pkg_add(1)`.

To view system version:

```sh
$ uname -a
```

Example output:

```
OpenBSD myhost.example.org 7.8 GENERIC.MP#123 amd64
```
