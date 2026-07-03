# Apache

## Synopsis

The **Apache HTTP Server** is one of the most widely used web servers in the world. It offers a powerful configuration system, dynamic module loading, robust virtual hosting, authentication features, TLS support, and compatibility with CGI and scripting environments such as PHP.

Apache is **not included in the OpenBSD base system**. It is available as a package under the name `www/apache`.

### Web Server History on OpenBSD

Historically, OpenBSD shipped with **Apache 1.3** as the default web server. Over time, concerns about complexity, security, and tight integration with third-party modules led to its removal.

From OpenBSD 5.6 to 5.8, **nginx** was shipped as the default web server. However, due to increasing complexity and external dependency management, it too was removed from base.

Since OpenBSD 5.6, the project has maintained its own secure, minimal web server: **httpd(8)**. This daemon is written and maintained within the OpenBSD project and is now the default in the base system.

Apache remains available via packages and is suitable when full HTTP/1.1/2 support, dynamic content, or advanced features (such as `.htaccess`, mod\_proxy, or mod\_php) are needed.

## Web Server Comparison

| Feature | [httpd(8)](httpd.md) (base) | [nginx](nginx.md) (pkg) | [Apache](apache.md) (pkg) |
| --- | --- | --- | --- |
| Included in base | Yes | No | No |
| Configuration style | Simple, declarative | Modular, declarative | Modular, verbose |
| Dynamic content (CGI) | Yes (via `fastcgi`) | Yes | Yes (mod\_cgi, mod\_php, etc.) |
| TLS support | Yes (native) | Yes | Yes |
| HTTP/2 support | No | Yes | Yes |
| `.htaccess` support | No | No | Yes |
| Resource usage | Very low | Low | Medium to high |
| Module system | Static features only | Modular | Extensive |
| Reverse proxy support | Yes (`relay`) | Yes | Yes (mod\_proxy) |
| Recommended use | Static or FastCGI hosting | Static, FastCGI, reverse proxy | Dynamic content, legacy compatibility |

Use **httpd(8)** for base-system simplicity and static content. Use **nginx** or **Apache** for dynamic content or advanced reverse proxy needs.

## Installation

Apache is installed via the package system:

```sh
# pkg_add apache-httpd
```

This provides the Apache 2.4 binary, modules, configuration templates, and startup script.

## Configuration

Apache’s primary configuration file is:

```text
/etc/apache2/httpd.conf
```

The directory `/etc/apache2/` also includes subdirectories for optional configurations:

- `extra/` — supplementary virtual host and module settings
- `original/` — saved reference defaults

To begin, create a simple virtual host configuration:

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

Create the document root and test file:

```sh
# mkdir -p /htdocs
# echo 'OK' > /htdocs/index.html
# chown -R www:www /htdocs
```

## Starting Apache

Start Apache manually:

```sh
# /usr/local/sbin/httpd -f /etc/apache2/httpd.conf
```

To start it at boot, add the following line to `/etc/rc.local`:

```sh
if [ -x /usr/local/sbin/httpd ]; then
    echo -n ' apache'; /usr/local/sbin/httpd -f /etc/apache2/httpd.conf
fi
```

Apache does **not** come with an `rc.d` script by default. One may be created manually if `rcctl` support is needed.

## TLS Support

Apache supports HTTPS via `mod_ssl`.

Add the following to `httpd.conf`:

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

Ensure the certificate and key exist:

```sh
# ls -l /etc/ssl/example.crt /etc/ssl/private/example.key
```

You may use Let’s Encrypt and `acme-client(1)` to obtain valid certificates.

Restart Apache after enabling TLS:

```sh
# pkill httpd
# /usr/local/sbin/httpd -f /etc/apache2/httpd.conf
```

## Access Logs

Apache logs access and error information by default:

- `/var/log/apache2-access.log`
- `/var/log/apache2-error.log`

To view:

```sh
# tail -f /var/log/apache2-access.log
```

## Module Management

Apache supports many modules. Common examples include:

- `mod_rewrite` — URL rewriting
- `mod_alias` — aliasing paths
- `mod_cgi` — running shell scripts or compiled CGI
- `mod_php` — dynamic PHP integration
- `mod_proxy` — reverse proxy capabilities

Modules must be loaded via `LoadModule` in `httpd.conf`. Paths refer to `.so` files under `/usr/local/lib/apache2`.

## Security Notes

Apache runs as the unprivileged `_www` user. Ensure content files are owned appropriately and do not grant write access to system users.

Example:

```sh
# chown -R www:www /htdocs
# chmod -R o-rwx /htdocs/private
```

Avoid enabling `.htaccess` (`AllowOverride`) unless explicitly needed.

Use `pf.conf` to restrict access to port 80/443 as needed:

```pf
pass in on $int_if proto tcp from any to (self) port {80 443}
```
