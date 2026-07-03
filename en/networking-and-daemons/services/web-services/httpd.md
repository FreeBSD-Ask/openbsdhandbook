# httpd

## Synopsis

[httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
is OpenBSD’s native web server daemon. It is simple and security-focused, runs in a [chroot(2)](https://man.openbsdhandbook.com/chroot.2/)
to `/var/www` by default, serves static content, and dispatches dynamic requests to FastCGI backends. TLS is built in and integrates with [acme-client(1)](https://man.openbsdhandbook.com/acme-client.1/)
for automated certificate management.

### Web Server History on OpenBSD

OpenBSD initially shipped with Apache 1.3, transitioned briefly to nginx (OpenBSD 5.6–5.8), and since OpenBSD 5.9 has included the native [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
in base.

## Web Server Comparison

| Feature | [httpd(8)](httpd.md) (base) | [nginx](nginx.md) (pkg) | [Apache](apache.md) (pkg) |
| --- | --- | --- | --- |
| Included in base | Yes | No | No |
| Configuration style | Simple, declarative | Modular, declarative | Modular, verbose |
| Dynamic content | FastCGI | Yes | Yes |
| TLS support | Native | Yes | Yes |
| HTTP/2 support | No | Yes | Yes |
| `.htaccess` | No | No | Yes |
| Reverse proxy | With [relayd(8)](https://man.openbsdhandbook.com/relayd.8/) | Yes | Yes |
| Recommended use | Static/FastCGI hosting | Reverse proxy, TLS offload | Legacy/complex deployments |

## Configuration

The configuration file is [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)
. Paths in `server` and `location` blocks are relative to the `/var/www` chroot unless noted otherwise.

```text
/etc/httpd.conf
```

Minimal configuration to serve static content:

```
types { include "/usr/share/misc/mime.types" }

server "default" {
    listen on * port 80
    root "/htdocs"
}
```

Create the document root and a test page:

```sh
# mkdir -p /var/www/htdocs
# printf 'Welcome to OpenBSD httpd\n' > /var/www/htdocs/index.html
```

Enable and start the service with [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
:

```sh
# rcctl enable httpd
# rcctl start httpd
```

Validate configuration at any time:

```sh
# httpd -n
```

## Logging

By default, logs are written inside the chroot:

```text
/var/www/logs/access.log
/var/www/logs/error.log
```

Follow requests in real time:

```sh
# tail -f /var/www/logs/access.log
```

Per-server logging can be customized:

```
server "example.org" {
    log style combined
    access log "example-access.log"
    error  log "example-error.log"
    root "/htdocs/example"
    listen on * port 80
}
```

## TLS with acme-client(1)

Issue and renew TLS certificates from Let’s Encrypt with [acme-client(1)](https://man.openbsdhandbook.com/acme-client.1/)
; the procedure below provides a complete, current workflow.

### Prerequisites

- DNS A/AAAA records resolve to the server.
- TCP ports 80 and 443 are reachable (adjust [pf(4)](https://man.openbsdhandbook.com/pf.4/)
  ).
- The ACME “http-01” challenge path `/.well-known/acme-challenge/` is exposed.

Create the standard challenge directory inside the chroot:

```sh
# install -d -o root -g www -m 0755 /var/www/acme
```

### Expose the ACME challenge

Map the challenge path to `/var/www/acme`:

```
server "example.org" {
    listen on * port 80
    root "/htdocs/example.org"

    # ACME http-01 challenge
    location "/.well-known/acme-challenge/*" {
        root "/acme"
        request strip 2
        directory no auto index
    }
}
```

Reload:

```sh
# rcctl reload httpd
```

### Configure acme-client(1)

Create `/etc/acme-client.conf` (see [acme-client.conf(5)](https://man.openbsdhandbook.com/acme-client.conf.5/)
):

```
authority letsencrypt {
    api url "https://acme-v02.api.letsencrypt.org/directory"
    account key "/etc/acme/letsencrypt-privkey.pem"
}

domain "example.org" {
    alternative names { "www.example.org" }
    domain key "/etc/ssl/private/example.org.key"
    domain full chain certificate "/etc/ssl/example.org.fullchain.pem"
    sign with letsencrypt
    challengedir "/var/www/acme"
}
```

Private keys under `/etc/ssl/private/` must be readable by root only.

### Obtain the initial certificate

```sh
# acme-client -v example.org && rcctl reload httpd
```

On success, the certificate chain is `/etc/ssl/example.org.fullchain.pem` and the private key is `/etc/ssl/private/example.org.key`.

### Enable HTTPS and redirect HTTP

Keep port 80 for ACME challenges and 301 redirects; serve the site on port 443 with HSTS.

```
types { include "/usr/share/misc/mime.types" }

# HTTP: serve ACME challenges; redirect everything else
server "example.org" {
    listen on * port 80
    root "/htdocs/example.org"

    location "/.well-known/acme-challenge/*" {
        root "/acme"
        request strip 2
        directory no auto index
    }

    location * {
        block return 301 "https://$HTTP_HOST$REQUEST_URI"
    }
}

# HTTPS: primary site
server "example.org" {
    listen on * tls port 443
    hsts

    tls {
        certificate "/etc/ssl/example.org.fullchain.pem"
        key         "/etc/ssl/private/example.org.key"
        # ocsp       "/etc/ssl/example.org.ocsp"   # see OCSP section
    }

    root "/htdocs/example.org"
    directory index "index.html"
}
```

### Automated renewal

Schedule regular renewals and reload only on change; `~` randomizes the minute (see [crontab(5)](https://man.openbsdhandbook.com/crontab.5/)
).

```sh
~ 3 * * * acme-client example.org && rcctl reload httpd
```

## OCSP Stapling

**Online Certificate Status Protocol (OCSP)** provides certificate revocation status. **OCSP stapling** allows the server to fetch a signed OCSP response from the certificate authority and include (“staple”) it in the TLS handshake, improving privacy and reducing latency.

Generate and maintain a stapled OCSP response with [ocspcheck(8)](https://man.openbsdhandbook.com/ocspcheck.8/)
, then reference it in the `tls {}` block. Paths for `tls ocsp` are absolute, not relative to the chroot.

```sh
# ocspcheck -o /etc/ssl/example.org.ocsp /etc/ssl/example.org.fullchain.pem
# rcctl reload httpd
```

Refresh periodically via [cron(8)](https://man.openbsdhandbook.com/cron.8/)
) because OCSP responses expire more frequently than certificates:

```sh
30 2 * * * ocspcheck -q -o /etc/ssl/example.org.ocsp /etc/ssl/example.org.fullchain.pem && rcctl reload httpd
```

If the stapling file is missing or expired, TLS continues to operate without stapling; clients may fall back to live OCSP queries.

## Common Configuration Examples

All directives are documented in [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)
.

### 301 redirects (permanent)

**HTTP→HTTPS (shown above)** and common host canonicalization:

```
# Redirect www to apex
server "www.example.org" {
    listen on * port 80
    listen on * tls port 443
    tls { certificate "/etc/ssl/example.org.fullchain.pem"
          key         "/etc/ssl/private/example.org.key" }
    block return 301 "https://example.org$REQUEST_URI"
}
```

**Directory relocation:**

```
location "/old-section/*" {
    block return 301 "/new-section/$REQUEST_URI"
}
```

### URL prefix mapping and clean paths

Use `request strip` to remove leading components before file lookup.

```
server "example.org" {
    listen on * tls port 443
    root "/htdocs"

    # Serve /app/* from /var/www/htdocs/app
    location "/app/*" {
        root "/htdocs/app"
        request strip 1
        directory index "index.html"
    }
}
```

**Single-page application fallback:**

```
server "spa.example.org" {
    listen on * tls port 443
    root "/htdocs/spa"
    directory index "index.html"

    # Known asset types are served as-is
    location "/*.{css,js,png,jpg,svg,ico}" { }

    # Everything else falls back to index.html
    location * {
        block return 200 "/index.html"
    }
}
```

### HTTP Basic authentication

OpenBSD’s `httpd` supports HTTP Basic Authentication for restricting access to specific locations.

#### Use an htpasswd file

This approach separates credential management from the primary configuration file and simplifies long-term administration.

1. Create a password file

Create a directory within the default chroot and generate bcrypt password hashes using `htpasswd(1)`:

```sh
$ doas mkdir -p /var/www/auth
$ doas htpasswd -c /var/www/auth/mysite alice
$ doas htpasswd /var/www/auth/mysite bob
$ doas chown www: /var/www/auth/mysite
```

> The path specified in `httpd.conf` must be relative to `/var/www` (the default chroot environment).

2. Reference it in `httpd.conf`

```conf
server "example.org" {
    listen on * tls port 443
    root "/htdocs"

    location "/private/*" {
        authenticate "Restricted Area" with "/auth/mysite"
        request strip 1
    }
}
```

When `/private/` is accessed, the server prompts for credentials and validates them against the specified htpasswd file.

> `httpd(8)` does **not** support inline username/password blocks in `httpd.conf`.
>
> Use `authenticate ... with "/path/to/htpasswd"` as shown above. The `authentication { user ... }` style does not apply to OpenBSD `httpd`.

### Static gzip delivery (gzip-static)

Enable static gzip compression to save bandwidth. When gzip encoding is accepted and a requested file also exists with a `.gz` suffix, `httpd` serves the compressed file with `Content-Encoding: gzip`.

Enable at server or location scope:

```
server "example.org" {
    listen on * tls port 443
    root "/htdocs/site"
    gzip-static

    # Optionally, restrict to an assets subtree
    location "/assets/*" {
        gzip-static
        request strip 1
        root "/htdocs/site/assets"
    }
}
```

Precompress assets during deployment. Example for common text types:

```sh
# find /var/www/htdocs/site -type f \
  -name '*.css' -o -name '*.js' -o -name '*.svg' -o -name '*.html' \
  -exec gzip -k -9 {} \;
```

### Custom error responses

```
server "example.org" {
    listen on * tls port 443
    root "/htdocs/site"

    # Return 410 for a retired path
    location "/deprecated/*" { block return 410 }

    # Custom 404 page
    location * {
        block return 404 "/errors/404.html"
    }
}
```

## FastCGI and Dynamic Content

Dynamic applications are served via FastCGI.

### Using slowcgi(8) (base)

Enable the wrapper and direct matching locations to its socket:

```sh
# rcctl enable slowcgi
# rcctl start slowcgi
```

```
server "cgi.example.org" {
    listen on * port 80
    root "/htdocs/cgi.example"

    location "*.cgi" {
        fastcgi
        fastcgi socket "/run/slowcgi.sock"   # inside chroot
    }
}
```

### Using PHP via php-fpm (packages)

Install and run `php-fpm` for the selected PHP version (service name includes the version, e.g., `php84_fpm`). Ensure the FastCGI socket path is visible inside `/var/www`.

```sh
# pkg_add php php-fpm
# rcctl enable php84_fpm
# rcctl start php84_fpm
```

```
server "php.example.org" {
    listen on * port 80
    root "/htdocs/php.example"
    directory index "index.php"

    location "*.php" {
        fastcgi
        fastcgi socket "/run/php-fpm.sock"
    }
}
```

Test script:

```sh
# printf '<?php phpinfo(); ?>' > /var/www/htdocs/php.example/info.php
```

## Security Notes

- [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)
  runs as `_httpd` and is **chrooted to `/var/www`** by default. Paths in `server`/`location` contexts are relative to the chroot. Paths in `tls { certificate|key|ocsp }` are absolute.
- Avoid write access on served files for untrusted users. Use [chmod(1)](https://man.openbsdhandbook.com/chmod.1/)
  and [chown(8)](https://man.openbsdhandbook.com/chown.8/)
  .
- Keep an HTTP virtual server on port 80 for ACME renewal and to 301-redirect to HTTPS.
- Open required ports in [pf(4)](https://man.openbsdhandbook.com/pf.4/)
  :

  ```pf
  pass in on $ext_if proto tcp to (self) port { 80 443 }
  ```

## Service Management and Troubleshooting

Manage the daemon with [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/)
:

```sh
# rcctl check httpd
# rcctl restart httpd
```

Syntax checks and logs:

```sh
# httpd -n
# tail -f /var/www/logs/error.log
```

Common issues:

- ACME challenge returns 404: verify the challenge `location` block and that `/var/www/acme` exists and is readable.
- TLS fails to start: confirm readable certificate and key paths; see [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)
  `tls` block.
- FastCGI errors: confirm backend socket path and permissions inside the chroot.
