# Virtualization

Virtualization is a technique that allows for the creation of isolated environments, called virtual machines (VMs), where software can run as if it were operating on a physical machine. This enables the sharing of physical resources among multiple virtual instances, each behaving as an independent system. OpenBSD provides a lightweight and secure approach to virtualization through the **vmm(4)** subsystem.

## OpenBSD’s Approach to Virtualization

OpenBSD’s native virtualization support is centered around its **vmm(4)** driver, also known as “Virtual Machine Monitor.” This subsystem allows for the creation, management, and execution of virtual machines directly within OpenBSD. First introduced in OpenBSD 5.9, it is continually improved with each release. The design of OpenBSD’s virtualization emphasizes security.

Virtual machines on OpenBSD can be used to run instances of OpenBSD itself or other operating systems, provided that they are compatible with the hardware and virtualization requirements of **vmm(4)**.

### Key Components

1. **vmm(4) - Virtual Machine Monitor**: This kernel driver handles the core virtualization functionalities in OpenBSD. It manages the creation and control of virtual machines and interacts with the host system to allocate resources to the virtual machines.
2. **vmd(8) - Virtual Machine Daemon**: **vmd** is the userland counterpart to **vmm(4)**. It provides the necessary control mechanisms to create, start, stop, and manage virtual machines. The daemon can be configured through a simple configuration file, making it easier to maintain.
3. **vmctl(8) - Virtual Machine Control Tool**: **vmctl** is the primary interface for interacting with the virtualization system. It allows administrators to manage VMs, create new virtual machine images, assign resources such as CPU and memory, and inspect the state of existing virtual machines.

### Hardware Virtualization Support

OpenBSD requires hardware-assisted virtualization features to run virtual machines efficiently. These features are available on modern Intel and AMD processors and are typically enabled through BIOS or UEFI settings. The required features are:

- **Intel VT-x**: Intel’s hardware virtualization technology.
- **AMD SVM (Secure Virtual Machine)**: AMD’s equivalent of Intel VT-x.

Without these features, OpenBSD’s virtualization will not be supported, as it relies on the hardware to provide the necessary virtualization extensions.

#### Checking CPU Support for Virtualization

To check whether a system’s CPU supports hardware-assisted virtualization, the following command can be used:

```sh
$ dmesg | egrep '(VMX/EPT|SVM/RVI)'
```

- **VMX/EPT** indicates support for Intel’s hardware-assisted virtualization.
- **SVM/RVI** indicates support for AMD’s hardware-assisted virtualization.

If the output includes these flags, the system supports the necessary hardware virtualization technology.

### Key Files

OpenBSD’s virtualization system involves several important files that are used for configuring and managing virtual machines. These files handle everything from defining virtual machine settings to managing network interfaces and routing traffic.

#### /etc/vm.conf

**vm.conf** is the primary configuration file for the **vmd(8)** daemon. This file allows for the persistent configuration of virtual machines, including their memory, CPU, disk, and network settings. It can also define virtual network switches that allow virtual machines to connect to the host’s network or to other virtual machines.

