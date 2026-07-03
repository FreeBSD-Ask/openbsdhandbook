# memcached

## 概述

**memcached** 是高性能内存键值存储，主要用于缓存数据，以减轻动态 Web 应用中数据库的负载与延迟。它提供简单的 TCP 协议来设置和获取短期有效的值，常被 Django、Rails、PHP 等框架使用。

memcached 可通过 OpenBSD 的软件包系统安装，并通过 `rcctl(8)` 管理服务。

与 Redis 不同，memcached 不支持持久化，仅支持非常简单的键值操作。

## 安装

通过软件包系统安装 memcached：

```sh
# pkg_add memcached
```

此命令会安装：

- **/usr/local/bin/memcached**——守护进程
- **/etc/rc.d/memcached**——`rcctl` 控制脚本

## 配置

memcached 通常通过命令行选项配置。这些选项可通过编辑 `rcctl` flags 持久化设置：

```sh
# rcctl set memcached flags "-m 64 -l 127.0.0.1 -p 11211"
```

常用选项：

- `-m 64`——分配 64 MB 内存
- `-l 127.0.0.1`——仅绑定到 localhost
- `-p 11211`——默认 TCP 端口

应用更改：

```sh
# rcctl enable memcached
# rcctl start memcached
```

## 测试基本用法

使用 `telnet(1)` 客户端测试 memcached：

```sh
$ telnet localhost 11211
Trying 127.0.0.1...
Connected to localhost.
set testkey 0 300 5
hello
STORED
get testkey
VALUE testkey 0 5
hello
END
```

- `set <key> <flags> <ttl> <bytes>`
- `get <key>`

键在指定的 TTL（秒）后过期。

退出：

```
quit
```

## 配合客户端使用

memcached 受多种编程语言支持。示例：

- PHP：`memcached` 或 `memcache` 扩展
- Python：`python3-memcached` 或 `pymemcache`
- Ruby：`dalli` gem

Python 使用示例：

```python
import memcache
mc = memcache.Client(['127.0.0.1:11211'])
mc.set('greeting', 'hello', time=300)
print(mc.get('greeting'))
```

## 安全注意事项

memcached **不支持认证与 TLS**，只能在受信任的网络中运行。

默认应绑定到 `127.0.0.1`。若必须远程访问：

1. 调整 flags：

```sh
# rcctl set memcached flags "-m 128 -l 192.0.2.1 -p 11211"
```

2. 用 `pf(4)` 限制访问：

```pf
pass in on $int_if proto tcp from 192.0.2.0/24 to port 11211
block in on $int_if proto tcp from any to port 11211
```

3. 重新加载 `pf` 并重启 memcached：

```sh
# pfctl -f /etc/pf.conf
# rcctl restart memcached
```

切勿将 memcached 暴露到公共互联网。

## 服务管理

启动并设置开机自启：

```sh
# rcctl enable memcached
# rcctl start memcached
```

停止或重启：

```sh
# rcctl stop memcached
# rcctl restart memcached
```

查看状态：

```sh
# rcctl check memcached
```

## 日志

memcached 将日志记录到 syslog。查看日志：

```sh
# tail -f /var/log/daemon
```

要提高详细程度，可在 flags 中添加 `-v` 或 `-vv`：

```sh
# rcctl set memcached flags "-m 64 -v"
# rcctl restart memcached
```
