# relayd

## 概述

**relayd(8)** 是 OpenBSD 原生的应用层代理与过滤守护进程。它支持：

- 反向代理（HTTP/HTTPS）
- TLS 终止，可选再次加密
- 带主动健康检查的 HTTP/HTTPS 负载均衡
- 七层过滤、头部改写与重定向

relayd 包含于 OpenBSD 基本系统，与 `pf(4)` 和 `httpd(8)` 紧密集成，通过 **/etc/relayd.conf** 配置。

需要安全地对外提供 HTTP 或 HTTPS 服务、在应用层过滤流量、或将请求分发到多个后端时，可使用 relayd。

## 常见用例

- 终止 TLS，并将 HTTP 转发到本地后端
- 过滤并将入站 HTTPS 转发到多台应用服务器
- 将 FastCGI 流量负载均衡到 PHP-FPM 套接字
- 以独立域名与 TLS 密钥实现虚拟主机

## 基本 TLS 终止反向代理

relayd 最常见的用例是接收 TLS 连接，再以未加密方式转发到后端。

### 示例：面向本地 httpd(8) 的 TLS 前端

```
relay "tlsproxy" {
    listen on egress port 443 tls
        tls certificate "/etc/ssl/example.org.fullchain.pem"
        tls key "/etc/ssl/private/example.org.key"

    forward to 127.0.0.1 port 8080
}
```

后端上，httpd(8) 应在 localhost 端口 8080 监听：

```
server "example.org" {
    listen on 127.0.0.1 port 8080
    root "/htdocs/example"
}
```

启动两个服务：

```sh
# rcctl enable relayd httpd
# rcctl start relayd httpd
```

在 **pf.conf** 中放行端口 443：

```pf
pass in on egress proto tcp to port 443
```

## 带负载均衡的后端池

relayd 可使用 **table** 将流量分发到多个后端主机：

```pf
table <webservers> { 192.0.2.10, 192.0.2.11, 192.0.2.12 }

relay "cluster" {
    listen on egress port 80
    forward to <webservers> port 80
}
```

默认通过 TCP 连接检查监控后端。

使用 HTTP 健康检查：

```pf
table <webservers> {
    192.0.2.10 check http "/" code 200
    192.0.2.11 check http "/status" code 200
}
```

查看实时状态：

```sh
# relayctl show hosts
```

## 将 HTTP 重定向到 HTTPS

relayd 支持内联 HTTP 重定向：

```
relay "http-redirect" {
    listen on egress port 80
    protocol "redirect"
}

protocol "redirect" {
    match request path "*" forward to "https://example.org"
}
```

该配置在端口 80 监听，并将所有 HTTP 流量重定向到 HTTPS。

## 配合 FastCGI 使用 relayd

要通过 httpd(8) 提供 FastCGI 后端（例如 PHP-FPM），relayd 可将请求转发到 Unix 套接字：

```pf
table <phpfpm> { socket "/run/php-fpm.sock" }

relay "fcgi" {
    listen on 127.0.0.1 port 9000
    forward to <phpfpm>
}
```

在 **httpd.conf** 中：

```
server "site" {
    listen on egress port 80
    root "/htdocs/site"
    location "*.php" {
        fastcgi socket "127.0.0.1" port 9000
    }
}
```

注：多数情况下，httpd(8) 可不经 relayd 直接连接到套接字。

## 日志与监控

在 **/etc/relayd.conf** 中启用日志：

```
log updates
log state
```

通过 syslog 跟踪日志：

```sh
# tail -f /var/log/daemon
```

查询运行时状态：

```sh
# relayctl show sessions
# relayctl show hosts
# relayctl statistics
```

不停机重载配置：

```sh
# rcctl reload relayd
```

## 安全注意事项

- relayd 以 `_relayd` 用户运行，不 chroot
- TLS 私钥须对 `_relayd` 可读，通常权限为 640、属主 `root:_relayd`
- 确保仅预期 IP 可访问 TLS 端口
- 配合 `pf(4)` 实现完整流量过滤

## pf.conf 规则示例

```pf
block in all
pass in on egress proto tcp from any to (self) port 443
pass in on egress proto tcp from any to (self) port 80
```

仅限内部访问的中继可使用回环或 RFC1918 地址绑定。

## 服务管理

开机启用并通过 `rcctl` 管理：

```sh
# rcctl enable relayd
# rcctl start relayd
# rcctl check relayd
```

检查语法错误：

```sh
# relayd -n -f /etc/relayd.conf
```

重载配置：

```sh
# rcctl reload relayd
```
