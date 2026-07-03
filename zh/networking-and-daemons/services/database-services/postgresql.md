# PostgreSQL

## 概述

**PostgreSQL** 是一款稳健的开源关系型数据库管理系统（RDBMS），以标准合规性、可扩展性与强一致性保证著称。它广泛用于需要可靠事务数据存储、通过 SQL 进行结构化查询，以及高级数据类型或函数的应用场景。

PostgreSQL 可通过 OpenBSD 的软件包系统安装，并通过 `rcctl(8)` 管理服务。它支持本地 Unix 套接字连接、TCP 网络、TLS、通过 `pg_hba.conf` 实现的细粒度访问控制，以及灵活的认证方式。

## 安装

安装 PostgreSQL 服务器软件包：

```sh
# pkg_add postgresql-server
```

此命令会提供 PostgreSQL 守护进程、客户端工具（`psql`、`createdb` 等）以及默认配置文件。

安装的版本通常为较新的稳定主发行版（如 PostgreSQL 15 或 16）。

## 初始化

创建 PostgreSQL 数据目录并初始化数据库集群：

```sh
# su - _postgresql
$ initdb -D /var/postgresql/data -U postgres -A md5
```

- `-D /var/postgresql/data` 指定数据库目录
- `-U postgres` 创建初始超级用户
- `-A md5` 配置密码认证

确保属主与权限正确：

```sh
# chown -R _postgresql:_postgresql /var/postgresql/data
```

## 启用服务

启用并启动 PostgreSQL 守护进程：

```sh
# rcctl enable postgresql
# rcctl start postgresql
```

守护进程以 `_postgresql` 身份运行，默认监听 Unix 套接字：

```text
/var/postgresql/data/.s.PGSQL.5432
```

## 配置

主配置文件为：

```text
/var/postgresql/data/postgresql.conf
```

常用调整：

```conf
listen_addresses = 'localhost'
port = 5432
max_connections = 100
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql.log'
```

修改后重启服务：

```sh
# rcctl restart postgresql
```

## 访问控制

客户端认证由以下文件管理：

```text
/var/postgresql/data/pg_hba.conf
```

默认条目只允许本地套接字连接。要通过 TCP 启用基于密码的认证：

```
# 类型     数据库    用户       地址           方法
host       all       all        127.0.0.1/32    md5
host       all       all        ::1/128         md5
```

重新加载配置：

```sh
# su - _postgresql -c "pg_ctl reload -D /var/postgresql/data"
```

## 创建用户与数据库

切换到 `_postgresql` 用户：

```sh
# su - _postgresql
```

创建用户与数据库：

```sh
$ createuser appuser -P
$ createdb appdb -O appuser
```

连接数据库：

```sh
$ psql -U appuser -d appdb
```

列出数据库：

```sql
\l
```

列出用户：

```sql
\du
```

## TLS 支持（可选）

1. 生成服务器密钥与证书：

```sh
# mkdir -p /var/postgresql/data/certs
# openssl req -x509 -newkey rsa:4096 -keyout /var/postgresql/data/certs/server.key \
  -out /var/postgresql/data/certs/server.crt -days 365 -nodes
# chmod 600 /var/postgresql/data/certs/server.key
# chown _postgresql:_postgresql /var/postgresql/data/certs/server.*
```

2. 更新 **postgresql.conf**：

```conf
ssl = on
ssl_cert_file = 'certs/server.crt'
ssl_key_file = 'certs/server.key'
```

3. 重启 PostgreSQL：

```sh
# rcctl restart postgresql
```

4. 客户端必须以 `sslmode=require` 连接：

```sh
$ psql "host=127.0.0.1 dbname=appdb user=appuser sslmode=require"
```

## 远程访问（可选）

要允许 TCP 连接：

1. 在 **postgresql.conf** 中：

```conf
listen_addresses = '*'
```

2. 在 **pg_hba.conf** 中：

```
host all all 192.0.2.0/24 md5
```

3. 在 **pf.conf** 中放行 5432 端口：

```pf
pass in on $int_if proto tcp from 192.0.2.0/24 to port 5432
```

4. 重启服务：

```sh
# rcctl restart postgresql
```

## 日志与诊断

若启用 `logging_collector`，日志会写入 **/var/postgresql/data/log/**。

查看日志：

```sh
# tail -f /var/postgresql/data/log/postgresql.log
```

查看服务器与连接状态：

```sql
SELECT version();
SELECT * FROM pg_stat_activity;
```

检查配置值：

```sql
SHOW config_file;
SHOW listen_addresses;
```
