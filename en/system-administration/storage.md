# Storage and File Systems

## Synopsis

OpenBSD provides a consistent and secure framework for managing disks, partitions, file systems, and advanced storage options such as encryption and software RAID. This chapter describes how to detect new disks, initialize them with an appropriate partitioning scheme (MBR or GPT), define disklabel partitions, create and mount file systems, and ensure proper boot-time configuration with `/etc/fstab`. It also covers removable media, file system checks, and the use of disk monitoring tools.

## Disk Device and Partition Naming

OpenBSD uses a clear and structured device naming convention for storage hardware.

### Whole Disks

Disks are named by driver type and discovery order:

| Device | Description |
| --- | --- |
| `sd0` | First SCSI or SATA disk |
| `sd1` | Second SCSI or SATA disk |
| `wd0` | First IDE disk (rare in modern systems) |
| `vnd0` | First virtual disk (e.g., memory disk) |

These names are used across tools such as `fdisk(8)`, `gpt(8)`, `disklabel(8)`, and file system utilities.

### Partitions

Each disk is divided into partitions using `disklabel(8)`. OpenBSD allows up to 16 partitions per disk, labeled `a` through `p`.

| Partition | Typical Usage |
| --- | --- |
| `a` | Root file system or main mount point |
| `b` | Swap partition |
| `c` | Entire disk (automatically defined; must not be altered or mounted) |
| `d`–`h` | Commonly used for `/usr`, `/home`, `/var`, etc. |
| `i`–`p` | Available for additional file systems or special purposes |

The `c` partition always represents the entire disk and is required by the system. It must never be formatted, mounted, or manually modified.

Disklabel partitions are created and modified with:

```sh
# disklabel -E sd0
```

Once partitions are defined, they can be formatted, mounted, encrypted, or used in RAID sets.

### Device Nodes

OpenBSD uses both block and character device files to interact with partitions:

| Device Node | Usage |
| --- | --- |
| `/dev/sd0a` | Block device (for mounting) |
| `/dev/rsd0a` | Raw character device (for `newfs`, `fsck`) |

Use the raw device (`r` prefix) for formatting and file system checks. Use the block device for mounting.

## Disk Detection

New storage devices are automatically recognized by the kernel. To check which disks are detected:

```sh
# dmesg | grep ^sd
```

This will list devices such as `sd0`, `sd1`, etc. Each detected disk can then be examined and initialized.

## Disk Initialization

Disks must be initialized with either an MBR or GPT partitioning scheme before disklabel partitions can be defined. OpenBSD uses MBR by default on BIOS-based systems and GPT on UEFI-based systems.

### Choosing MBR or GPT

The partitioning method depends on the boot firmware:

| Scheme | Platform | Tools |
| --- | --- | --- |
| MBR | BIOS (legacy) | `fdisk(8)` |
| GPT | UEFI | `gpt(8)` |

For data-only disks that are not used for booting, either scheme may be used.

### MBR with `fdisk`

To initialize a disk with a single OpenBSD partition (type `A6`) on an MBR system:

```sh
# fdisk -iy sd0
```

This overwrites the MBR and creates one OpenBSD partition marked as active. To verify the layout:

```sh
# fdisk sd0
```

If needed, the active (bootable) flag can be set manually:

```sh
# fdisk -e sd0
fdisk: 1> flag 0
fdisk:*1> write
fdisk: 1> quit
```

On some systems, the MBR bootstrap must be installed manually:

```sh
# fdisk -b /usr/mdec/mbr sd0
```

MBR configuration is not required for disks that will not be used for booting.

### GPT with `fdisk`

UEFI-based systems use GPT. OpenBSD does not provide `gpt(8)`; GPT partition tables are created and managed using `fdisk(8)`.

To initialize a disk with a GPT and create an EFI System Partition:

```sh
# fdisk -gy sd0
# fdisk -e sd0
fdisk: 1> setpid 1 EFI System
fdisk: 1> edit 1
Partition id ('0' to disable)  [0 - FF]: EF
Do you wish to edit in CHS mode? [n] n
Partition offset: [default] (press enter)
Partition size: [+512M]
fdisk: 1> write
fdisk: 1> quit
```

After creating the EFI System Partition, create a FAT32 file system on it and install the bootloader:

