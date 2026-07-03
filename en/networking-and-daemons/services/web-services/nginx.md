# nginx

## Synopsis

**nginx** is a high-performance HTTP server and reverse proxy known for its efficiency, low memory footprint, and rich feature set. It supports static content, TLS, FastCGI and SCGI backends, reverse proxying, load balancing, and basic authentication.

nginx is **not included in the OpenBSD base system**, but is available via packages.

### Web Server History on OpenBSD

OpenBSD historically shipped with **Apache 1.3** in base, which was removed due to complexity and security concerns. For a brief period (OpenBSD 5.6 to 5.8), **nginx** was included as the default web server. It was later removed in favor of a simpler, security-focused alternative: **httpd(8)**, which is now the default in base.

nginx remains available via `pkg_add` and is well-suited for modern workloads involving TLS termination, static file hosting, FastCGI application serving, and reverse proxy setups.

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
| Recommended use | Static or FastCGI hosting | Reverse proxy, TLS offload | Dynamic content, legacy sites |

Use **httpd(8)** for base simplicity and static content, **nginx** for efficient TLS or FastCGI proxying, and **Apache** for complex dynamic configurations.

## Installation

Install nginx from packages:

```sh
# pkg_add nginx
```

This installs:

- `/etc/nginx/nginx.conf` – main configuration file
- `/usr/local/sbin/nginx` – daemon binary
- `rc.d` service script for `rcctl(8)`

## Basic Configuration

The default configuration file is installed at `/etc/nginx/nginx.conf`.

Minimal example for static file hosting:

```
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;

        root /htdocs;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

Create the document root and test file:

```sh
# mkdir -p /htdocs
# echo "OK" > /htdocs/index.html
# chown -R www:www /htdocs
```

## Starting nginx

Enable and start the service:

```sh
# rcctl enable nginx
# rcctl start nginx
```

To test or reload configuration:

```sh
# nginx -t
# nginx -s reload
```

The process runs as the `_nginx` user.

## TLS Configuration

To enable HTTPS, obtain a certificate (e.g., via `acme-client`) and adjust the configuration:

```
server {
    listen 443 ssl;
    server_name example.org;

    ssl_certificate     /etc/ssl/example.org.fullchain.pem;
    ssl_certificate_key /etc/ssl/private/example.org.key;

    root /htdocs;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Do not forget to open port 443 in `pf.conf`:

```pf
pass in on $int_if proto tcp from any to (self) port 443
```

## FastCGI Support (PHP, etc.)

To serve dynamic content (e.g., via `php-fpm`):

1. Install `php` and `php-fpm`:

```sh
# pkg_add php php-fpm
# rcctl enable php_fpm
# rcctl start php_fpm
```

2. Enable the FastCGI pass in nginx:

```
location ~ \.php$ {
    root           /htdocs;
    fastcgi_pass   unix:/run/php-fpm.sock;
    fastcgi_index  index.php;
    include        fastcgi_params;
    fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
```

Ensure PHP files exist in the document root:

```sh
# echo "<?php phpinfo(); ?>" > /htdocs/index.php
```

## Reverse Proxy Example

nginx can proxy requests to a backend service, such as a Python or Node.js application:

```
server {
    listen 80;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

This setup is commonly used for application frontends, TLS termination, or caching.

## Logging and Diagnostics

Access and error logs are defined in the `http` block:

```nginx
access_log  /var/log/nginx/access.log;
error_log   /var/log/nginx/error.log;
```

To follow logs:

```sh
# tail -f /var/log/nginx/access.log
```

Use `nginx -t` to verify syntax, and `rcctl restart nginx` after changes.

## Security Practices

- Run nginx as `_nginx` (default).
- Ensure files in `/htdocs` are owned by `www` and not world-writable.
- Use `try_files` to avoid path traversal.
- Do not expose nginx to the Internet without enabling TLS.
- Validate all reverse proxy headers if used.

Example `pf.conf` rules:

```pf
block in proto tcp from any to (self) port { 80 443 }
pass in on $int_if proto tcp from 192.0.2.0/24 to (self) port { 80 443 }
```