For setting up **bridged networking**, see the [Bridged Networking section](#bridged-networking)
for more details.

Example entry in **vm.conf**:

```
switch "uplink" {
    interface bridge0
}

vm "vm1" {
    memory 1G
    disk "/path/to/disk.img"
    interface "uplink"
}
```

Changes to this file take effect after restarting or reloading **vmd** using `rcctl(8)` commands.

#### /var/run/vmd.sock

This is the control socket that **vmctl(8)** uses to communicate with the **vmd** daemon. It acts as the connection point for all commands sent to **vmd**, such as starting, stopping, or modifying virtual machines. Users typically do not interact with this file directly, but it is essential for managing virtual machines.

#### Virtual Disk Images

Virtual machine disk images are typically stored in user-specified directories, such as `/var/vm/`. These images represent the virtual machine’s hard drive and can be created using **vmctl(8)** or external tools. Disk images are referenced in **vm.conf** or specified directly when starting virtual machines with **vmctl**.

Example of creating a disk image:

```
# vmctl create /var/vm/disk.img -s 40G
```

#### /etc/hostname.if

This file configures network interfaces on the host, including bridge interfaces that are used for virtual machine networking. For **bridged networking** setups, a bridge interface like **bridge0** needs to be defined here to link the host’s physical network interface with the virtual machines. Refer to the [Bridged Networking section](#bridged-networking)
for more information.

Example of **hostname.bridge0**:

```
add em0
up
```

In this example, `bridge0` is configured to include the physical interface `em0`.

#### /etc/pf.conf

When using **NAT (Network Address Translation)** for virtual machines, **pf(4)**, OpenBSD’s packet filter, must be configured to allow the translation of virtual machine network traffic. The **pf.conf** file contains the rules that handle this traffic translation.

For more details on setting up NAT, refer to the [NAT Networking section](#nat-network-address-translation-networking)
.

Example of a NAT rule in **pf.conf**:

```pf
ext_if="em0"
nat on $ext_if from 10.0.0.0/24 to any -> ($ext_if)
```

In this case, traffic from the virtual machines (in the `10.0.0.0/24` subnet) will be translated to the host’s external IP address for outgoing network communication.

## Running a Virtual Machine

### Preparing to Run Virtual Machines

Once the **vmm(4)** subsystem is enabled and properly configured, a virtual machine can be created and run on OpenBSD. This section outlines the necessary steps for enabling the virtual machine monitor, preparing virtual disk images, and starting a basic virtual machine.

#### 1. Enabling and Starting the Virtual Machine Monitor (vmm)

Before a virtual machine can be run, the **vmd(8)** daemon, which manages virtual machines, must be enabled and started. To ensure that **vmm(4)** is active on system boot, the following configuration must be added:

##### Enable vmd on Boot

Edit the `/etc/rc.conf.local` file to include the following line, which tells OpenBSD to automatically start **vmd** at boot:

```conf
vmd_flags=""
```

If the file does not exist, create it with this entry. This will ensure that the system starts **vmd** with no special flags.

##### Start vmd Manually

To manually start **vmd** without rebooting, the following command is used:

```sh
# rcctl start vmd
```

This will launch the **vmd** daemon and enable the system to create and manage virtual machines. To ensure **vmd** starts automatically during subsequent reboots, use:

```sh
# rcctl enable vmd
```

#### 2. Verifying vmd is Running

To verify that **vmd** is running, use the following command:

```sh
# rcctl check vmd
```

This command will display the status of the **vmd** daemon, indicating whether it is currently active.

### Creating and Running a Virtual Machine

Once **vmd** is running, the next step is to create a virtual machine. This requires creating a virtual disk image, obtaining an installation ISO, and starting the virtual machine.

#### 1. Creating a Virtual Disk Image

A virtual disk image acts as the virtual machine’s hard drive. The **vmctl(8)** tool can be used to create a virtual disk image of a specified size. For example, to create a 40 GB disk image:

```
# vmctl create -s 40G disk.img
```

This command creates a file named `disk.img` that will be used as the virtual machine’s storage.

#### 2. Obtaining an Installation ISO

To install an operating system in the virtual machine, an installation ISO is required. For example, to install OpenBSD, the installation ISO can be downloaded using the **ftp** utility:

```sh
$ ftp https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/install78.iso
```

This command downloads the OpenBSD 7.8 installation ISO. This ISO file will be used to boot the virtual machine and install the operating system.

#### 3. Starting the Virtual Machine

Once the virtual disk image and the installation ISO are ready, the virtual machine can be started using **vmctl(8)**. The following command starts a virtual machine and connects to its console for direct interaction:

```
# vmctl start -c -m 1G -L -r install78.iso -d disk.img "vm1"
```

- `-c`: Connects to the console for interactive use.
- `-m 1G`: Allocates 1 GB of memory to the virtual machine.
- `-L`: Add a local network interface.
- `-r install78.iso`: Specifies the installation ISO to boot from.
- `-d disk.img`: Specifies the virtual disk image created earlier.
- `"vm1"`: The name of the virtual machine.

Once started, the virtual machine will boot from the ISO, allowing for the installation of an operating system in the virtual environment. During this process, the user interacts with the VM via the console.

\*\*Console Redirection Required\*\*
When prompted during installation, choose to redirect the console to \*\*com0\*\*, or a login prompt will not appear on the virtual console. SSH login will still be available after network configuration.

**Network Interfaces After First Boot**
After the first boot of the virtual machine, the host system will have a **tap** interface corresponding to the virtual machine, and the guest system will have a **vio0** interface for its network access.

**Host Tap Interface (tap0):**

```conf
tap0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> mtu 1500
  lladdr fe:e1:ba:d4:3e:e2
  description: vm1-if0-vm1
  index 10 priority 0 llprio 3
  groups: tap
  status: active
  inet 100.64.1.2 netmask 0xfffffffe
```

The **tap0** interface on the host corresponds to the virtual machine’s network connection. It has a link-local address and operates as a bridge between the host and guest networks.

**Guest Network Interface (vio0):**

```conf
vio0: flags=808b43<UP,BROADCAST,RUNNING,PROMISC,ALLMULTI,SIMPLEX,MULTICAST,AUTOCONF4> mtu 1500
  lladdr fe:e1:bb:d1:1e:fb
  index 1 priority 0 llprio 3
  groups: egress
  media: Ethernet autoselect
  status: active
  inet 100.64.1.3 netmask 0xfffffffe
```

The **vio0** interface inside the guest provides network connectivity and is automatically configured. The guest’s IP address is in the same subnet as the tap interface on the host, enabling communication between them.

#### 4. Managing the Virtual Machine

After the operating system installation is complete, the virtual machine can be stopped and started as needed. To stop the VM, use the following command:

```
# vmctl stop "vm1"
```

This halts the virtual machine. To restart the VM later without the installation ISO:

```
# vmctl start -c -m 1G -d disk.img "vm1"
```

The virtual machine will boot from the virtual disk rather than the ISO, loading the operating system that was installed during the initial setup.

#### 5. Checking the Status of Virtual Machines

To check the status of running virtual machines, use **vmctl(8)**:

```
# vmctl status
```

This command provides information on all active virtual machines, including their names, IDs, allocated memory, and current state.

### Connecting to a Virtual Machine with vmctl

Once a virtual machine is running, it can be accessed via the console using **vmctl(8)**. The console provides direct interaction with the virtual machine, functioning similarly to a physical machine’s terminal.

#### Connecting to the Console When Starting the VM

The `-c` flag can be used when starting a virtual machine to automatically open the console and establish direct interaction:

```
# vmctl start -c -m 1G -d disk.img "vm1"
```

The `-c` flag opens the console, allowing interaction with the virtual machine immediately after it starts. This is useful for installing or configuring the operating system.

#### Connecting to an Already Running VM

If the virtual machine is already running but the console was not opened during startup, it is still possible to connect using **vmctl(8)**.

First, list all active virtual machines:

```
# vmctl status
```

The command will display running virtual machines and their associated IDs. For example:

```
   ID   PID VCPUS  MAXMEM  CURMEM     TTY        OWNER    STATE NAME
    1 85075     1    1.0G   1006M   ttyp8         root  running vm1
```

To connect to the console of a specific virtual machine, use its ID from the **vmctl status** output. For instance, if the virtual machine’s ID is `1`:

```
# vmctl console 1
```

This command opens the console for the virtual machine with ID `1`, allowing direct interaction with the virtual machine’s terminal.

#### Exiting the Console

To exit the console without stopping the virtual machine, press `Enter` followed by `~.` (tilde followed by a period):

```
Enter ~.
```

This key combination disconnects the console session while leaving the virtual machine running in the background.

#### Using SSH to Connect (After OS Installation)

After the operating system has been installed in the virtual machine and network settings configured, it is possible to connect to the virtual machine via **SSH**, provided network access is available.

To connect via SSH, use the virtual machine’s IP address:

```sh
$ ssh user@<vm_ip_address>
```

Replace `<vm_ip_address>` with the IP address assigned to the virtual machine, and `user` with the appropriate username.

## Making VM Configuration Permanent

To ensure that a virtual machine configuration is persistent across reboots, the settings can be added to the **/etc/vm.conf** file. This configuration allows **vmd(8)** to automatically manage the virtual machine and ensures it starts at boot.

### Configure the Virtual Machine

Open or create the **/etc/vm.conf** file:

```
# vi /etc/vm.conf
```

Add the configuration for the virtual machine, specifying the memory, disk image, and interface configuration. For example, to create a virtual machine named `vm1` with 1 GB of memory, a disk image, and automatic startup, the configuration would look like this:

```
vm "vm1" {
    memory 1G
    enable
    disk /var/vm/disk.img
    local interface
}
```

In this example:

- `vm "vm1"` defines the virtual machine’s name.
- `memory 1G` allocates 1 GB of memory to the virtual machine.
- `enable` ensures that this virtual machine will automatically start when **vmd(8)** is launched during system boot.
- `disk /var/vm/disk.img` specifies the path to the virtual machine’s disk image.
- `local interface` creates a virtual network interface that allows the virtual machine to communicate with the host using a local-only network connection.

After (re)starting the **vmd** daemon with **/etc/vm.conf** in place, running `vmctl status` will display the virtual machine, regardless of its state. For example:

```
   ID   PID VCPUS  MAXMEM  CURMEM     TTY        OWNER    STATE NAME
    1     -     1    1.0G       -       -         root  stopped vm1
```

### Managing the Local Interface

When specifying `local interface` in the configuration, **vmd(8)** will automatically create a **tap** interface on the host and a corresponding **vio0** interface in the guest. This local-only interface allows communication between the host and the virtual machine without connecting to external networks.

- The **tap** interface on the host (e.g., `tap0`) will look like this:

  ```conf
  tap0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> mtu 1500
    lladdr fe:e1:ba:d4:3e:e2
    description: vm1-if0-vm1
    index 10 priority 0 llprio 3
    groups: tap
    status: active
    inet 100.64.1.2 netmask 0xfffffffe
  ```
- The **vio0** interface inside the guest will be configured as follows:

  ```conf
  vio0: flags=808b43<UP,BROADCAST,RUNNING,PROMISC,ALLMULTI,SIMPLEX,MULTICAST,AUTOCONF4> mtu 1500
    lladdr fe:e1:bb:d1:1e:fb
    index 1 priority 0 llprio 3
    groups: egress
    media: Ethernet autoselect
    status: active
    inet 100.64.1.3 netmask 0xfffffffe
  ```

This setup allows for local communication between the guest and host without requiring external network access.

### Ensure vmd Starts at Boot

To ensure that **vmd(8)** and the configured virtual machine start automatically at boot, enabling **vmd** with **rcctl** is necessary:

```sh
# rcctl enable vmd
```

To start **vmd** immediately:

```sh
# rcctl start vmd
```

### Reload vmd Configuration

Reload the **vmd(8)** configuration to apply the changes:

```sh
# rcctl reload vmd
```

This command will instruct **vmd** to load the updated configuration from **/etc/vm.conf** without restarting the service.

## Networking Options

OpenBSD’s virtualization subsystem, **vmm(4)**, supports several networking configurations that allow virtual machines to communicate with the host system and external networks. Virtual machines can be connected to physical interfaces, isolated, or networked in a private environment. The **vmd(8)** daemon handles the network setup for virtual machines, and various options are available depending on the desired networking behavior.

### Bridged Networking

Bridged networking allows virtual machines to appear as regular devices on the local network, effectively sharing the host machine’s physical network interface. In this mode, virtual machines can obtain their own IP addresses (e.g., via DHCP) from the network’s router or DHCP server, and communicate with other machines on the network as if they were independent systems.

#### Setting Up Bridged Networking

A bridge interface must be created to link the host’s network interface with the virtual machine’s virtual interface. The bridge aggregates the virtual machine’s network traffic with the host’s traffic. The following commands demonstrate how to create and configure a bridge:

```sh
# ifconfig bridge0 create
# ifconfig bridge0 add em0
```

Here, `em0` is the host’s physical network interface. After creating the bridge, virtual machines can be connected to it by assigning their virtual network interfaces to the bridge.

In the **vm.conf(5)** configuration file, specify the bridge interface as part of the virtual machine configuration:

```
switch "uplink" {
    interface bridge0
}

vm "vm1" {
    memory 1G
    disk "/path/to/disk.img"
    interface { switch "uplink" }
}
```

When the VM is started, its network traffic will pass through the `bridge0` interface, enabling the virtual machine to communicate on the same network as the host system.

#### Making Bridge Configuration Permanent

To make the **bridge0** configuration permanent in OpenBSD, the configuration must be saved in the appropriate interface configuration file under **/etc/hostname.bridge0**. This will ensure that the bridge is automatically created and configured on boot without needing to manually run the `ifconfig` commands each time.

1. **Create the hostname file for bridge0:**

   Open or create the **/etc/hostname.bridge0** file. This file will store the configuration that automatically sets up **bridge0** at boot.

   ```
   # vi /etc/hostname.bridge0
   ```
2. **Add the necessary configuration:**

   Add the following lines to **/etc/hostname.bridge0** to permanently create the bridge and add the physical interface (e.g., `em0`):

   ```
   add em0
   up
   ```
   - `add em0`: Adds the physical interface `em0` to the bridge.
   - `up`: Ensures that the bridge interface is activated on boot.
3. **Configure the physical interface (optional):**

   If any specific configuration is required for the **em0** interface, this can be defined in **/etc/hostname.em0**. For example, to set `em0` to come up at boot with no specific IP configuration:

   ```
   up
   ```

   This ensures that **em0** is up and ready for the bridge.
4. **Apply the changes:**

   To test the configuration without rebooting, run the following commands to manually bring up the bridge and apply the new configuration:

   ```
   # sh /etc/netstart bridge0
   ```

   This will apply the configuration in **/etc/hostname.bridge0** and bring up **bridge0** with the `em0` interface added.

##### Verifying the Bridge

To verify that **bridge0** is active and properly configured, run:

```sh
# ifconfig bridge0
```

The output should show that **bridge0** is up and includes the **em0** interface.

### NAT (Network Address Translation) Networking

With Network Address Translation (NAT), virtual machines share the host system’s IP address for outbound network communication. This mode is useful for scenarios where virtual machines require Internet access, but the network has limited IP addresses, or the virtual machine should be hidden from the external network.

#### Setting Up NAT Networking

In this configuration, the host system acts as a router, forwarding traffic from the virtual machine to the external network. **pf(4)** (Packet Filter) is used to set up NAT. Ensure that **pf** is enabled and configured to handle NAT:

##### Enabling pf

**pf** is enabled by default in OpenBSD, but if it is not running, it can be started with the following command:

```sh
# pfctl -e
```

##### Set Up NAT Rules in /etc/pf.conf

Add the following NAT rules to the **/etc/pf.conf** file to enable NAT for the virtual machine:

```pf
# Set up NAT for virtual machines
ext_if="em0"  # External interface
match out on $ext_if from 100.64.0.0/10 to any nat-to $ext_if
pass out on $ext_if from 100.64.0.0/10 to any nat-to $ext_if
```

In this example:

- `ext_if="em0"`: `em0` is the external interface of the host system.
- The virtual machines will use the `100.64.0.0/10` range, and the host will translate their traffic to its external IP address.

##### Test the pf Configuration

Before applying the changes, the **pf.conf** file can be tested for syntax correctness:

```sh
# pfctl -n -f /etc/pf.conf
```

##### Load the pf Configuration

To apply the changes, load the updated **/etc/pf.conf** configuration:

```sh
# pfctl -f /etc/pf.conf
```

##### Enable Packet Forwarding

The host system must act as a router by enabling packet forwarding. This can be done with the following command:

```sh
# sysctl net.inet.ip.forwarding=1
```

To make this configuration permanent, add the following line to **/etc/sysctl.conf**:

```conf
net.inet.ip.forwarding=1        # Permit forwarding (routing) of IPv4
```

##### Configure /etc/vm.conf for the Virtual Machine

Next, configure **/etc/vm.conf** to create a local interface for the virtual machine:

```
vm "vm1" {
    memory 1G
    disk "/var/vm/disk.img"
    local interface
}
```

This configuration defines a virtual machine named `vm1` with 1 GB of memory, a disk image located at `/var/vm/disk.img`, and a local network interface.

##### Starting the Virtual Machine

When the virtual machine is (re)started, it will use the internal network interface (e.g., `vmtap0`), and **pf** will handle NAT, allowing the virtual machine to access the external network via the host.

### Host-Only Networking

Host-only networking isolates the virtual machine’s network traffic from the external network, allowing communication only between the host and the virtual machines. This mode is useful for testing environments or scenarios where external network access is not required but communication between virtual machines and the host is necessary.

#### Setting Up Host-Only Networking

In this configuration, a virtual switch connects virtual machines to the host without routing traffic to external networks. No external interface is added to the bridge, keeping the communication isolated. Configure **vm.conf** as follows:

```
vm "vm1" {
    memory 1G
    disk "/path/to/disk.img"
    local interface
}
```

This configuration does not include any physical interface, so the virtual machines will only be able to communicate with the host through the virtual switch.

### Isolated Networking

In isolated networking, the virtual machine is completely cut off from both the host and external networks. This mode is useful for security testing or environments where external communication must be entirely restricted.

#### Setting Up Isolated Networking

By not configuring any network interface for the virtual machine, it can be entirely isolated from any network communication. In **/etc/vm.conf**, omit the `interface` line entirely:

```
vm "vm1" {
    memory 1G
    disk "/path/to/disk.img"
}
```

This configuration creates a virtual machine with no network interfaces, ensuring complete isolation from other machines, the host, and external networks.

## Command Reference

### vmctl(8) Command Overview

| Command | Description | Example |
| --- | --- | --- |
| `vmctl create -s <size> <disk.img>` | Creates a virtual disk image with a specified size. | `vmctl create -s 40G /var/vm/disk.img` |
| `vmctl start -m <memory> -d <disk.img> <name>` | Starts a virtual machine with the specified name, memory allocation, and disk image. | `vmctl start -m 1G -d /var/vm/disk.img "vm1"` |
| `vmctl start -c <name>` | Starts a virtual machine and connects to its console. The `-c` option opens the VM’s console upon startup. | `vmctl start -c -m 1G -d /var/vm/disk.img "vm1"` |
| `vmctl stop <id>` | Stops a running virtual machine by its ID. | `vmctl stop 1` |
| `vmctl console <id>` | Connects to the console of a running virtual machine by its ID. | `vmctl console 1` |
| `vmctl status` | Displays the status of all virtual machines, including their ID, name, memory, and status (running/stopped). | `vmctl status` |
| `vmctl reload` | Reloads the **vmd(8)** daemon configuration from the `/etc/vm.conf` file. | `vmctl reload` |
| `vmctl wait` | Waits for **vmd** to complete booting and loading its configuration. | `vmctl wait` |
| `vmctl terminate` | Terminates the **vmd** daemon (similar to a shutdown). | `vmctl terminate` |
| `vmctl load <filename>` | Loads a saved virtual machine state from a file. | `vmctl load /var/vm/saved_vm_state.img` |
| `vmctl dump <id> -f <filename>` | Dumps the state of a running virtual machine to a file for later recovery. | `vmctl dump 1 -f /var/vm/saved_vm_state.img` |
| `vmctl reset <id>` | Resets a running virtual machine, similar to a hard reset on a physical machine. | `vmctl reset 1` |
| `vmctl pause <id>` | Pauses a running virtual machine by its ID. | `vmctl pause 1` |
| `vmctl unpause <id>` | Unpauses a paused virtual machine by its ID. | `vmctl unpause 1` |
| `vmctl show <id>` | Shows detailed information about a virtual machine, including disk usage, memory allocation, and network setup. | `vmctl show 1` |

### Other Relevant Commands

| Command | Description | Example |
| --- | --- | --- |
| `rcctl enable vmd` | Enables the **vmd(8)** daemon to start at boot. | `rcctl enable vmd` |
| `rcctl start vmd` | Starts the **vmd(8)** daemon manually. | `rcctl start vmd` |
| `rcctl stop vmd` | Stops the **vmd(8)** daemon manually. | `rcctl stop vmd` |
| `rcctl reload vmd` | Reloads the **vmd(8)** daemon configuration from **/etc/vm.conf**. | `rcctl reload vmd` |
| `ifconfig bridge0 create` | Creates a network bridge interface for use with virtual machines. | `ifconfig bridge0 create` |
| `ifconfig bridge0 add <iface>` | Adds a physical interface to a bridge for network bridging. | `ifconfig bridge0 add em0` |
| `ifconfig bridge0 up` | Brings the bridge interface up. | `ifconfig bridge0 up` |
| `dmesg | egrep ‘(VMX/EPT | SVM/RVI)'` |
| `pfctl -f /etc/pf.conf` | Reloads **pf(4)** rules from **/etc/pf.conf**. Useful when setting up NAT for virtual machines. | `pfctl -f /etc/pf.conf` |
| `ftp <url>` | Downloads a file, such as an installation ISO, for use in a virtual machine. | `ftp https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/install78.iso` |
