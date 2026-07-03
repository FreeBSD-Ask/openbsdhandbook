# Redis

## 概述

**Redis** 是内存键值存储，专为高性能、低延迟设计，支持字符串、哈希、集合、有序集合、流等多种数据结构。它支持可选的磁盘持久化、复制与 Lua 脚本。

Redis 常用于缓存、消息传递、任务队列与临时数据存储。

Redis **未包含在 OpenBSD 基本系统中**，但可通过软件包获取。Redis 服务器以守护进程（`redis-server`）形式运行，由 `rcctl(8)` 管理。

## 安装

安装 Redis 软件包：

```sh
# pkg_add redis
```

此命令会安装：

- `redis-server`：主守护进程
- `redis-cli`：命令行客户端
- 默认配置文件 **/etc/redis.conf**

## 配置

主配置文件为：

```text
/etc/redis.conf
```

最小化配置示例：

```sh
bind 127.0.0.1
port 6379

daemonize no
supervised auto
pidfile /var/run/redis.pid
logfile /var/log/redis.log

dir /var/redis
dbfilename dump.rdb
```

关键选项：

- `bind`：限制仅监听回环地址
- `port`：默认为 6379
- `daemonize`：设为 `no` 以兼容 `rcctl`
- `dir`：数据持久化的工作目录
- `logfile`：可选的文件日志（否则记录到 syslog）

若数据目录不存在，则创建：

```sh
# mkdir -p /var/redis
# chown _redis:_redis /var/redis
```

## 启用服务

启用并启动 Redis：

```sh
# rcctl enable redis
# rcctl start redis
```

查看状态：

```sh
# rcctl check redis
```

查看日志：

```sh
# tail -f /var/log/redis.log
```

## 基本用法

使用 Redis 客户端连接：

```sh
$ redis-cli
127.0.0.1:6379> SET keyname "hello"
OK
127.0.0.1:6379> GET keyname
"hello"
```

按 `CTRL+D` 或输入 `QUIT` 退出客户端。

## 持久化

Redis 有两种将数据集保存到磁盘的方式：

1. **RDB（快照）**——默认方式，使用 `dump.rdb`
2. **AOF（仅追加文件）**——记录每次写操作

在 **/etc/redis.conf** 中启用 AOF 持久化：

```
appendonly yes
appendfilename "appendonly.aof"
```

重启 Redis 使其生效：

```sh
# rcctl restart redis
```

可从 **/var/redis** 复制 `.rdb` 或 `.aof` 文件进行备份。

## 远程访问与安全

Redis 默认仅绑定 `127.0.0.1`。

要允许远程主机访问（不推荐，除非已加保护）：

1. 在 **/etc/redis.conf** 中修改绑定地址：

```
bind 127.0.0.1 192.0.2.1
```

2.（可选）设置密码：

```
requirepass "strongsecret"
```

3. 用 `pf(4)` 限制 6379 端口：

```pf
pass in on $int_if proto tcp from 192.0.2.0/24 to port 6379
```

4. 重启 Redis：

```sh
# rcctl restart redis
```

带认证访问：

```sh
$ redis-cli -a strongsecret
```

**重要**：OpenBSD 软件包中的 Redis 原生不支持 TLS（截至 7.8）。要代理 TLS，可使用 `relayd(8)` 或 `stunnel`。

## 服务管理

标准服务命令：

```sh
# rcctl enable redis
# rcctl start redis
# rcctl restart redis
# rcctl check redis
```