```sh
# newfs_msdos /dev/rsd0i
# mount_msdos /dev/sd0i /mnt
# mkdir -p /mnt/efi/boot
# cp /usr/mdec/BOOTX64.EFI /mnt/efi/boot/
# umount /mnt
```

OpenBSD partitions inside the disk are then managed using `disklabel(8)`.

To inspect the partition table:

```sh
# fdisk sd0
```

Unlike MBR, GPT does not require marking partitions as active. The UEFI firmware locates the EFI System Partition automatically based on its partition type.

## Partitioning with `disklabel`

After the outer MBR or GPT scheme is in place, OpenBSD-specific partitions are created inside it using `disklabel(8)`.

To create a partition:

```sh
# disklabel -E sd1
sd1> a a
offset: [64]
size: [*]
FS type: [4.2BSD]
sd1*> w
sd1> q
```

This creates a partition `a` using all remaining space after the standard offset (64 sectors), of type `4.2BSD`.

To review the resulting label:

```sh
# disklabel sd1
```

The automatically created `c` partition will always span the entire disk and must not be changed.

## Creating and Mounting File Systems

Once a partition is defined, it can be formatted with a file system and mounted.

### Formatting with `newfs`

To initialize the partition with a standard FFS file system:

```sh
# newfs /dev/rsd1a
```

The raw character device (`rsd1a`) must be used.

### Mounting the File System

Create the mount point and attach the file system:

```sh
# mkdir -p /mnt/storage
# mount /dev/sd1a /mnt/storage
```

To confirm the file system is mounted:

```sh
# df -h /mnt/storage
```

### Configuring `/etc/fstab`

To mount the file system automatically at boot, add a line to `/etc/fstab`:

```
/dev/sd1a /mnt/storage ffs rw 1 2
```

The six fields in each `/etc/fstab` line are:

| Field | Purpose | Example |
| --- | --- | --- |
| 1 | Device path | `/dev/sd1a` |
| 2 | Mount point | `/mnt/storage` |
| 3 | File system type | `ffs` |
| 4 | Mount options | `rw`, `noauto`, etc. |
| 5 | Dump flag (legacy; usually 0 or 1) | `1` |
| 6 | `fsck` pass number | `1` (root), `2` (others), `0` (skip) |

#### Common File System Types

| Type | Description |
| --- | --- |
| `ffs` | OpenBSD Fast File System |
| `cd9660` | ISO 9660 CD/DVD media |
| `msdos` | FAT-formatted file systems |
| `ext2fs` | Read-only access to Linux EXT2/EXT3 volumes |
| `swap` | Swap partition |
| `sw` | Preferred keyword for swap |

#### Mount Options

| Option | Description |
| --- | --- |
| `rw` | Read-write |
| `ro` | Read-only |
| `noexec` | Do not allow execution of binaries |
| `nodev` | Ignore device nodes |
| `nosuid` | Ignore setuid/setgid bits |
| `noauto` | Do not mount at boot |
| `wxallowed` | Allow writable+executable mappings (e.g., JITs) |
| `softdep` | Use soft updates (metadata journal; default on FFS) |
| `userquota` | Enable user quotas |
| `groupquota` | Enable group quotas |

Example `/etc/fstab` entry for a secondary data volume:

```
/dev/sd1a /mnt/data ffs rw,nodev,nosuid 1 2
```

### File System Consistency Checks

If a file system is not unmounted cleanly, it will be checked with `fsck(8)` at boot. To check manually:

```sh
# fsck /dev/sd1a
```

To automatically repair:

```sh
# fsck -y /dev/sd1a
```

Proper unmounting or clean shutdown ensures the file system remains marked clean:

```sh
# umount /mnt/storage
# shutdown -p now
```

## Swap Configuration

Swap space provides virtual memory by allowing the operating system to page memory contents to disk when physical RAM is insufficient. OpenBSD uses dedicated disklabel partitions for swap, conventionally labeled `b`.

### Creating a Swap Partition

During installation, a `b` partition for swap is usually created automatically. If needed, one can be added manually using `disklabel(8)`:

```sh
# disklabel -E sd1
sd1> a b
offset: [next available]
size: [e.g., 4G]
FS type: [swap]
sd1*> w
sd1> q
```

To verify the result:

```sh
# disklabel sd1
```

Look for a line such as:

