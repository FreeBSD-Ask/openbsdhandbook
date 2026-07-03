# Printing

## Synopsis

OpenBSD supports both the traditional BSD printing system (`lpd`) and the more feature-rich CUPS (Common UNIX Printing System). This chapter describes how to configure local and network printers, install printer drivers and filters, print to PDF, and configure an OpenBSD system as a print server.

## Overview of Printing in OpenBSD

The base system includes `lpd(8)`, a lightweight line printer daemon that handles print spooling and basic filtering. More complex setups, such as those requiring drivers, graphical administration, or network browsing, are better handled using CUPS, which is available as a package.

Common printer types supported:

- USB printers connected via `/dev/ulpt0`
- Network printers supporting IPP, JetDirect (port 9100), or LPD
- PostScript printers that require no additional drivers
- Host-based printers (e.g., some HP, Samsung) via driver packages

Filters such as `a2ps(1)`, `enscript(1)`, and `ghostscript` are commonly used to prepare print jobs for printers that do not support plain text or PostScript natively.

## Configuring Printing with lpd

The `lpd(8)` daemon is the traditional Berkeley line printer spooler included in the OpenBSD base system. It is simple, reliable, and suitable for local or network printing where advanced features such as automatic driver detection or graphical configuration are not required. Printers must be configured manually in `/etc/printcap`, and format conversion (e.g., plain text to PostScript) is handled by user-supplied filters. `lpd` is lightweight and often preferred for small or dedicated systems.

### Enabling lpd

Enable the base printing system using `rcctl`:

```sh
# rcctl enable lpd
# rcctl start lpd
```

Ensure the printing device (such as `/dev/ulpt0`) is writable by the `daemon` user:

```sh
# chown daemon /dev/ulpt0
# chmod 600 /dev/ulpt0
```

### Configuring /etc/printcap

The `printcap(5)` file describes available printers.

Example for a local USB printer:

```conf
lp|Local USB Printer:\
    :lp=/dev/ulpt0:\
    :sd=/var/spool/output/lpd:\
    :lf=/var/log/lpd-errs:
```

Create and prepare the spool directory:

```sh
# mkdir -p /var/spool/output/lpd
# chown daemon:daemon /var/spool/output/lpd
```

To print to a remote LPD server:

```conf
remotehp:\
    :rm=192.0.2.10:\
    :rp=raw:\
    :sd=/var/spool/output/remotehp:\
    :lf=/var/log/lpd-errs:
```

### Adding Print Filters

If the printer does not support plain text, a print filter is required using the `if=` field.

Install `enscript` and `ghostscript`:

```sh
# pkg_add enscript ghostscript
```

Create a filter script:

```sh
# vi /usr/local/libexec/psfilter
```

Example content:

```sh
#!/bin/sh
exec enscript -o - | gs -q -dNOPAUSE -dBATCH -sDEVICE=ijs -sOutputFile=- - 2>/dev/null
```

This pipeline works as follows:

- `enscript -o -` converts plain text to PostScript and sends it to standard output.
- `gs` (Ghostscript) reads the PostScript input, rasterizes it using the ijs driver, and sends the output to standard output.
- `-q,` `-dNOPAUSE`, and `-dBATCH` suppress prompts and exit after processing.
- `-sDEVICE=ijs` selects the IJS raster output interface; alternatives include pxlmono, ljet4, and uniprint, depending on the printer.
- `-sOutputFile=-` writes to standard output; 2>/dev/null suppresses errors.

This allows lpd to support a wide range of printers using external filters. Run gs -h to list supported output devices.

Make it executable:

```sh
# chmod +x /usr/local/libexec/psfilter
```

Update `/etc/printcap`:

```conf
lp:\
    :lp=/dev/ulpt0:\
    :if=/usr/local/libexec/psfilter:\
    :sd=/var/spool/output/lpd:\
    :lf=/var/log/lpd-errs:
```

### Printing from the Command Line

Send a job:

```sh
$ lpr file.txt
```

Check the queue:

```sh
$ lpq
```

Remove jobs:

```sh
$ lprm -
```

## Printing with CUPS

