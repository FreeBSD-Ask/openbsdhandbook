# Apache

## 概述

**Apache HTTP Server** 是全球使用最广泛的 Web 服务器之一。它提供强大的配置系统、动态模块加载、稳健的虚拟主机、认证功能、TLS 支持，并与 CGI 及 PHP 等脚本环境兼容。

Apache **不包含于 OpenBSD 基本系统**，可通过 `www/apache` 软件包获取。

### OpenBSD 上的 Web 服务器历史

历史上，OpenBSD 曾将 **Apache 1.3** 作为默认 Web 服务器随系统发布。随着时间推移，出于复杂性、安全性以及与第三方模块耦合过紧的考虑，Apache 被移除。

OpenBSD 5.6 至 5.8 期间，**nginx** 成为默认 Web 服务器。但同样由于复杂度上升与外部依赖管理问题，nginx 也从基本系统中移除。

自 OpenBSD 5.6 起，项目维护自有的一款安全、精简的 Web 服务器：**httpd(8)**。该守护进程由 OpenBSD 项目内部编写与维护，现为基本系统的默认 Web 服务器。

Apache 仍可通过软件包获取，适用于需要完整 HTTP/1.1/2 支持、动态内容或高级功能（如 **.htaccess**、`mod_proxy` 或 `mod_php`）的场合。

## Web 服务器对比

| 特性 | [httpd(8)](httpd.md)（基本系统） | [nginx](nginx.md)（软件包） | [Apache](apache.md)（软件包） |
| --- | --- | --- | --- |
| 是否包含于基本系统 | 是 | 否 | 否 |
| 配置风格 | 简洁、声明式 | 模块化、声明式 | 模块化、详尽 |
| 动态内容（CGI） | 是（经 `fastcgi`） | 是 | 是（`mod_cgi`、`mod_php` 等） |
| TLS 支持 | 是（原生） | 是 | 是 |
| HTTP/2 支持 | 否 | 是 | 是 |
| **.htaccess** 支持 | 否 | 否 | 是 |
| 资源占用 | 极低 | 低 | 中至高 |
| 模块体系 | 仅静态特性 | 模块化 | 丰富 |
| 反向代理支持 | 是（`relay`） | 是 | 是（`mod_proxy`） |
| 推荐用途 | 静态或 FastCGI 托管 | 静态、FastCGI、反向代理 | 动态内容、遗留兼容 |

基本系统的简洁与静态内容场景可使用 **httpd(8)**。动态内容或高级反向代理需求可使用 **nginx** 或 **Apache**。

## 安装

通过软件包系统安装 Apache：

```sh
# pkg_add apache-httpd
```

该命令提供 Apache 2.4 二进制文件、模块、配置模板与启动脚本。

## 配置

Apache 的主配置文件为：

```text
/etc/apache2/httpd.conf
```

**/etc/apache2/** 目录还包含若干可选配置子目录：

- `extra/` — 补充的虚拟主机与模块设置
- `original/` — 保存的参考默认配置

首先，创建一个简单的虚拟主机配置：

```
ServerRoot "/etc/apache2"
Listen 80

LoadModule mpm_prefork_module lib/apache2/mod_mpm_prefork.so
LoadModule dir_module lib/apache2/mod_dir.so
LoadModule mime_module lib/apache2/mod_mime.so
LoadModule log_config_module lib/apache2/mod_log_config.so
LoadModule alias_module lib/apache2/mod_alias.so
LoadModule authz_core_module lib/apache2/mod_authz_core.so
LoadModule unixd_module lib/apache2/mod_unixd.so
LoadModule rewrite_module lib/apache2/mod_rewrite.so

User www
Group www

DocumentRoot "/htdocs"
<Directory "/htdocs">
    Require all granted
    Options Indexes FollowSymLinks
    AllowOverride None
</Directory>

ErrorLog "/var/log/apache2-error.log"
CustomLog "/var/log/apache2-access.log" common
```

创建文档根目录与测试文件：

```sh
# mkdir -p /htdocs
# echo 'OK' > /htdocs/index.html
# chown -R www:www /htdocs
```

## 启动 Apache

手动启动 Apache：

```sh
# /usr/local/sbin/httpd -f /etc/apache2/httpd.conf
```

要在开机时启动，在 **/etc/rc.local** 中添加以下内容：

```sh
if [ -x /usr/local/sbin/httpd ]; then
    echo -n ' apache'; /usr/local/sbin/httpd -f /etc/apache2/httpd.conf
fi
```

Apache 默认 **不**附带 `rc.d` 脚本。若需要 `rcctl` 支持，可手动创建。

## TLS 支持

Apache 通过 `mod_ssl` 支持 HTTPS。

在 **httpd.conf** 中添加：

```
LoadModule ssl_module lib/apache2/mod_ssl.so

Listen 443
<VirtualHost _default_:443>
    DocumentRoot "/htdocs"
    SSLEngine on
    SSLCertificateFile "/etc/ssl/example.crt"
    SSLCertificateKeyFile "/etc/ssl/private/example.key"
</VirtualHost>
```

确保证书与密钥存在：

```sh
# ls -l /etc/ssl/example.crt /etc/ssl/private/example.key
```

可使用 Let's Encrypt 与 `acme-client(1)` 获取有效证书。

启用 TLS 后重启 Apache：

```sh
# pkill httpd
# /usr/local/sbin/httpd -f /etc/apache2/httpd.conf
```

## 访问日志

Apache 默认记录访问与错误信息：

- **/var/log/apache2-access.log**
- **/var/log/apache2-error.log**

查看：

```sh
# tail -f /var/log/apache2-access.log
```

## 模块管理

Apache 支持众多模块。常见示例包括：

- `mod_rewrite` — URL 重写
- `mod_alias` — 路径别名
- `mod_cgi` — 运行 shell 脚本或编译型 CGI
- `mod_php` — 动态 PHP 集成
- `mod_proxy` — 反向代理能力

模块须通过 **httpd.conf** 中的 `LoadModule` 加载。路径指向 **/usr/local/lib/apache2** 下的 `.so` 文件。

## 安全说明

Apache 以非特权用户 `_www` 运行。确保内容文件属主设置得当，且不向系统用户授予写权限。

示例：

```sh
# chown -R www:www /htdocs
# chmod -R o-rwx /htdocs/private
```

除非确有必要，否则不要启用 **.htaccess**（`AllowOverride`）。

按需使用 **pf.conf** 限制对端口 80/443 的访问：

```pf
pass in on $int_if proto tcp from any to (self) port {80 443}
```