```
  b:  8388608  1953600  swap
```

### Enabling Swap at Runtime

To activate the swap partition immediately:

```sh
# swapctl -a /dev/sd1b
```

To view active swap devices:

```sh
# swapctl -l
```

### Configuring Persistent Swap in `/etc/fstab`

To enable swap space automatically at boot, add an entry to `/etc/fstab`. The mount point field is always set to `none`, and the file system type should be `swap` or the preferred synonym `sw`.

```
/dev/sd1b none swap sw 0 0
```

The `sw` keyword is preferred in OpenBSD but both forms are supported.

### Multiple Swap Devices

If more than one swap partition is defined, OpenBSD will distribute paging across them. Additional swap devices can be added at runtime using repeated `swapctl -a` calls.

### Removing Swap

To deactivate a swap partition:

```sh
# swapctl -d /dev/sd1b
```

This command only removes the device from the active swap list. It does not modify the disklabel or delete the data.

Swap space is critical for systems with limited physical memory, for large builds or workloads, or to enable system hibernation (if supported on the platform).

## Mounting Removable Media

Removable media such as USB drives, ISO images, and memory-backed disks can be mounted and used like regular file systems. These devices are typically mounted manually when needed.

### USB Storage Devices

USB mass storage devices (such as flash drives) appear as `sdN`, just like other disks. When a USB drive is inserted, the kernel assigns it the next available device number (e.g., `sd2`).

To determine the correct device and partition:

```sh
# dmesg | grep ^sd
# disklabel sd2
```

USB drives are commonly formatted with FAT file systems. To mount a FAT-formatted USB drive:

```sh
# mkdir -p /mnt/usb
# mount -t msdos /dev/sd2i /mnt/usb
```

If the correct partition is unknown, examine the output of `disklabel sd2`. The appropriate partition often has type `MSDOS`.

To safely remove the drive:

```sh
# umount /mnt/usb
```

Ensure no shell or background process is accessing the mount point before unmounting. Devices in use will cause `umount` to fail with “device busy.”

Removable disks should not be listed in `/etc/fstab` unless configured with the `noauto` option to prevent mounting during boot.

### Mounting ISO Images

ISO files (such as OpenBSD install images) can be mounted as read-only file systems using a virtual disk configured with `vnd(4)`.

Attach the ISO image to a virtual disk:

```sh
# vnconfig vnd0 install78.iso
# mount -t cd9660 /dev/vnd0c /mnt/iso
```

The file system type `cd9660` corresponds to the ISO 9660 standard used for CD-ROM media. The `c` partition of `vnd0` always refers to the full image.

To unmount and detach the image:

```sh
# umount /mnt/iso
# vnconfig -u vnd0
```

### Creating and Using Memory Disks

OpenBSD supports memory-backed virtual disks using the `vnd(4)` driver. These are useful for temporary file systems, testing, or recovery environments. They can be backed by a regular file or by anonymous system memory.

#### Using a Backing File

Create a 64 MB image file:

```sh
# dd if=/dev/zero of=/tmp/ramdisk.img bs=1m count=64
# vnconfig vnd0 /tmp/ramdisk.img
# disklabel -E vnd0
vnd0> a a
offset: [64]
size: [*]
FS type: [4.2BSD]
vnd0*> w
vnd0> q
# newfs /dev/rvnd0a
# mount /dev/vnd0a /mnt/ram
```

Unmount and detach when finished:

```sh
# umount /mnt/ram
# vnconfig -u vnd0
```

#### Using Anonymous Memory

To allocate a memory-only disk with no backing file:

```sh
# vnconfig -s labels -c vnd0
# disklabel -E vnd0
# newfs /dev/rvnd0a
# mount /dev/vnd0a /mnt/tmpfs
```

This creates a temporary file system that resides entirely in RAM. Its contents will be lost when the device is unmounted or the system reboots.

The `vnd(4)` driver supports multiple instances (`vnd0`, `vnd1`, etc.), and both ISO images and memory disks can be mounted concurrently.

### Creating ISO Image Files

OpenBSD supports the use of image files for archival, transfer, and secure storage. These can be either plain ISO 9660 images or encrypted container files using `softraid(4)`. Image files are typically attached using `vnconfig(8)` and mounted like normal file systems.

#### Creating a Plain ISO Image

To create a standard ISO 9660 image from a directory:

