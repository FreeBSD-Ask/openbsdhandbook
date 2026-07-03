# System Configuration

## Synopsis

System configuration in OpenBSD relies on a small set of well-defined utilities and text files located in `/etc`. Settings can be applied at runtime using tools like `sysctl`, `wsconsctl`, and `rcctl`, or persisted across reboots in files such as `/etc/sysctl.conf`, `/etc/wsconsctl.conf`, and `/etc/rc.conf.local`. In addition, system maintenance includes managing service behavior, login environment limits, bootloader parameters, root mail notifications, and scheduled updates.

## Kernel Parameters with `sysctl`

The `sysctl` utility displays and modifies kernel state variables. These variables control aspects such as networking, security, hardware behavior, memory limits, and process constraints.

To display all values:

```sh
$ sysctl
```

To read or set a single value:

```sh
$ sysctl hw.smt
# sysctl hw.smt=0
```

To persist settings across reboots, place them in `/etc/sysctl.conf`:

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

### Common Examples

- `hw.smt=0` — Disable hyperthreading (SMT), often recommended for security.
- `net.inet.ip.forwarding=1` — Enable IP packet forwarding for routing.
- `kern.maxfiles` — Increase the number of open file descriptors.
- `vm.swapencrypt.enable=1` — Encrypt data swapped to disk.
- `kern.nosuidcoredump=1` — Prevent core dumps for set-user-ID binaries.
- `fs.posix.setuid=0` — Disable setuid behavior on POSIX semaphores.

Changes to `/etc/sysctl.conf` are applied at boot by the system startup scripts.

## Console Input and Display with `wsconsctl`

The `wsconsctl` utility modifies the behavior of the console keyboard and display. Settings can be applied at runtime or saved in `/etc/wsconsctl.conf` for persistence.

### Keyboard Layout and Behavior

To adjust the keyboard layout:

```sh
# wsconsctl keyboard.layout=us
```

Other settings include:

```sh
# wsconsctl keyboard.bell.volume=0
# wsconsctl keyboard.repeat.del1=400
# wsconsctl keyboard.repeat.deln=40
```

To persist keyboard settings, add them to `/etc/wsconsctl.conf`:

```conf
keyboard.layout=us
keyboard.bell.volume=0
keyboard.repeat.del1=400
keyboard.repeat.deln=40
```

### Console Font Configuration

Fonts may be loaded dynamically using `wsfontload`. For example:

```sh
# wsfontload -N Spleen32 -n 10 -f /usr/share/wscons/fonts/spleen32x64.fnt
# wsconsctl display.font=Spleen32
```

To load fonts at boot, place the font-loading commands in `/etc/rc.local`:

```sh
# vi /etc/rc.local
```

Add the following:

```conf
wsfontload -N Spleen32 -n 10 -f /usr/share/wscons/fonts/spleen32x64.fnt
wsconsctl display.font=Spleen32
exit 0
```

Font selection depends on screen resolution and personal preference. Fonts reside in `/usr/share/wscons/fonts`.

## Managing Services with `rcctl`

OpenBSD uses `rcctl` to enable, disable, and control services. These services are defined under `/etc/rc.d/`.

To enable a service at boot:

```sh
# rcctl enable ntpd
```

To start a service immediately:

```sh
# rcctl start ntpd
```

To check its status:

```sh
# rcctl check ntpd
```

To disable a service:

```sh
# rcctl disable ntpd
```

To restart a service:

```sh
# rcctl restart ntpd
```

To set runtime flags for a service:

```sh
# rcctl set ntpd flags "-s"
```

These settings are saved in `/etc/rc.conf.local`.

## Persistent System Configuration Files

System behavior is primarily defined by configuration files in `/etc`. Below is a summary of key files:

- `/etc/rc.conf` — Default system settings (do not edit)
- `/etc/rc.conf.local` — Local overrides for services and variables
- `/etc/hostname.if` — Interface configuration (e.g., `hostname.em0`)
- `/etc/myname` — System hostname
- `/etc/mygate` — Default route (gateway)
- `/etc/resolv.conf` — DNS resolver configuration
- `/etc/fstab` — Filesystem mounting configuration
- `/etc/doas.conf` — Permissions for privilege elevation
- `/etc/login.conf` — User login classes and limits
- `/etc/sysctl.conf` — Kernel parameters
- `/etc/wsconsctl.conf` — Console and keyboard settings
- `/etc/ttys` — Terminal and console definitions
- `/etc/mail/aliases` — Email aliasing, including for root
- `/etc/rc.local` — Startup commands

### Editing User and Group Databases

To safely edit the password and user database:

```sh
# vipw
```

This ensures updates are atomic and that related shadow files are kept in sync. For group modifications, edit `/etc/group` directly with a text editor.

### Using `/etc/examples`

When installing or enabling new components, configuration templates are often found in `/etc/examples`. For example:

```sh
# cp /etc/examples/ntpd.conf /etc/ntpd.conf
```

These defaults are maintained by the system and are safe to copy and customize.

## Login Classes and Resource Limits

The `/etc/login.conf` file defines environment and resource limits for user classes. Each user is associated with a class (e.g., `default`, `staff`, `daemon`).

Example definition:

```conf
staff:\
  :openfiles-cur=1024:\
  :stacksize-cur=8M:\
  :tc=default:
```

After editing the file, rebuild the capability database:

```sh
# cap_mkdb /etc/login.conf
```

Changes take effect at the next login.

## Bootloader Configuration

Bootloader behavior is controlled through `/etc/boot.conf`. This file can contain commands that are automatically passed to the bootloader during early system startup.

Example configuration:

```
set timeout 5
boot /bsd
```

This sets a 5-second delay and instructs the loader to boot `/bsd`. At boot time, users may interrupt the countdown and issue commands manually. Refer to `boot(8)` for boot prompt syntax.

## Root Mail and Internal Notifications

OpenBSD uses local mail delivery to notify administrators about system events. Sources of mail include:

- `cron(8)` job output
- `daily(8)`, `weekly(8)`, and `monthly(8)` reports
- Errors or messages from system daemons

To view root’s mail:

```sh
# mail
```

To forward root’s mail to an external address, edit `/etc/mail/aliases`:

```
root: admin@example.com
```

Then run:

```sh
# newaliases
```

Mail is stored in `/var/mail/root`.

## Alternate Root Partition (`/altroot`)

OpenBSD supports an alternate root partition, typically mounted at `/altroot`. It is used for:

- Hosting a backup root filesystem
- Storing system dumps
- Redundant kernel images or boot configuration

To create a backup with `dump`:

```sh
# dump -0au -f /altroot/root.dump /
```

This partition should reside on a separate physical device when possible.

## Scheduling Automatic Updates

It is recommended to automate periodic tasks such as security patching with `syspatch`. One way to do this is to add commands to `/etc/daily.local`, which is run by the daily script.

Example:

File: `/etc/daily.local`

```sh
#!/bin/sh
syspatch -c
exit 0
```

Make it executable:

```sh
# chmod +x /etc/daily.local
```

System scripts also generate logs in `/var/log/`:

- `/var/log/daily`
- `/var/log/weekly`
- `/var/log/monthly`

These logs are emailed to root. Ensure they are reviewed regularly or redirected.
