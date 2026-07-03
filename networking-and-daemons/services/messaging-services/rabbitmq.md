# RabbitMQ

## 概述

**RabbitMQ** 是实现 AMQP（高级消息队列协议，Advanced Message Queuing Protocol）的消息代理。它通过队列、主题与发布订阅机制，解耦服务并分发任务。

RabbitMQ 支持持久化队列、确认机制、集群、TLS 加密以及包括 Web 管理在内的各类插件。

RabbitMQ 可通过软件包系统在 OpenBSD 上获取。它运行于 **Erlang 运行时**之上，通过 TCP 通信（AMQP 默认端口 5672，管理接口可选 HTTPS 端口 15671）。

## 安装

安装 RabbitMQ 及其 Erlang 运行时依赖：

```sh
# pkg_add rabbitmq
```

此命令会安装：

- `rabbitmq-server`：主守护进程
- `rabbitmqctl`：管理命令行工具
- `rabbitmq-plugins`：插件管理工具
- 默认配置目录：**/etc/rabbitmq/**

RabbitMQ 没有可写数据目录时将拒绝运行。

## 服务初始化

启动 RabbitMQ 服务前须先初始化：

```sh
# install -d -o _rabbitmq -g _rabbitmq /var/rabbitmq
```

启用并启动服务：

```sh
# rcctl enable rabbitmq
# rcctl start rabbitmq
```

检查状态：

```sh
# rabbitmqctl status
```

默认情况下，守护进程监听：

- TCP 端口 **5672**（AMQP）
- TCP 端口 **25672**（Erlang 分布式通信，未组建集群时不使用）

## 配置

RabbitMQ 在 **/etc/rabbitmq/** 下使用基于目录的配置。

主要有两个文件：

- **/etc/rabbitmq/rabbitmq-env.conf**：shell 环境变量覆盖
- **/etc/rabbitmq/rabbitmq.conf**：主配置文件（INI 风格格式）

**/etc/rabbitmq/rabbitmq.conf** 示例：

```conf
# 禁用回环限制
loopback_users.guest = false

# 默认监听器
listeners.tcp.default = 127.0.0.1:5672

# 启用 TLS
# listeners.ssl.default = 0.0.0.0:5671
# ssl_options.certfile = /etc/ssl/rabbitmq/server.crt
# ssl_options.keyfile = /etc/ssl/rabbitmq/server.key
# ssl_options.cacertfile = /etc/ssl/rabbitmq/ca.crt
```

更改配置后重启 RabbitMQ：

```sh
# rcctl restart rabbitmq
```

## 用户与访问控制

创建新用户与虚拟主机：

```sh
# rabbitmqctl add_user myuser secretpass
# rabbitmqctl add_vhost myvhost
# rabbitmqctl set_permissions -p myvhost myuser ".*" ".*" ".*"
```

为安全起见，禁用默认的 `guest` 用户：

```sh
# rabbitmqctl delete_user guest
```

列出用户与虚拟主机：

```sh
# rabbitmqctl list_users
# rabbitmqctl list_vhosts
```

## Web 管理插件

启用 RabbitMQ 管理界面：

```sh
# rabbitmq-plugins enable rabbitmq_management
# rcctl restart rabbitmq
```

此操作将暴露：

- **端口 15672**：Web 界面（HTTP）
- **端口 15671**：Web 界面（HTTPS，若配置了 TLS）

访问 <http://localhost:15672>，使用已创建的用户凭据登录。

创建管理员用户：

```sh
# rabbitmqctl add_user adminuser strongpass
# rabbitmqctl set_user_tags adminuser administrator
```

## TLS 支持

启用 TLS，在 **/etc/rabbitmq/rabbitmq.conf** 中添加：

```conf
listeners.ssl.default = 0.0.0.0:5671
ssl_options.certfile = /etc/ssl/rabbitmq/server.crt
ssl_options.keyfile = /etc/ssl/rabbitmq/server.key
ssl_options.cacertfile = /etc/ssl/rabbitmq/ca.crt
ssl_options.verify = verify_peer
ssl_options.fail_if_no_peer_cert = true
```

确保权限正确：

```sh
# chown _rabbitmq:_rabbitmq /etc/ssl/rabbitmq/*
# chmod 640 /etc/ssl/rabbitmq/*
```

然后重启：

```sh
# rcctl restart rabbitmq
```

## 防火墙配置

若在 localhost 之外运行，在 `pf.conf` 中放行端口：

```pf
pass in on $int_if proto tcp from trusted_nets to port { 5672 5671 15672 }
```

除非已启用 TLS，否则避免将 RabbitMQ 暴露到不可信网络。

## 服务管理

标准服务控制命令：

```sh
# rcctl enable rabbitmq
# rcctl start rabbitmq
# rcctl check rabbitmq
# rcctl restart rabbitmq
```

查看运行时信息：

```sh
# rabbitmqctl status
# rabbitmqctl list_queues
# rabbitmqctl list_connections
```

## 日志

RabbitMQ 日志写入：

```text
/var/log/rabbitmq/
```

使用 `tail -f` 跟踪日志输出：

```sh
# tail -f /var/log/rabbitmq/rabbit@$(hostname -s).log
```
