# Set Up WordPress

## Overview

This chapter describes how to deploy WordPress on OpenBSD using the base web server [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
, PHP-FPM from packages, and MariaDB from packages. OpenBSD runs the web server in a **chroot(2)** at `/var/www`, so name resolution and interprocess communication must work from within that environment.

All commands assume the root shell (`#`).

## Preparation: Name Resolution in the httpd Chroot

Ensure that processes running inside `/var/www` can resolve hostnames. Create the chroot’s `/etc` and provide resolver configuration per [resolv.conf(5)](https://man.openbsdhandbook.com/resolv.conf.5/)
. Optionally create `/etc/hosts` entries per [hosts(5)](https://man.openbsdhandbook.com/hosts.5/)
.

```sh
# install -d -o root -g wheel -m 0755 /var/www/etc
# cp -p /etc/resolv.conf /var/www/etc/resolv.conf
# printf '127.0.0.1\tlocalhost\n' > /var/www/etc/hosts
```

Providing `resolv.conf` in the chroot avoids brittle workarounds using hard-coded upstream host IP addresses.

## Install Required Packages

Install PHP with required extensions, MariaDB server and client tools, and basic utilities.

```sh
# pkg_add php php-curl php-mysqli php-zip php-gd php-intl mariadb-server mariadb-client wget unzip
```

## Configure PHP and PHP-FPM

Copy the sample PHP configuration files into place, adjusting the version component to match what was installed (for example, `php-8.2`).

```sh
# cp /etc/php-8.4.sample/* /etc/php-8.4/
```

Create a minimal PHP-FPM pool that runs as user `www`, listens on a UNIX socket inside the chroot, and itself chroots to `/var/www`.

```sh
# install -d -o root -g wheel -m 0755 /etc/php-fpm.d
# vi /etc/php-fpm.d/www.conf
```

```ini
; Simple pool "www" for httpd FastCGI
[www]
user = www
group = www

listen = /var/www/run/php-fpm.sock
listen.owner = www
listen.group = www
listen.mode  = 0660

pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35

chroot = /var/www
```

Enable and start the PHP-FPM daemon.

```sh
# rcctl enable php84_fpm
# rcctl start php84_fpm

# Verify the service is enabled
# rcctl ls on | grep php

# Verify the service is running
# rcctl check php84_fpm
```

See [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
for service management.

## Configure httpd

Create `/etc/httpd.conf` with a single server stanza. The FastCGI socket path is relative to the chroot; PHP-FPM listens at `/var/www/run/php-fpm.sock`, which appears as `/run/php-fpm.sock` to [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
. Refer to [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)
for directive details.

```sh
# vi /etc/httpd.conf
```

```
types { include "/usr/share/misc/mime.types" }

server "default" {
    listen on egress port 80
    root "/wordpress"
    directory index index.php

    location "*.php" {
        fastcgi socket "/run/php-fpm.sock"
    }
}
```

Start and enable the web server:

```sh
# rcctl enable httpd
# rcctl start httpd

# Verify the service is enabled
# rcctl ls on | grep httpd

# Verify the service is running
# rcctl check httpd
```

If HTTPS is required, configure TLS as described in the Handbook’s web server chapter and update the `listen` and certificate directives accordingly. See [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)
.

## Initialize MariaDB

Initialize the database, start the server, and run the secure setup program. These tools are provided by the MariaDB packages.

```sh
# mariadb-install-db
# mkdir -p /var/run/mysql
# chown _mysql:_mysql /var/run/mysql/
# rcctl enable mysqld
# rcctl start mysqld
# mariadb-secure-installation
```

Optionally create `/etc/my.cnf` to store client defaults (including the administrative password) for convenience.

```sh
# The following options will be passed to all MariaDB clients
[client]
user    = root
password= your_password
port    = 3306
socket  = /var/run/mysql/mysql.sock
```

Restrict access if the file contains credentials:

```sh
# chmod 600 /etc/my.cnf
```

## Download and Install WordPress

Fetch the latest WordPress release, extract it, and place it under the httpd document root inside the chroot.

```sh
# cd /tmp
# ftp https://wordpress.org/latest.zip
# unzip latest.zip
# mv wordpress /var/www/
```

For improved security, keep WordPress core files **read-only** to the web server. A safe default is ownership by `root:wheel`, with write access granted only to directories that must be writable at runtime (typically `wp-content/uploads`; optionally plugin-specific cache/upgrade paths).

```sh
# chown -R root:wheel /var/www/wordpress
# find /var/www/wordpress -type d -exec chmod 755 {} \;
# find /var/www/wordpress -type f -exec chmod 644 {} \;

# Runtime-writable content (minimum required)
# install -d -o www -g www -m 0755 /var/www/wordpress/wp-content/uploads

# Optional: only if a plugin/theme requires write access
# install -d -o www -g www -m 0755 /var/www/wordpress/wp-content/cache
# install -d -o www -g www -m 0755 /var/www/wordpress/wp-content/upgrade
```

Avoid making the entire document root writable by `www`. When updates are performed through the admin UI, temporarily relax permissions only for required paths, then restore read-only ownership.

(`ftp(1)` is available in the OpenBSD base system. See [ftp(1)](https://man.openbsdhandbook.com/ftp.1/)
.)

## Create the WordPress Database and User

Connect to MariaDB and create a database and a dedicated user account with privileges limited to that database. Replace `StrongPassword` with a chosen strong password.

```sh
# mysql -u root -p
```

```sql
CREATE DATABASE wordpress;
CREATE USER 'wordpress'@'127.0.0.1' IDENTIFIED BY 'StrongPassword';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'127.0.0.1';
FLUSH PRIVILEGES;
EXIT;
```

Using `127.0.0.1` ensures TCP is used, which avoids reliance on the server’s UNIX socket from within the httpd chroot.

## Configure WordPress

Copy the sample configuration and edit the database parameters. Use the loopback address `127.0.0.1` for the database host to avoid socket path issues across the chroot boundary.

```sh
# cp /var/www/wordpress/wp-config-sample.php /var/www/wordpress/wp-config.php
# vi /var/www/wordpress/wp-config.php
```

```
/** WordPress database name */
define('DB_NAME', 'wordpress');

/** Database username */
define('DB_USER', 'wordpress');

/** Database password */
define('DB_PASSWORD', 'StrongPassword');

/** Database hostname (use TCP) */
define('DB_HOST', '127.0.0.1');
```

## Complete the Installation

Navigate to the server’s hostname or IP address in a web browser. WordPress will present the installation wizard to create the initial administrator account and site metadata.

If permissions prevent writes, confirm that the document root is `/var/www/wordpress`, that uploads are writable by `www`, and that PHP-FPM is running and reachable at `/run/php-fpm.sock` from the chroot.

## References

Consult the Handbook-hosted manual pages for base utilities and configuration files discussed in this chapter:

- [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
- [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)
- [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
- [ftp(1)](https://man.openbsdhandbook.com/ftp.1/)
- [resolv.conf(5)](https://man.openbsdhandbook.com/resolv.conf.5/)
- [hosts(5)](https://man.openbsdhandbook.com/hosts.5/)
