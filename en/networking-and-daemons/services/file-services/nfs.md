# NFS

## Synopsis

**NFS (Network File System)** is a distributed file system protocol supported in the OpenBSD base system. It allows one system to share directories with others across a network. Clients can mount these shared directories and use them as if they were part of the local filesystem.

This chapter documents NFS usage on OpenBSD, covering both server and client roles, daemon management, security options, and best practices for deployment.

## Server Configuration

An OpenBSD system acting as an NFS server must run the `mountd(8)` and `nfsd(8)` daemons. For environments where file locking is needed, `rpc.lockd(8)` and `rpc.statd(8)` should also be enabled. Additionally, `portmap(8)` is required for NFS operation.

Shared directories must be declared in `/etc/exports`. For example, to share `/export/projects` with a local network:

```conf
/export/projects -alldirs -maproot=root: -network 192.168.1.0 -mask 255.255.255.0
```

The `-alldirs` option allows clients to mount subdirectories. The `-maproot=root:` directive preserves root access for trusted clients. The `-network` and `-mask` options limit access to a specific subnet.

After editing `/etc/exports`, the `mountd` daemon must be reloaded:

```sh
# kill -HUP $(cat /var/run/mountd.pid)
```

Create the shared directory and apply appropriate permissions:

```sh
# mkdir -p /export/projects
# chown root:users /export/projects
# chmod 775 /export/projects
```

The server daemons can be enabled and started as follows:

```sh
# rcctl enable portmap mountd nfsd
# rcctl start portmap
# rcctl start mountd
# rcctl start nfsd
```

To support locking:

```sh
# rcctl enable rpc.lockd rpc.statd
# rcctl start rpc.lockd
# rcctl start rpc.statd
```

If `pf(4)` is enabled, it may be necessary to allow TCP and UDP traffic on `sunrpc` and `2049`, or assign static ports using `mountd_flags` and `nfsd_flags` in `/etc/rc.conf.local`.

## Client Configuration

The NFS client does not require additional packages. To mount a remote export manually:

```sh
# mount -t nfs 192.168.1.10:/export/projects /mnt
```

To make the mount persistent, add it to `/etc/fstab`:

```
192.168.1.10:/export/projects /mnt nfs rw 0 0
```

Additional mount options such as `soft`, `bg`, or `nolock` may be useful in certain environments, particularly when the remote server is unreliable or lacks locking support. For example:

```
192.168.1.10:/export/projects /mnt nfs rw,bg,soft 0 0
```

The `portmap` service is required on the client:

```sh
# rcctl enable portmap
# rcctl start portmap
```

To unmount a filesystem:

```sh
# umount /mnt
```

## Service Behavior and Options

NFS requests may use TCP or UDP. OpenBSD defaults to TCP. If firewalling is in use, access to `mountd`, `nfsd`, and `portmap` ports must be permitted. OpenBSD’s `nfsd` supports both protocols but does not default to dynamic port assignment unless configured to do so.

The `mount_nfs(8)` utility recognizes a number of options that influence behavior. The `hard` option (default) causes system calls to retry indefinitely. The `soft` option allows calls to fail after a timeout. Using `bg` causes failed mount attempts to retry in the background. These settings should be chosen based on application tolerance for delay or failure.

## Access Control

The `maproot` and `mapall` options in `/etc/exports` control how client user IDs map to local users. Mapping root to `nobody` is a common security measure to prevent remote root access:

```conf
/export/projects -maproot=nobody -network 192.168.1.0 -mask 255.255.255.0
```

Use care when exporting writable directories to untrusted clients. It is advisable to restrict access by network, use read-only exports, and assign minimal privileges when root mapping is not required.

## Monitoring and Debugging

To confirm exports:

```sh
# showmount -e localhost
```

To verify service status:

```sh
# rcctl check mountd
# rcctl check nfsd
```

Use `tail -f /var/log/messages` or `dmesg` to observe errors or mount failures. On the client, mounting with `-v` may provide additional output:

```sh
# mount -v -t nfs server:/path /mnt
```

## Compatibility Notes

The OpenBSD implementation of NFS is compatible with other Unix systems using NFSv2 or NFSv3. NFSv4 is not supported. Certain Linux clients may default to NFSv4 and must be configured to use an earlier version explicitly when interoperating with OpenBSD:

```sh
# mount -t nfs -o vers=3 openbsd:/export /mnt
```

The `nfsd` and `mountd` daemons do not use `inetd`, and must be run via `rcctl` or managed using local `rc` hooks.
