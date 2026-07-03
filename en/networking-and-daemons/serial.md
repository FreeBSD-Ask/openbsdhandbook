# Serial Communication

## Synopsis

Serial communication provides essential functionality for OpenBSD systems, particularly in scenarios involving headless servers, embedded systems, hardware debugging, and network devices. OpenBSD includes full support for configuring and using serial terminals through the base system. This includes login over serial console, connection to serial devices using tools like `cu` and `tip`, configuration of USB-to-serial adapters, and verification of serial port functionality.

This chapter explains how to configure serial login access via `/etc/ttys`, how to use the serial console at boot via `/etc/boot.conf`, how to interface with serial ports using standard base utilities, and how to test and validate serial hardware and communication settings.

## Enabling Serial Console Login

On OpenBSD, serial ports are typically available as `/dev/tty00`, `/dev/tty01`, and so on. To enable login on a serial port, modify the `/etc/ttys` file.

To configure the first serial port (`tty00`) for login:

```
tty00   "/usr/libexec/getty std.9600"   vt220   on secure
```

This entry instructs `init` to launch `getty(8)` on `/dev/tty00` using the `std.9600` entry from `gettytab(5)`. The terminal type `vt220` is appropriate for most serial terminals. The `secure` keyword allows root logins on this line.

To activate the change:

```sh
# kill -HUP 1
```

This signals the `init(8)` process to reload its configuration and spawn `getty` on the specified line.

Ensure the serial port is not already in use by another process. Use `fstat` or `ps` to confirm.

## Using cu and tip

The OpenBSD base system includes two terminal communication programs for interacting with serial devices: `cu(1)` and `tip(1)`.

### Using `cu`

The `cu` utility allows direct connection to a serial device. To open a connection with `/dev/tty00` at 9600 baud:

```sh
# cu -l /dev/tty00 -s 9600
```

To exit the session, type `~.` at the beginning of a new line.

### Using `tip`

The `tip` utility can be used interactively or with configuration entries defined in `/etc/remote`. A direct invocation without defining an alias:

```sh
# tip -9600 tty00
```

To create a reusable entry in `/etc/remote`:

```conf
console:\
	:dv=/dev/tty00:br#9600:pa=none:
```

Then connect using:

```sh
# tip console
```

The escape sequence to exit is the same: `~.` on a new line.

## Using a Serial Console at Boot

To configure OpenBSD to use a serial port for bootloader and kernel messages, edit `/etc/boot.conf`.

To enable serial console output via the first serial port (`com0`, corresponding to `/dev/tty00`):

```
set tty com0
```

This will redirect output from the bootloader and kernel to the serial port, typically at 9600 baud.

When interacting with a system through a USB-to-serial adapter from another machine, use `cu`:

```sh
# cu -l /dev/cuaU0 -s 115200
```

Interrupt the bootloader and enter:

```
set tty com0
boot
```

This will direct both boot and kernel output to the serial interface.

## USB Serial Adapters

Modern systems often lack traditional RS-232 ports. OpenBSD supports a wide variety of USB-to-serial adapters through drivers such as:

- `ucom(4)` — generic USB serial interface
- `uplcom(4)` — Prolific-based devices
- `umodem(4)` — USB modems
- `uvscom(4)` — SUNTAC serial devices

When an adapter is connected, the kernel will log its detection:

```
uplcom0 at uhub1 port 2: Prolific Technology USB-Serial Controller
ucom0 at uplcom0
```

Device nodes will appear as:

- `/dev/cuaU0` — for outgoing connections (`cu`, `tip`)
- `/dev/ttyU0` — for incoming connections (e.g., login)

To connect at 115200 baud:

```sh
# cu -l /dev/cuaU0 -s 115200
```

## Testing Serial Ports

Serial ports may be tested using various techniques.

### Checking Device Presence

Ensure the relevant device node exists:

```sh
# ls -l /dev/tty00
```

### Performing a Loopback Test

Physically connect the transmit and receive pins (e.g., pins 2 and 3 on a DB9 connector), then:

```sh
# cu -l /dev/tty00 -s 9600
```

Typing characters should cause them to echo back to the screen.

### Monitoring Port Usage

Use `fstat` to determine which process, if any, is using a serial device:

```sh
# fstat | grep tty00
```

Live activity can also be monitored using `systat`:

```sh
# systat tty
```

### Checking Kernel Messages

Inspect kernel logs for detected serial interfaces:

```
com0 at isa0 port 0x3f8/8 irq 4: ns16550a, 16 byte fifo
```

USB-based adapters will show `ucom`, `uplcom`, or similar device lines.

## Serial Parameters and Flow Control

Serial communications depend on consistent settings across both endpoints. Common parameters include:

- **Baud rate**: such as 9600, 19200, or 115200
- **Data bits**: typically 8 (`cs8`)
- **Parity**: none (`-parenb`), or `even`, `odd`
- **Stop bits**: usually 1 (`-cstopb`)
- **Flow control**: none, XON/XOFF, or RTS/CTS