```sh
# mkhybrid -o archive.iso -R -J /path/to/data
```

This command generates `archive.iso`, a read-only ISO image containing the contents of `/path/to/data`.

To mount the image:

```sh
# vnconfig vnd0 archive.iso
# mount -t cd9660 /dev/vnd0c /mnt
```

To unmount and detach:

```sh
# umount /mnt
# vnconfig -u vnd0
```

This process can be repeated on any system with the image file and `vnd(4)` support.

#### Creating an Encrypted Image File

To create an encrypted file-based volume suitable for sensitive data:

1. **Create the backing file and attach it to a virtual disk:**

   ```sh
   # dd if=/dev/zero of=secure.img bs=1m count=128
   # vnconfig vnd0 secure.img
   ```
2. **Create a `softraid` volume using the CRYPTO discipline:**

   ```sh
   # bioctl -c C -l vnd0 softraid0
   New passphrase:
   Re-type passphrase:
   softraid0: CRYPTO volume attached as sd3
   ```
3. **Label, format, and mount the encrypted file system:**

   ```sh
   # disklabel -E sd3
   sd3> a a
   offset: [64]
   size: [*]
   FS type: [4.2BSD]
   sd3*> w
   sd3> q
   # newfs /dev/rsd3a
   # mkdir -p /mnt/secure
   # mount /dev/sd3a /mnt/secure
   ```

To use the encrypted volume later:

1. Attach the image and unlock it:

   ```sh
   # vnconfig vnd0 secure.img
   # bioctl -c C -l vnd0 softraid0
   Passphrase:
   softraid0: CRYPTO volume attached as sd3
   ```
2. Mount the file system:

   ```sh
   # mount /dev/sd3a /mnt/secure
   ```
3. After use, unmount and detach:

   ```sh
   # umount /mnt/secure
   # bioctl -d sd3
   # vnconfig -u vnd0
   ```

This method allows the creation of secure, portable encrypted containers that behave like removable storage. Encrypted images are system-dependent and require the OpenBSD `softraid(4)` driver to be accessed.

## Advanced Features

OpenBSD includes several advanced features for managing storage, including file system snapshots, quotas, encrypted volumes, and software RAID. These features build on standard file system usage and offer enhanced control, flexibility, and data integrity.

### Snapshots with `fssnap`

A file system snapshot is a read-only, point-in-time image of a mounted file system. Snapshots allow consistent backups or forensic inspection without requiring the file system to be taken offline.

OpenBSD provides snapshot support for FFS2 (Fast File System version 2) via `fssnap(8)`.

To create a snapshot of `/home`:

```sh
# fssnap -c /home/.snap0 /home
```

The snapshot must reside on the same file system and will appear as a read-only file. To mount it:

```sh
# mount -o ro /home/.snap0 /mnt/snap
```

This makes the snapshot accessible at `/mnt/snap` for inspection or backup purposes. To remove the snapshot:

```sh
# umount /mnt/snap
# fssnap -d /home/.snap0
```

Snapshots persist across reboots until explicitly deleted. Multiple snapshots may be created simultaneously.

### Disk Quotas

Quotas allow administrators to restrict disk usage by user or group on a per-file-system basis. Only file systems of type `ffs` support quotas.

To enable quotas:

1. Modify `/etc/fstab` to include `userquota` or `groupquota` in the options field:

   ```
   /dev/sd1a /home ffs rw,userquota 1 2
   ```
2. Create the necessary quota files in the file system root:

   ```sh
   # touch /home/quota.user
   # chmod 600 /home/quota.user
   # edquota -u alice
   ```
3. Enable quotas on the file system:

   ```sh
   # quotaon /home
   ```

To check current usage:

```sh
# quota -u alice
# repquota /home
```

Quotas are enforced immediately and will remain active after reboot, as long as they are defined in `/etc/fstab`.

### Encrypted Volumes with `softraid`

OpenBSD supports full-disk encryption using the `softraid(4)` driver with the `CRYPTO` discipline. This creates an encrypted volume backed by an underlying partition.

To create an encrypted volume from an unused partition (e.g., `sd1a`):

```sh
# bioctl -c C -l sd1a softraid0
New passphrase:
Re-type passphrase:
softraid0: CRYPTO volume attached as sd3
```

