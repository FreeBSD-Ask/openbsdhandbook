# nginx

## 概述

**nginx** 是一款高性能的 HTTP 服务器与反向代理，以高效、内存占用低、功能丰富著称。它支持静态内容、TLS、FastCGI 与 SCGI 后端、反向代理、负载均衡以及基本认证。

nginx **不包含于 OpenBSD 基本系统**，但可通过软件包获取。

### OpenBSD 上的 Web 服务器历史

OpenBSD 历史上在基本系统中附带 **Apache 1.3**，后因复杂性与安全顾虑被移除。OpenBSD 5.6 至 5.8 期间曾短暂将 **nginx** 作为默认 Web 服务器。后被移除，改用更简洁、更注重安全的 **httpd(8)**，现为基本系统默认。

nginx 仍可通过 `pkg_add` 获取，适合涉及 TLS 终止、静态文件托管、FastCGI 应用服务与反向代理的现代负载。

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
| 推荐用途 | 静态或 FastCGI 托管 | 反向代理、TLS 卸载 | 动态内容、遗留站点 |

基本系统的简洁与静态内容场景可使用 **httpd(8)**，高效的 TLS 或 FastCGI 代理可使用 **nginx**，复杂动态配置可使用 **Apache**。

## 安装

从软件包安装 nginx：

```sh
# pkg_add nginx
```

该命令安装：

- **/etc/nginx/nginx.conf** — 主配置文件
- **/usr/local/sbin/nginx** — 守护进程二进制
- `rc.d` 服务脚本，供 `rcctl(8)` 使用

## 基础配置

默认配置文件安装于 **/etc/nginx/nginx.conf**。

静态文件托管的最小示例：

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

创建文档根目录与测试文件：

```sh
# mkdir -p /htdocs
# echo "OK" > /htdocs/index.html
# chown -R www:www /htdocs
```

## 启动 nginx

启用并启动服务：

```sh
# rcctl enable nginx
# rcctl start nginx
```

测试或重载配置：

```sh
# nginx -t
# nginx -s reload
```

进程以 `_nginx` 用户运行。

## TLS 配置

要启用 HTTPS，获取证书（例如通过 `acme-client`）并调整配置：

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

别忘了在 **pf.conf** 中开放端口 443：

```pf
pass in on $int_if proto tcp from any to (self) port 443
```

## FastCGI 支持（PHP 等）

要提供动态内容（例如通过 `php-fpm`）：

1. 安装 `php` 与 `php-fpm`：

```sh
# pkg_add php php-fpm
# rcctl enable php_fpm
# rcctl start php_fpm
```

2. 在 nginx 中启用 FastCGI 传递：

```
location ~ \.php$ {
    root           /htdocs;
    fastcgi_pass   unix:/run/php-fpm.sock;
    fastcgi_index  index.php;
    include        fastcgi_params;
    fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
```

确保文档根目录中存在 PHP 文件：

```sh
# echo "<?php phpinfo(); ?>" > /htdocs/index.php
```

## 反向代理示例

nginx 可将请求代理到后端服务，例如 Python 或 Node.js 应用：

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

此配置常用于应用前端、TLS 终止或缓存。

## 日志与诊断

访问与错误日志在 `http` 块中定义：

```nginx
access_log  /var/log/nginx/access.log;
error_log   /var/log/nginx/error.log;
```

跟踪日志：

```sh
# tail -f /var/log/nginx/access.log
```

使用 `nginx -t` 校验语法，更改后用 `rcctl restart nginx` 重启。

## 安全实践

- 以 `_nginx` 用户运行 nginx（默认）。
- 确保 **/htdocs** 中的文件属主为 `www`，且不可全局写。
- 使用 `try_files` 避免路径遍历。
- 未启用 TLS 前，不要将 nginx 暴露到互联网。
- 若使用反向代理，校验所有代理头。

**pf.conf** 规则示例：

```pf
block in proto tcp from any to (self) port { 80 443 }
pass in on $int_if proto tcp from 192.0.2.0/24 to (self) port { 80 443 }
```