To manually configure these settings:

```sh
# stty -f /dev/tty00 115200 cs8 -cstopb -parenb
```

This sets 115200 baud, 8 data bits, 1 stop bit, and no parity on `/dev/tty00`.

## Kernel Debugging over Serial

Serial communication is essential when using the OpenBSD kernel debugger (`ddb`) on headless or embedded systems, or when debugging kernel panics remotely. When a serial console is properly configured, `ddb` output and input can be redirected to the serial line.

To enable kernel debugging over serial, the system must first be configured to use a serial console. This includes:

1. **Configuring `/etc/boot.conf`**:

   To redirect console output to the first serial port (`com0`, typically `/dev/tty00`):

   ```
   set tty com0
   ```
2. **Ensuring `ddb` is enabled**:

   The default OpenBSD kernel includes `ddb`. To ensure it is accessible, confirm the following `sysctl` values:

   ```sh
   # sysctl ddb.console ddb.panic
   ddb.console=1
   ddb.panic=1
   ```

   These settings mean that:

   - `ddb.console=1`: the kernel debugger is available from the console
   - `ddb.panic=1`: the kernel will enter `ddb` on a panic

   To make these settings persistent, add the following to `/etc/sysctl.conf`:

   ```conf
   ddb.console=1
   ddb.panic=1
   ```
3. **Using the debugger from serial**:

   Once configured, when the system panics or `ddb` is manually entered (e.g., with `sysctl ddb.trigger=1`), the prompt will appear on the serial console. Commands such as `trace`, `ps`, `show registers`, and `reboot` may be entered.

   ```sh
   ddb> trace
   ddb> ps
   ddb> reboot
   ```
4. **Triggering `ddb` manually**:

   Use the following command to break into the debugger manually (for testing or debugging):

   ```sh
   # sysctl ddb.trigger=1
   ```

   This will drop the system into `ddb` from the current console. When using a serial console, the prompt will be visible over the serial line.
5. **Handling systems without local displays**:

   On systems lacking VGA or framebuffer output, such as many embedded platforms, serial console access is the only way to interact with `ddb`. For this reason, early configuration of serial console output is critical during system bring-up or platform porting.

Kernel debugging over serial allows recovery from panics, system state inspection, and automated crash logging when combined with remote serial capture. The configuration is fully supported using base system facilities without additional software.

## Using OpenBSD as a Console Server

An OpenBSD system can serve as a reliable console server, providing serial access to one or more remote devices such as switches, headless servers, firewalls, or embedded platforms. This setup is often used in data centers, labs, or remote deployments where out-of-band management is required.

### Hardware Setup

Most modern systems lack traditional serial ports but can be equipped using USB-to-serial adapters. OpenBSD supports many such adapters via drivers like `uplcom`, `umodem`, `uvscom`, and `ucom`. Multi-port PCI or PCIe serial cards (e.g., based on `ns8250`) are also supported via the `com(4)` driver.

Upon connecting a USB serial adapter, the device will be visible as `/dev/cuaU0`, `/dev/cuaU1`, etc. Kernel messages can be inspected using `dmesg`:

```
uplcom0 at uhub1 port 3: Prolific USB-Serial Controller
ucom0 at uplcom0
```

Use a proper null modem cable between the OpenBSD system and the device under management.

### Connecting to Devices

Each serial port can be accessed interactively using `cu`:

```sh
# cu -l /dev/cuaU0 -s 115200
```

To simplify access, define named entries in `/etc/remote`:

```conf
router1:\
	:dv=/dev/cuaU0:br#115200:pa=none:
switch1:\
	:dv=/dev/cuaU1:br#9600:pa=none:
```

Then connect with:

```sh
# tip router1
```

### Managing Multiple Connections

For operators managing several devices simultaneously, `tmux` can be used to provide one pane per serial session. Example:

```sh
$ tmux
$ cu -l /dev/cuaU0 -s 9600   # Pane 1
$ cu -l /dev/cuaU1 -s 115200 # Pane 2
```

Alternatively, a wrapper script or shell alias can provide quick access.

### Optional: Enabling Login on Serial Ports

If inbound access to the OpenBSD console server is required via serial, `/etc/ttys` can be modified to start a login session on each port:

```
ttyU0   "/usr/libexec/getty std.9600"   vt220   on secure
ttyU1   "/usr/libexec/getty std.115200" vt220   on secure
```

Reload the configuration:

```sh
# kill -HUP 1
```

This enables users to connect to the OpenBSD system via serial for administrative tasks.

### Security Considerations

To ensure proper isolation:

- Use `doas.conf` to restrict which users can run `cu` or `tip`
- Configure `login.conf` to limit resource usage
- Log access and commands through a restricted shell or shell wrapper
- Avoid `getty` on ports where only outbound connections are needed

Access to the console server itself should be protected by `ssh` with key-based authentication, and physical access should be limited where possible.
