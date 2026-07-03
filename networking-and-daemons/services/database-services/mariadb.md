# MariaDB

## 概述

**MariaDB** 是开源关系型数据库系统，可作为 MySQL 的直接替代。Web 应用、脚本与服务可用它存储和访问结构化数据。MariaDB 提供 SQL 访问表格数据、用户权限管理、复制功能，并满足 ACID 合规要求。

MariaDB **可通过 OpenBSD 的 ports 与软件包系统获取**，是部署 MySQL 兼容服务器的推荐方式。

### 为何不收录 MySQL

OpenBSD 的 ports 树中**未收录 Oracle MySQL**，原因在于许可证限制、可移植性不足以及维护成本高。Oracle MySQL 依赖的构建工具与库要么与 OpenBSD 策略不兼容，要么不被接受。

MariaDB 是 MySQL 的社区维护分支，保持了可移植性、开放性，且无许可证负担。因此 OpenBSD 上支持的 MySQL 兼容方案就是 MariaDB。

OpenBSD 的软件包系统**不支持** Percona Server 与 Oracle 提供的 MySQL。

## 安装

通过软件包安装 MariaDB：

```sh
# pkg_add mariadb-server
```

此命令会安装服务器守护进程、`mysql` 与 `mysqldump` 等工具，以及默认配置文件。

安装的版本通常为较新的稳定发行版（如 10.6 或更高）。

## 配置

MariaDB 使用的配置文件位于：

```text
/etc/my.cnf
```

最小化配置示例：

```ini
[mysqld]
datadir=/var/mysql
socket=/var/mysql/mysql.sock
user=_mysql
bind-address=127.0.0.1

[client]
socket=/var/mysql/mysql.sock
```

创建并初始化数据目录：

```sh
# mariadb-install-db
```

此命令会在 **/var/mysql** 下创建系统数据库、设置权限，并填充初始 schema。

## 启用服务

启用并启动守护进程：

```sh
# rcctl enable mysqld
# rcctl start mysqld
```

现在可以使用 `mysql` 客户端连接：

```sh
# mysql -u root
```

停止服务器：

```sh
# rcctl stop mysqld
```

## 加固安装

MariaDB 默认不对 `root@localhost` 强制认证。要限制访问：

1. 运行安全脚本：

```sh
# mariadb-secure-installation
```

该脚本支持：

- 为 `root@localhost` 设置密码
- 删除匿名用户
- 禁止 root 远程登录
- 删除 test 数据库

2. 重启服务：

```sh
# rcctl restart mysqld
```

3. 测试登录：

```sh
# mysql -u root -p
```

## 用户与数据库管理

创建新数据库：

```sql
CREATE DATABASE appdb;
```

创建用户并授予权限：

```sql
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'secretpass';
GRANT ALL PRIVILEGES ON appdb.* TO 'appuser'@'localhost';
FLUSH PRIVILEGES;
```

列出现有数据库：

```sql
SHOW DATABASES;
```

连接到指定数据库：

```sh
$ mysql -u appuser -p appdb
```

## 启用远程访问（可选）

要允许远程 TCP 访问，修改 **/etc/my.cnf**：

```ini
[mysqld]
bind-address=0.0.0.0
```

然后用 `GRANT` 授权访问：

```sql
GRANT ALL ON appdb.* TO 'appuser'@'192.0.2.10' IDENTIFIED BY 'secretpass';
```

用 `pf(4)` 限制 3306 端口：

```pf
pass in on $int_if proto tcp from 192.0.2.0/24 to port 3306
```

## TLS 支持（可选）

要启用加密的客户端连接：

1. 生成服务器证书与密钥：

```sh
# mkdir -p /etc/ssl/mariadb
# openssl req -x509 -newkey rsa:4096 -keyout /etc/ssl/mariadb/server.key \
  -out /etc/ssl/mariadb/server.crt -days 365 -nodes
```

2. 在 **/etc/my.cnf** 中添加：

```ini
[mysqld]
ssl-ca=/etc/ssl/mariadb/server.crt
ssl-cert=/etc/ssl/mariadb/server.crt
ssl-key=/etc/ssl/mariadb/server.key
```

3. 重启守护进程：

```sh
# rcctl restart mysqld
```

4. 从客户端测试：

```sh
$ mysql --ssl -u appuser -p
```

## 日志与诊断

MariaDB 默认将日志写入 **/var/mysql/**：

- **mysql.err**：启动与运行时错误
- **mysqld.log**：常规查询与慢查询（若启用）

查看日志：

```sh
# tail -f /var/mysql/mysql.err
```

查询服务器状态：

```sql
SHOW STATUS;
SHOW VARIABLES;
```
