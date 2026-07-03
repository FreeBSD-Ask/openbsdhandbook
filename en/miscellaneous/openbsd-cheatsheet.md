# OpenBSD Cheatsheet

This cheatsheet summarizes common OpenBSD administration tasks with concise commands. Commands use `$` for the unprivileged user and `#` for the superuser. Use [doas(1)](https://man.openbsdhandbook.com/doas.1/)
to escalate privileges and configure it via [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/)
.

## Privilege escalation

First-time setup from the documented example:

```sh
$ doas cp /etc/examples/doas.conf /etc/doas.conf  # seed config from example
```

Grant cached auth for wheel:

```
permit persist keepenv :wheel
```

Test:

```sh
$ doas id  # run a simple command via doas
```

## Packages

Package tools: [pkg\_add(1)](https://man.openbsdhandbook.com/pkg_add.1/)
, [pkg\_delete(1)](https://man.openbsdhandbook.com/pkg_delete.1/)
, [pkg\_info(1)](https://man.openbsdhandbook.com/pkg_info.1/)
, overview in [packages(7)](https://man.openbsdhandbook.com/packages.7/)
.

```sh
$ doas pkg_add htop            # install package
$ doas pkg_delete htop         # remove package
$ pkg_info | less              # list installed packages
$ doas pkg_add -Uu             # upgrade all packages for current release
```

Package mirrors are configured with [installurl(5)](https://man.openbsdhandbook.com/installurl.5/)
.

## Base updates and release upgrades

Base security/bug patches: [syspatch(8)](https://man.openbsdhandbook.com/syspatch.8/)
.
Release upgrades: [sysupgrade(8)](https://man.openbsdhandbook.com/sysupgrade.8/)
.

```sh
$ doas syspatch -c  # list pending base patches
$ doas syspatch     # apply base patches (reboot if asked)

$ doas sysupgrade   # fetch and upgrade to next release
$ doas pkg_add -Uu  # upgrade packages after release upgrade
```

## Services (rc)

Service management: [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
. System startup: [rc(8)](https://man.openbsdhandbook.com/rc.8/)
, [rc.conf(8)](https://man.openbsdhandbook.com/rc.conf.8/)
.

```sh
$ doas rcctl check sshd      # show status
$ doas rcctl start sshd       # start now
$ doas rcctl stop sshd        # stop now
$ doas rcctl reload sshd      # reload config
$ doas rcctl enable sshd      # enable at boot
$ doas rcctl disable sshd     # disable at boot
$ doas rcctl ls on            # list enabled services
$ doas rcctl get sshd flags   # show daemon flags
$ doas rcctl set sshd flags "-v"  # set daemon flags persistently
```

## Networking

Interfaces: [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
. Persistent config files: [hostname.if(5)](https://man.openbsdhandbook.com/hostname.if.5/)
, [myname(5)](https://man.openbsdhandbook.com/myname.5/)
, [mygate(5)](https://man.openbsdhandbook.com/mygate.5/)
, [resolv.conf(5)](https://man.openbsdhandbook.com/resolv.conf.5/)
. Apply with [netstart(8)](https://man.openbsdhandbook.com/netstart.8/)
.

### Inspect and temporary changes

```sh
$ ifconfig -A                 # all interfaces
$ doas ifconfig em0 10.0.0.10 255.255.255.0  # temp IPv4
$ doas ifconfig em0 inet6 2001:db8::10 64    # temp IPv6
$ doas sh /etc/netstart em0   # re-load a single interface
$ doas sh /etc/netstart       # re-load all from files
```

### Persistent examples

`/etc/hostname.em0` (choose one of the following forms):

```
dhcp
```
```
inet 192.0.2.10 255.255.255.0
```
```
inet6 2001:db8:6000:1::10 64
```

Hostname, default gateway, and resolvers:

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

### Useful network diagnostics

```sh
$ ping -n 1.1.1.1            # ICMP test (no DNS)
$ traceroute -n 1.1.1.1      # path without DNS
$ netstat -rn                # routing table
$ route show                 # routing sockets view
$ tcpdump -ni em0 port 53    # capture DNS on em0
```

## Firewall (PF)

Packet filter tools: [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
. Configuration: [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
.

### Quick operations

```sh
$ doas pfctl -sr           # show rules
$ doas pfctl -sn           # show NAT
$ doas pfctl -si           # show stats
$ doas pfctl -e            # enable PF
$ doas pfctl -d            # disable PF
$ doas pfctl -f /etc/pf.conf  # reload ruleset
```

### Minimal allow-out ruleset

```pf
# /etc/pf.conf
set block-policy drop
set skip on lo

block all
pass out inet proto { tcp udp icmp } from (egress) to any modulate state
pass out inet6 proto { tcp udp icmp6 } from (egress) to any modulate state
```

Activate:

```sh
$ doas pfctl -f /etc/pf.conf
$ doas pfctl -e
```

## Web and SSH daemons

HTTP server: [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
with [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)
.
Secure shell: [sshd(8)](https://man.openbsdhandbook.com/sshd.8/)
, [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)
.

```sh
$ doas rcctl enable httpd
$ doas rcctl start httpd
$ doas rcctl enable sshd
$ doas rcctl start sshd
```

## Users and groups

User and group management: [useradd(8)](https://man.openbsdhandbook.com/useradd.8/)
, [usermod(8)](https://man.openbsdhandbook.com/usermod.8/)
, [userdel(8)](https://man.openbsdhandbook.com/userdel.8/)
, [groupadd(8)](https://man.openbsdhandbook.com/groupadd.8/)
, [passwd(1)](https://man.openbsdhandbook.com/passwd.1/)
, chsh(1).

```sh
$ doas useradd -m -G wheel -s /bin/ksh alice  # add admin user
$ doas passwd alice                            # set password
$ doas usermod -G wheel,staff alice            # adjust groups
$ doas userdel -r bob                          # remove user and home
$ chsh -s /bin/ksh                             # change login shell (self)
```

## Filesystems and storage

Filesystems: [fstab(5)](https://man.openbsdhandbook.com/fstab.5/)
, [mount(8)](https://man.openbsdhandbook.com/mount.8/)
, [umount(8)](https://man.openbsdhandbook.com/umount.8/)
, [df(1)](https://man.openbsdhandbook.com/df.1/)
. Disk setup: [fdisk(8)](https://man.openbsdhandbook.com/fdisk.8/)
(MBR/GPT), [disklabel(8)](https://man.openbsdhandbook.com/disklabel.8/)
.

```sh
$ df -h                          # usage
$ mount                          # mounted filesystems
$ doas mount /home               # mount by fstab entry
$ doas umount /home              # unmount
$ dmesg | egrep '^sd|^wd|^cd'    # list disks as seen at boot
$ doas disklabel -E sd0          # edit OpenBSD disklabel
```

## Time and ntpd

Network time: [ntpd(8)](https://man.openbsdhandbook.com/ntpd.8/)
. Configuration: [ntpd.conf(5)](https://man.openbsdhandbook.com/ntpd.conf.5/)
.

```sh
$ doas rcctl enable ntpd
$ doas rcctl start ntpd
$ rcctl check ntpd
```

## Logging and rotation

Syslog: [syslogd(8)](https://man.openbsdhandbook.com/syslogd.8/)
, configuration via [syslog.conf(5)](https://man.openbsdhandbook.com/syslog.conf.5/)
. Rotation: [newsyslog(8)](https://man.openbsdhandbook.com/newsyslog.8/)
.

```sh
$ doas tail -f /var/log/messages       # main system log
$ doas tail -f /var/log/daemon         # daemon log
$ doas newsyslog -n                    # show what would rotate
$ doas newsyslog                       # rotate now
```

## System information and diagnostics

```sh
$ dmesg | less               # kernel messages and hardware probe
$ sysctl kern.version        # kernel version
$ sysctl hw.model hw.ncpu    # hardware info
$ top                        # interactive process/CPU view
$ vmstat 1                   # system counters
$ systat iostat              # per-device I/O (curses)
$ systat ifstat              # per-interface traffic (curses)
$ ps auxww | less            # process list
$ pgrep httpd; pkill httpd   # find/terminate processes
```

## OpenSSH client basics

Client: [ssh(1)](https://man.openbsdhandbook.com/ssh.1/)
, keys via [ssh-keygen(1)](https://man.openbsdhandbook.com/ssh-keygen.1/)
, config via [ssh\_config(5)](https://man.openbsdhandbook.com/ssh_config.5/)
.

```sh
$ ssh -o StrictHostKeyChecking=accept-new user@host
$ ssh-keygen -t ed25519 -C "me@host"     # generate key
$ ssh-copy-id user@host                  # if installed; else append ~/.ssh/id_ed25519.pub on server
```

## Handy paths

```
/etc/rc.conf.local   # local daemon flags and enables
/etc/installurl      # package/upgrade mirror
/var/db/pkg/         # installed package metadata
/var/log/            # system logs
/etc/pf.conf         # PF ruleset
/etc/hostname.*      # per-interface network config
```

For deeper coverage, consult the referenced manual pages in each section.
