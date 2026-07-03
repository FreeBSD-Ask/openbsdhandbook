# httpd

## 概述

[httpd(8)](https://man.openbsdhandbook.com/httpd.8/) 是 OpenBSD 原生的 Web 服务器守护进程。它简洁且以安全为核心，默认在 [chroot(2)](https://man.openbsdhandbook.com/chroot.2/) 中运行于 **/var/www**，提供静态内容，并将动态请求分发给 FastCGI 后端。TLS 内建支持，并与 [acme-client(1)](https://man.openbsdhandbook.com/acme-client.1/) 集成以实现证书自动化管理。

### OpenBSD 上的 Web 服务器历史

OpenBSD 最初随系统附带 Apache 1.3，曾短暂改用 nginx（OpenBSD 5.6–5.8），自 OpenBSD 5.9 起在基本系统中内置原生的 [httpd(8)](https://man.openbsdhandbook.com/httpd.8/)。

## Web 服务器对比

| 特性 | [httpd(8)](httpd.md)（基本系统） | [nginx](nginx.md)（软件包） | [Apache](apache.md)（软件包） |
| --- | --- | --- | --- |
| 是否包含于基本系统 | 是 | 否 | 否 |
| 配置风格 | 简洁、声明式 | 模块化、声明式 | 模块化、详尽 |
| 动态内容 | FastCGI | 是 | 是 |
| TLS 支持 | 原生 | 是 | 是 |
| HTTP/2 支持 | 否 | 是 | 是 |
| **.htaccess** | 否 | 否 | 是 |
| 反向代理 | 配合 [relayd(8)](https://man.openbsdhandbook.com/relayd.8/) | 是 | 是 |
| 推荐用途 | 静态/FastCGI 托管 | 反向代理、TLS 卸载 | 遗留/复杂部署 |

## 配置

配置文件为 [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)。`server` 与 `location` 块中的路径相对于 **/var/www** chroot，除非另作说明。

```text
/etc/httpd.conf
```

提供静态内容的最小配置：

```
types { include "/usr/share/misc/mime.types" }

server "default" {
    listen on * port 80
    root "/htdocs"
}
```

创建文档根目录与测试页：

```sh
# mkdir -p /var/www/htdocs
# printf 'Welcome to OpenBSD httpd\n' > /var/www/htdocs/index.html
```

使用 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/) 启用并启动服务：

```sh
# rcctl enable httpd
# rcctl start httpd
```

随时校验配置：

```sh
# httpd -n
```

## 日志

默认情况下，日志写入 chroot 内部：

```text
/var/www/logs/access.log
/var/www/logs/error.log
```

实时跟踪请求：

```sh
# tail -f /var/www/logs/access.log
```

可按服务器定制日志：

```
server "example.org" {
    log style combined
    access log "example-access.log"
    error  log "example-error.log"
    root "/htdocs/example"
    listen on * port 80
}
```

## 配合 acme-client(1) 使用 TLS

使用 [acme-client(1)](https://man.openbsdhandbook.com/acme-client.1/) 从 Let's Encrypt 签发并续签 TLS 证书；下面的流程提供一套完整且现行的工作方式。

### 前提

- DNS A/AAAA 记录已解析到本服务器。
- TCP 端口 80 与 443 可达（按需调整 [pf(4)](https://man.openbsdhandbook.com/pf.4/)）。
- ACME “http-01” 验证路径 `/.well-known/acme-challenge/` 已暴露。

在 chroot 内创建标准验证目录：

```sh
# install -d -o root -g www -m 0755 /var/www/acme
```

### 暴露 ACME 验证

将验证路径映射到 **/var/www/acme**：

```
server "example.org" {
    listen on * port 80
    root "/htdocs/example.org"

    # ACME http-01 验证
    location "/.well-known/acme-challenge/*" {
        root "/acme"
        request strip 2
        directory no auto index
    }
}
```

重载：

```sh
# rcctl reload httpd
```

### 配置 acme-client(1)

创建 **/etc/acme-client.conf**（参见 [acme-client.conf(5)](https://man.openbsdhandbook.com/acme-client.conf.5/)）：

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

**/etc/ssl/private/** 下的私钥须仅 root 可读。

### 获取初始证书

```sh
# acme-client -v example.org && rcctl reload httpd
```

成功后，证书链为 **/etc/ssl/example.org.fullchain.pem**，私钥为 **/etc/ssl/private/example.org.key**。

### 启用 HTTPS 并重定向 HTTP

端口 80 保留用于 ACME 验证与 301 重定向；端口 443 提供站点服务并启用 HSTS。

```
types { include "/usr/share/misc/mime.types" }

# HTTP：服务 ACME 验证；其余一律重定向
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

# HTTPS：主站点
server "example.org" {
    listen on * tls port 443
    hsts

    tls {
        certificate "/etc/ssl/example.org.fullchain.pem"
        key         "/etc/ssl/private/example.org.key"
        # ocsp       "/etc/ssl/example.org.ocsp"   # 参见 OCSP 一节
    }

    root "/htdocs/example.org"
    directory index "index.html"
}
```

### 自动续签

定期调度续签，且仅在证书变化时重载；`~` 会随机化分钟（参见 [crontab(5)](https://man.openbsdhandbook.com/crontab.5/)）。

```sh
~ 3 * * * acme-client example.org && rcctl reload httpd
```

## OCSP 装订

**在线证书状态协议（OCSP）** 提供证书吊销状态。**OCSP 装订** 让服务器从证书机构获取已签名的 OCSP 响应，并在 TLS 握手时一并附带（“装订”），从而提升隐私并降低延迟。

使用 [ocspcheck(8)](https://man.openbsdhandbook.com/ocspcheck.8/) 生成并维护装订的 OCSP 响应，然后在 `tls {}` 块中引用。`tls ocsp` 的路径为绝对路径，不相对于 chroot。

```sh
# ocspcheck -o /etc/ssl/example.org.ocsp /etc/ssl/example.org.fullchain.pem
# rcctl reload httpd
```

通过 [cron(8)](https://man.openbsdhandbook.com/cron.8/) 定期刷新，因为 OCSP 响应比证书过期更频繁：

```sh
30 2 * * * ocspcheck -q -o /etc/ssl/example.org.ocsp /etc/ssl/example.org.fullchain.pem && rcctl reload httpd
```

若装订文件缺失或过期，TLS 仍可正常运行，只是不带装订；客户端可能回退到实时 OCSP 查询。

## 常见配置示例

所有指令均见 [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/)。

### 301 重定向（永久）

**HTTP→HTTPS（见上文）** 与常见的主机名规范化：

```
# 将 www 重定向到主域名
server "www.example.org" {
    listen on * port 80
    listen on * tls port 443
    tls { certificate "/etc/ssl/example.org.fullchain.pem"
          key         "/etc/ssl/private/example.org.key" }
    block return 301 "https://example.org$REQUEST_URI"
}
```

**目录迁移：**

```
location "/old-section/*" {
    block return 301 "/new-section/$REQUEST_URI"
}
```

### URL 前缀映射与整洁路径

使用 `request strip` 在文件查找前去除前导路径段。

```
server "example.org" {
    listen on * tls port 443
    root "/htdocs"

    # 从 /var/www/htdocs/app 提供 /app/* 内容
    location "/app/*" {
        root "/htdocs/app"
        request strip 1
        directory index "index.html"
    }
}
```

**单页应用回退：**

```
server "spa.example.org" {
    listen on * tls port 443
    root "/htdocs/spa"
    directory index "index.html"

    # 已知资源类型按原样提供
    location "/*.{css,js,png,jpg,svg,ico}" { }

    # 其余一律回退到 index.html
    location * {
        block return 200 "/index.html"
    }
}
```

### HTTP 基本认证

OpenBSD 的 `httpd` 支持 HTTP 基本认证，可限制对特定位置的访问。

#### 使用 htpasswd 文件

这种方式将凭据管理从主配置文件中分离，便于长期维护。

1. 创建密码文件

在默认 chroot 内创建目录，并使用 `htpasswd(1)` 生成 bcrypt 密码哈希：

```sh
$ doas mkdir -p /var/www/auth
$ doas htpasswd -c /var/www/auth/mysite alice
$ doas htpasswd /var/www/auth/mysite bob
$ doas chown www: /var/www/auth/mysite
```

> **httpd.conf** 中指定的路径须相对于 **/var/www**（默认 chroot 环境）。

2. 在 **httpd.conf** 中引用

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

访问 `/private/` 时，服务器会提示输入凭据，并按指定的 htpasswd 文件校验。

> `httpd(8)` **不**支持在 **httpd.conf** 中内联用户名/密码块。
>
> 请按上文所示使用 `authenticate ... with "/path/to/htpasswd"`。`authentication { user ... }` 风格不适用于 OpenBSD `httpd`。

### 静态 gzip 投递（gzip-static）

启用静态 gzip 压缩可节省带宽。当客户端接受 gzip 编码、且请求文件同时存在带 `.gz` 后缀的版本时，`httpd` 会以 `Content-Encoding: gzip` 投递压缩文件。

在 server 或 location 作用域启用：

```
server "example.org" {
    listen on * tls port 443
    root "/htdocs/site"
    gzip-static

    # 也可限制在资源子树内
    location "/assets/*" {
        gzip-static
        request strip 1
        root "/htdocs/site/assets"
    }
}
```

部署时预压缩资源。常见文本类型的示例：

```sh
# find /var/www/htdocs/site -type f \
  -name '*.css' -o -name '*.js' -o -name '*.svg' -o -name '*.html' \
  -exec gzip -k -9 {} \;
```

### 自定义错误响应

```
server "example.org" {
    listen on * tls port 443
    root "/htdocs/site"

    # 对已弃用路径返回 410
    location "/deprecated/*" { block return 410 }

    # 自定义 404 页面
    location * {
        block return 404 "/errors/404.html"
    }
}
```

## FastCGI 与动态内容

动态应用通过 FastCGI 提供。

### 使用 slowcgi(8)（基本系统）

启用包装器，并将匹配的 location 指向其套接字：

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
        fastcgi socket "/run/slowcgi.sock"   # chroot 内部
    }
}
```

### 通过 php-fpm 使用 PHP（软件包）

为所选 PHP 版本安装并运行 `php-fpm`（服务名含版本号，例如 `php84_fpm`）。确保 FastCGI 套接字路径在 **/var/www** 内可见。

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

测试脚本：

```sh
# printf '<?php phpinfo(); ?>' > /var/www/htdocs/php.example/info.php
```

## 安全说明

- [httpd(8)](https://man.openbsdhandbook.com/httpd.8/) 以 `_httpd` 用户运行，默认 **chroot 到 /var/www**。`server`/`location` 上下文中的路径相对于 chroot。`tls { certificate|key|ocsp }` 中的路径为绝对路径。
- 避免让不受信任的用户对已提供文件拥有写权限。使用 [chmod(1)](https://man.openbsdhandbook.com/chmod.1/) 与 [chown(8)](https://man.openbsdhandbook.com/chown.8/)。
- 在端口 80 上保留一个 HTTP 虚拟服务器，用于 ACME 续签与 301 重定向到 HTTPS。
- 在 [pf(4)](https://man.openbsdhandbook.com/pf.4/) 中开放所需端口：

  ```pf
  pass in on $ext_if proto tcp to (self) port { 80 443 }
  ```

## 服务管理与排障

使用 [rcctl(8)](https://man.openbsdhandbook.com/rcctl.8/) 管理守护进程：

```sh
# rcctl check httpd
# rcctl restart httpd
```

语法检查与日志：

```sh
# httpd -n
# tail -f /var/www/logs/error.log
```

常见问题：

- ACME 验证返回 404：检查验证用的 `location` 块，确认 **/var/www/acme** 存在且可读。
- TLS 无法启动：确认证书与密钥路径可读；参见 [httpd.conf(5)](https://man.openbsdhandbook.com/httpd.conf.5/) 的 `tls` 块。
- FastCGI 错误：确认 chroot 内的后端套接字路径与权限。