This attaches the new volume as a device (e.g., `sd3`). Format and mount the volume:

```sh
# newfs /dev/rsd3c
# mkdir -p /secure
# mount /dev/sd3c /secure
```

On reboot, the system will prompt for the encryption passphrase before the volume is attached. Encrypted volumes may be configured for automatic unlocking using a keydisk if needed.

To view attached volumes and their status:

```sh
# bioctl -v softraid0
```

### Software RAID with `softraid`

The `softraid(4)` subsystem supports RAID configurations using standard disks. Common use cases include mirroring for redundancy (RAID 1) or striping for performance (RAID 0).

To create a mirrored RAID 1 volume from two partitions (`sd1a` and `sd2a`):

```sh
# bioctl -c 1 -l sd1a,sd2a softraid0
softraid0: RAID 1 volume attached as sd3
# newfs /dev/rsd3c
# mount /dev/sd3c /mnt/raid
```

The new device `sd3` acts like any other disk. It can be mounted, checked, and used normally.

To inspect the status:

```sh
# bioctl -v softraid0
```

The system will report whether the volume is healthy, rebuilding, or degraded.

Supported RAID levels in OpenBSD:

| RAID | Name | Supported | Notes |
| --- | --- | --- | --- |
| 0 | Striping | Yes | High performance, no redundancy |
| 1 | Mirroring | Yes | Redundant, suitable for fault tolerance |
| 5 | Striping + Parity | Yes | Balanced redundancy and capacity |
| 10 | RAID 1+0 | Manual | Mirror sets then stripe; setup must be managed manually |
| C | Crypto | Yes | Encrypted volume, not a RAID level in the strict sense |

RAID levels 6, 50, and 60 are not supported in base OpenBSD.

When using software RAID, ensure that all component disks are configured identically in terms of size and alignment. Always test recovery and rebuild procedures in advance. Booting from a softraid volume is supported but may require platform-specific configuration.

## Monitoring and Health

OpenBSD does not include native support for S.M.A.R.T. (Self-Monitoring, Analysis and Reporting Technology) in the base system. However, limited S.M.A.R.T. functionality is available through the `smartmontools` package, which provides the `smartctl(8)` utility and the `smartd(8)` monitoring daemon.

This functionality is useful for monitoring disk health and predicting potential failure in supported ATA and SATA drives. Support for NVMe and USB devices is limited and may depend on the controller.

### Installing `smartmontools`

Install the package using `pkg_add`:

```sh
# pkg_add smartmontools
```

This provides access to the `smartctl` and `smartd` utilities.

### Inspecting Disk Health with `smartctl`

To query a supported disk:

```sh
# smartctl -i /dev/sd0c
```

This command displays device identity and indicates whether S.M.A.R.T. support is enabled. To view all health and error attributes:

```sh
# smartctl -a /dev/sd0c
```

The device name should correspond to the full disk (typically the `c` partition). Not all controllers expose S.M.A.R.T. data; some devices may return limited or no results.

### Enabling Background Monitoring with `smartd`

The `smartd` daemon periodically checks one or more devices and issues warnings when thresholds are exceeded.

#### Step 1: Configure `/etc/smartd.conf`

Each line defines a device and its options. Example:

```
/dev/sd0c -a -m root -M exec /usr/local/libexec/smartmontools/smartd-runner
```

This monitors `/dev/sd0c` with all default checks (`-a`) and sends alerts to the `root` user by executing the default handler script.

To monitor multiple disks, add additional lines.

#### Step 2: Enable and Start the Daemon

Enable the service at boot:

```conf
smartd_flags=""
```

Then start the daemon:

```sh
# rcctl enable smartd
# rcctl start smartd
```

#### Step 3: Simulate and Test

To simulate a check or trigger a test manually:

```sh
# smartctl -t short /dev/sd0c
```

Logs will appear in `/var/log/daemon`. Alerts will be sent via local mail, so ensure that the system’s mail delivery is functioning and that `/etc/mail/aliases` includes a valid entry for `root`.

### Limitations

S.M.A.R.T. functionality on OpenBSD depends on hardware compatibility and driver support. USB-attached drives and NVMe devices are often unsupported or only partially functional. Monitoring of such devices may not be reliable.

The `smartmontools` package is suitable for systems using directly attached SATA disks where predictive failure reporting is desired.