CUPS (Common UNIX Printing System) is a modular printing system that supports a wide range of printer protocols, drivers, and administrative tools. Unlike `lpd`, CUPS provides a web-based interface, automatic printer discovery, and integration with many modern printing workflows. While not part of the OpenBSD base system, it is available as a package and is recommended when using USB inkjet printers, printers requiring PPD files, or complex network printing environments.

### Installing and Enabling CUPS

Install CUPS and supporting drivers:

```sh
# pkg_add cups gutenprint foomatic-db splix hplip
```

Enable and start the daemon:

```sh
# rcctl enable cupsd
# rcctl start cupsd
```

### Accessing the Web Interface

The CUPS web interface is accessible at:

```
http://localhost:631
```

Access may require adding the user to the `operator` group and creating a `doas` rule:

```
permit persist :operator as root cmd /usr/local/sbin/lpadmin
```

Printers can be added via the web interface or with `lpadmin`.

### Driver and Backend Support

Check backends:

```sh
$ lpinfo -v
```

List available models and drivers:

```sh
$ lpinfo -m
```

CUPS backends support protocols such as:

- `socket://` — JetDirect
- `ipp://` or `http://` — IPP printers
- `usb://` — for local printers

### Printing with CUPS

To print:

```sh
$ lp -d PRINTER file.pdf
```

Set the default printer:

```sh
$ lpoptions -d PRINTER
```

View jobs:

```sh
$ lpstat -o
```

Cancel a job:

```sh
$ cancel JOBID
```

## Setting Up an OpenBSD Print Server

### lpd-Based Print Server

Enable `lpd` and configure `/etc/printcap` with exported printers. Add trusted client hosts to `/etc/hosts.lpd`:

```
192.0.2.15
client.example.org
```

Client usage:

```sh
$ lpr -Pprinter@server file.txt
```

### CUPS-Based Print Server

Edit `/etc/cups/cupsd.conf`:

```
Listen *:631
Browsing On
BrowseLocalProtocols all
<Location />
  Order allow,deny
  Allow all
</Location>
```

Restart the daemon:

```sh
# rcctl restart cupsd
```

On client systems, configure:

```text
/etc/cups/client.conf
```

with:

```
ServerName printserver.example.net
```

Clients may then use `lp`, `lpr`, or print dialogs without further configuration.

## Printing to PDF

### From Graphical Applications

Most applications that support GTK, Qt, or X11 printing dialogs allow printing to file. Select “Print to File” or “Print to PDF” and choose a destination.

### Using Command-Line Tools

Convert text to PDF using `enscript` and `ghostscript`:

```sh
$ enscript file.txt -p - | ps2pdf - output.pdf
```

Or use PostScript directly:

```sh
$ ps2pdf input.ps output.pdf
```

### Virtual PDF Printer with CUPS

Install the virtual backend:

```sh
# pkg_add cups-pdf
```

Add a new printer called “PDF” in the CUPS web interface. The output will be written to:

```text
/var/spool/cups-pdf/USERNAME
```

The output directory can be changed in `/etc/cups/cups-pdf.conf`.

## Remote Printing

### To a Remote lpd Server

Use `/etc/printcap`:

```conf
remote:\
    :rm=printserver.example.net:\
    :rp=lp:\
    :sd=/var/spool/output/remote:\
    :lf=/var/log/lpd-errs:
```

### To a Remote CUPS Server

Create a configuration file:

```text
/etc/cups/client.conf
```

With contents:

```
ServerName cups.example.net
```

Alternatively, set the environment variable:

```sh
$ export CUPS_SERVER=cups.example.net
```

### Printing to Windows Shared Printers

Install `samba`:

```sh
# pkg_add samba
```

Use a device URI like:

```
smb://WORKGROUP/username:password@host/sharename
```

CUPS can use `smbspool` as a backend for SMB printing. Ensure credentials are embedded if printing unattended.

## Troubleshooting

- For `lpd`, check `/var/log/lpd-errs`
- For CUPS, use:

```sh
$ tail -f /var/log/cups/error_log
```

- Ensure printer device permissions:

```sh
# ls -l /dev/ulpt0
```

- Restart the service after changes:

```sh
# rcctl restart lpd
# rcctl restart cupsd
```

- For failed jobs, verify that the correct driver or filter is installed
- Check if spool directories exist and are writable by `daemon` (for `lpd`) or the CUPS user
