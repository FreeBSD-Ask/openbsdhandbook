# 部署 WordPress

## 概览

本章介绍如何在 OpenBSD 上部署 WordPress，使用基本系统自带的 [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
作为 Web 服务器，PHP-FPM 与 MariaDB 取自软件包。OpenBSD 将 Web 服务器运行在 `/var/www` 的 **chroot(2)** 中，因此名称解析与进程间通信必须在该环境内可用。

本章所有命令均假设在 root shell（`#`）下执行。

## 准备：httpd chroot 内的名称解析

确保运行在 `/var/www` 内的进程能够解析主机名。按 [resolv.conf(5)](https://man.openbsdhandbook.com/resolv.conf.5/)
在 chroot 中创建 `/etc` 目录并提供解析器配置。也可按 [hosts(5)](https://man.openbsdhandbook.com/hosts.5/)
创建 `/etc/hosts` 条目。

```sh
# install -d -o root -g wheel -m 0755 /var/www/etc
# cp -p /etc/resolv.conf /var/www/etc/resolv.conf
# printf '127.0.0.1\tlocalhost\n' > /var/www/etc/hosts
```

在 chroot 中提供 `resolv.conf`，可避免硬编码上游主机 IP 这类脆弱做法。

## 安装必需软件包

安装 PHP 及所需扩展、MariaDB 服务器与客户端工具，以及基础工具。

```sh
# pkg_add php php-curl php-mysqli php-zip php-gd php-intl mariadb-server mariadb-client wget unzip
```

## 配置 PHP 与 PHP-FPM

将 PHP 示例配置文件复制到位，版本号部分需按实际安装版本调整（例如 `php-8.2`）。

```sh
# cp /etc/php-8.4.sample/* /etc/php-8.4/
```

创建一个最小化的 PHP-FPM 池：以 `www` 用户运行，监听 chroot 内的 UNIX 套接字，并自身 chroot 到 `/var/www`。

```sh
# install -d -o root -g wheel -m 0755 /etc/php-fpm.d
# vi /etc/php-fpm.d/www.conf
```

```ini
; 用于 httpd FastCGI 的简单池 "www"
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

启用并启动 PHP-FPM 守护进程。

```sh
# rcctl enable php84_fpm
# rcctl start php84_fpm

# 验证服务已启用
# rcctl ls on | grep php

# 验证服务正在运行
# rcctl check php84_fpm
```

服务管理参见 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
。

## 配置 httpd

创建 `/etc/httpd.conf`，写入单个 server 段。FastCGI 套接字路径相对于 chroot；PHP-FPM 监听 `/var/www/run/php-fpm.sock`，对 [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
而言表现为 `/run/php-fpm.sock`。指令详情参见 [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)
。

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

启动并启用 Web 服务器：

```sh
# rcctl enable httpd
# rcctl start httpd

# 验证服务已启用
# rcctl ls on | grep httpd

# 验证服务正在运行
# rcctl check httpd
```

如需 HTTPS，按本手册 Web 服务器章节配置 TLS，并相应更新 `listen` 与证书指令。参见 [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)
。

## 初始化 MariaDB

初始化数据库、启动服务器并运行安全配置程序。这些工具由 MariaDB 软件包提供。

```sh
# mariadb-install-db
# mkdir -p /var/run/mysql
# chown _mysql:_mysql /var/run/mysql/
# rcctl enable mysqld
# rcctl start mysqld
# mariadb-secure-installation
```

为方便起见，可创建 `/etc/my.cnf` 保存客户端默认参数（含管理密码）。

```sh
# 以下选项将传递给所有 MariaDB 客户端
[client]
user    = root
password= your_password
port    = 3306
socket  = /var/run/mysql/mysql.sock
```

若文件中包含凭据，请限制访问权限：

```sh
# chmod 600 /etc/my.cnf
```

## 下载并安装 WordPress

获取最新版 WordPress，解压后放到 chroot 内的 httpd 文档根目录下。

```sh
# cd /tmp
# ftp https://wordpress.org/latest.zip
# unzip latest.zip
# mv wordpress /var/www/
```

为提升安全性，将 WordPress 核心文件设为对 Web 服务器**只读**。安全做法是归属设为 `root:wheel`，仅对运行时必须可写的目录授予写权限（通常是 `wp-content/uploads`；视情况还包括插件专属的缓存/升级目录）。

```sh
# chown -R root:wheel /var/www/wordpress
# find /var/www/wordpress -type d -exec chmod 755 {} \;
# find /var/www/wordpress -type f -exec chmod 644 {} \;

# 运行时可写的内容（最低要求）
# install -d -o www -g www -m 0755 /var/www/wordpress/wp-content/uploads

# 可选：仅当某插件/主题需要写权限时
# install -d -o www -g www -m 0755 /var/www/wordpress/wp-content/cache
# install -d -o www -g www -m 0755 /var/www/wordpress/wp-content/upgrade
```

不要让整个文档根目录都对 `www` 可写。通过管理界面执行更新时，仅临时放宽所需路径的权限，完成后恢复为只读归属。

（OpenBSD 基本系统自带 `ftp(1)`。参见 [ftp(1)](https://man.openbsdhandbook.com/ftp.1/)
。）

## 创建 WordPress 数据库与用户

连接 MariaDB，创建数据库以及一个专用用户账号，并将其权限限定在该数据库内。把 `StrongPassword` 替换为自选的强密码。

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

使用 `127.0.0.1` 会走 TCP，从而避免在 httpd chroot 内依赖服务器的 UNIX 套接字。

## 配置 WordPress

复制示例配置并编辑数据库参数。数据库主机使用回环地址 `127.0.0.1`，以避免跨 chroot 边界的套接字路径问题。

```sh
# cp /var/www/wordpress/wp-config-sample.php /var/www/wordpress/wp-config.php
# vi /var/www/wordpress/wp-config.php
```

```
/** WordPress 数据库名 */
define('DB_NAME', 'wordpress');

/** 数据库用户名 */
define('DB_USER', 'wordpress');

/** 数据库密码 */
define('DB_PASSWORD', 'StrongPassword');

/** 数据库主机名（使用 TCP）*/
define('DB_HOST', '127.0.0.1');
```

## 完成安装

在浏览器中访问服务器的主机名或 IP 地址。WordPress 会显示安装向导，用于创建初始管理员账号与站点元数据。

若权限阻止写入，请确认文档根目录为 `/var/www/wordpress`、上传目录对 `www` 可写，且 PHP-FPM 正在运行并能从 chroot 内通过 `/run/php-fpm.sock` 访问。

## 参考

本章涉及的基本系统工具与配置文件，可参阅本手册托管的手册页：

- [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
- [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)
- [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
- [ftp(1)](https://man.openbsdhandbook.com/ftp.1/)
- [resolv.conf(5)](https://man.openbsdhandbook.com/resolv.conf.5/)
- [hosts(5)](https://man.openbsdhandbook.com/hosts.5/)
