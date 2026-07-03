# Localization

## Synopsis

Localization refers to the configuration of system behavior to support regional language, character encoding, date formats, and input settings. While OpenBSD uses US English and POSIX standards by default, it supports localization through environment variables, UTF-8 character sets, alternate keymaps, and timezone settings.

This chapter explains how to:

- Set language and locale preferences via `LANG` and `LC_*` variables
- Enable UTF-8 character encoding in the base system
- Configure timezones with `tzsetup(8)`
- (Optionally) change the console keyboard layout using `wsconsctl(8)`

Localization is useful when editing text in other languages, processing Unicode files, or preparing systems for international use. These settings apply to the **base system** (console, shells, and text utilities) and not to X11 or graphical environments, which require additional configuration.

## Configuring Locales and UTF-8 Support

OpenBSD defaults to the `C` locale (also known as the POSIX locale), which uses US English messages and 7-bit ASCII encoding. To work with filenames, text files, or terminal input in other languages, users may enable support for UTF-8 and set locale-specific behavior using environment variables.

### Locale Environment Variables

The following environment variables control locale settings:

| Variable | Purpose |
| --- | --- |
| `LANG` | Default locale used for all categories |
| `LC_CTYPE` | Character classification and encoding |
| `LC_COLLATE` | String comparison and sorting order |
| `LC_MESSAGES` | Language for program messages |
| `LC_TIME` | Date and time formatting |
| `LC_NUMERIC` | Number formatting (e.g., decimal points) |
| `LC_ALL` | Overrides all the above (not recommended for long-term use) |

To check the current locale settings in effect for the session:

```sh
$ locale
```

This displays the current values of `LANG`, `LC_CTYPE`, and related variables.

To enable German message output and UTF-8 encoding, for example:

```sh
$ export LANG=de_DE.UTF-8
```

To set this system-wide for all users, add the variables to `/etc/profile` or configure them in `/etc/login.conf`.

### Available Locales

OpenBSD provides a set of compiled locales under `/usr/share/locale/`. These are generated during installation and include UTF-8 variants.

To list all available locales:

```sh
$ locale -a
```

To filter for UTF-8-capable locales:

```sh
$ locale -a | grep UTF-8
```

Examples include:

- `en_US.UTF-8` — US English with UTF-8
- `de_DE.UTF-8` — German
- `fr_FR.UTF-8` — French
- `ja_JP.UTF-8` — Japanese

Not all system messages are translated for every locale. If a desired locale is unavailable, it is safe to fall back to `en_US.UTF-8` for UTF-8 support with English messages.

### Using UTF-8 in the Console

To enable UTF-8 behavior in terminal sessions:

1. Set `LC_CTYPE` or `LANG` to a UTF-8 locale:

   ```sh
   $ export LC_CTYPE=en_US.UTF-8
   ```
2. Ensure that the terminal supports UTF-8 encoding. Most console terminals (`tmux`, `xterm`, `ssh`) support UTF-8 by default.
3. If needed, load a Unicode-compatible font using `wsfontload(8)` (see keyboard and display configuration).

Standard tools such as `less`, `awk`, and `grep` will then handle UTF-8 input and output correctly, assuming the locale is configured.

### Persistent Locale Configuration

To set locale variables at login, modify the user’s shell startup files or define them in `/etc/login.conf`.

#### Example: `.profile` (for `sh`, `ksh`, etc.)

```sh
export LANG=en_US.UTF-8
export LC_TIME=de_DE.UTF-8
```

#### Example: `/etc/login.conf`

Locate the appropriate login class (e.g., `default`) and set:

```conf
:lang=en_US.UTF-8:
```

Then rebuild the capability database:

```sh
# cap_mkdb /etc/login.conf
```

Locale settings in `login.conf` apply automatically for users in that login class. Per-session overrides in `.profile` or `.login` will take precedence.

## Timezone Configuration

OpenBSD stores the system timezone as a symbolic link from `/etc/localtime` to a file in the `zoneinfo` database, located under `/usr/share/zoneinfo/`. By default, systems are configured to use UTC unless the user selects a local timezone during installation.

Changing the timezone affects how the system displays time, logs events, and formats timestamps. It does not affect the hardware clock, which should remain set to UTC.

### Setting the Timezone with `tzsetup`

To configure or change the system timezone interactively:

```sh
# tzsetup
```

This command presents a menu of continents and cities. After selection, it sets `/etc/localtime` accordingly.

Example:

1. Select `Europe`
2. Select `Amsterdam`

The result will be a symbolic link:

```sh
# ls -l /etc/localtime
lrwxr-xr-x  1 root  wheel  33 Jul 29 14:21 /etc/localtime -> /usr/share/zoneinfo/Europe/Amsterdam
```

### Setting Timezone Manually

Alternatively, set the timezone without using `tzsetup`:

```sh
# ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
```

To take effect immediately, the new timezone must be readable by all users and processes. There is no need to reboot.

### Checking the Current Timezone

To display the system’s current time and time zone setting:

```sh
$ date
```

To verify the symbolic link:

```sh
$ readlink /etc/localtime
/usr/share/zoneinfo/Europe/Berlin
```

### Coordinated UTC and RTC Handling

The system clock should always be kept in UTC. OpenBSD uses the UTC assumption to avoid conflicts with daylight saving transitions and multi-boot environments.

To confirm the current time is tracked in UTC:

```sh
# sysctl kern.utc_offset
```

Hardware clocks are managed by `ntpd(8)` or can be updated manually with `date(1)` and `hwclock(8)`.

Timezone configuration affects system log timestamps (`syslog`, `dmesg`), `date`, and any application that consults the C library `localtime(3)` function.

## Console Keyboard Layouts

By default, OpenBSD uses a US English QWERTY keyboard layout in the system console. Users who prefer an alternate layout (e.g., German QWERTZ, French AZERTY) can reconfigure the console keyboard using `wsconsctl(8)`.

These settings apply only to the text console (such as `ttyC0` through `ttyC4`) and do not affect graphical environments such as X11.

### Listing Available Keymaps

Available keymaps are located in `/usr/share/wscons/keymaps/`. To list them:

```sh
$ ls /usr/share/wscons/keymaps/
```

Examples include:

- `de.nodead` — German layout without dead keys
- `fr` — French AZERTY layout
- `uk` — British QWERTY layout
- `us.colemak` — US Colemak layout

Layout names used with `wsconsctl` must match these filenames (without the `.wscons` extension).

### Setting the Keymap Temporarily

To apply a keymap immediately for the current session:

```sh
# wsconsctl keyboard.encoding=de.nodead
```

The setting takes effect at once and will persist until reboot. To confirm:

```sh
$ wsconsctl keyboard.encoding
keyboard.encoding=de.nodead
```

### Making the Setting Persistent

To apply the keyboard layout at every boot, edit `/etc/wsconsctl.conf`:

```conf
keyboard.encoding=de.nodead
```

This configuration file is processed during system startup by the `wsconsctl` service defined in `/etc/rc.d/`.

### Reverting to the Default Layout

To restore the default US English layout:

```sh
# wsconsctl keyboard.encoding=us
```

Changing the keyboard layout is useful for users with non-US physical keyboards or those working in multilingual environments. Only console sessions are affected; graphical environments manage keyboard settings separately using their own input systems.
