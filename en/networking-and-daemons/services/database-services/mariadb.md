# MariaDB

## Synopsis

**MariaDB** is an open source relational database system that is a drop-in replacement for MySQL. It is used for structured data storage and access by web applications, scripts, and services. MariaDB provides SQL-based access to tabular data, user privilege management, replication, and ACID compliance.

MariaDB is **available on OpenBSD via the ports and packages system** and is the recommended way to deploy a MySQL-compatible server.

### Why MySQL Is Not Included

OpenBSD **does not include Oracle MySQL** in its ports tree due to licensing constraints, lack of portability, and maintenance overhead. Oracle MySQL depends on build infrastructure and libraries that are either incompatible with or unacceptable under OpenBSD’s policies.

**MariaDB**, on the other hand, is a community-maintained fork of MySQL and is kept portable, open, and free of licensing encumbrances. It is therefore the supported and available MySQL-compatible option on OpenBSD.

There is **no support for Percona Server** or Oracle-provided MySQL in OpenBSD’s package system.

## Installation

Install MariaDB from packages:

```sh
# pkg_add mariadb-server
```

This installs the server daemon, utilities such as `mysql` and `mysqldump`, and the default configuration file.

The installed version is typically a recent stable release (e.g., 10.6 or newer).

## Configuration

MariaDB uses a configuration file located at:

```text
/etc/my.cnf
```

A minimal configuration example:

```ini
[mysqld]
datadir=/var/mysql
socket=/var/mysql/mysql.sock
user=_mysql
bind-address=127.0.0.1

[client]
socket=/var/mysql/mysql.sock
```

Create and initialize the data directory:

```sh
# mariadb-install-db
```

This creates the system database under `/var/mysql`, sets permissions, and populates the initial schema.

## Enabling the Service

Enable and start the daemon:

```sh
# rcctl enable mysqld
# rcctl start mysqld
```

You may now connect using the `mysql` client:

```sh
# mysql -u root
```

To stop the server:

```sh
# rcctl stop mysqld
```

## Securing the Installation

MariaDB does not enforce authentication on `root@localhost` by default. To restrict access:

1. Run the security script:

```sh
# mariadb-secure-installation
```

This allows:

- Set a password for `root@localhost`
- Remove anonymous users
- Disallow remote root login
- Remove the test database

2. Restart the service:

```sh
# rcctl restart mysqld
```

3. Test login:

```sh
# mysql -u root -p
```

## User and Database Management

Create a new database:

```sql
CREATE DATABASE appdb;
```

Create a user and grant privileges:

```sql
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'secretpass';
GRANT ALL PRIVILEGES ON appdb.* TO 'appuser'@'localhost';
FLUSH PRIVILEGES;
```

List existing databases:

```sql
SHOW DATABASES;
```

Connect to a specific database:

```sh
$ mysql -u appuser -p appdb
```

## Enabling Remote Access (Optional)

To allow remote TCP access, modify `/etc/my.cnf`:

```ini
[mysqld]
bind-address=0.0.0.0
```

Then use `GRANT` to permit access:

```sql
GRANT ALL ON appdb.* TO 'appuser'@'192.0.2.10' IDENTIFIED BY 'secretpass';
```

Restrict port 3306 using `pf(4)`:

```pf
pass in on $int_if proto tcp from 192.0.2.0/24 to port 3306
```

## TLS Support (Optional)

To enable encrypted client connections:

1. Generate server certificate and key:

```sh
# mkdir -p /etc/ssl/mariadb
# openssl req -x509 -newkey rsa:4096 -keyout /etc/ssl/mariadb/server.key \
  -out /etc/ssl/mariadb/server.crt -days 365 -nodes
```

2. Add to `/etc/my.cnf`:

```ini
[mysqld]
ssl-ca=/etc/ssl/mariadb/server.crt
ssl-cert=/etc/ssl/mariadb/server.crt
ssl-key=/etc/ssl/mariadb/server.key
```

3. Restart the daemon:

```sh
# rcctl restart mysqld
```

4. Test from client:

```sh
$ mysql --ssl -u appuser -p
```

## Logging and Diagnostics

MariaDB logs to `/var/mysql/` by default:

- `mysql.err`: startup and runtime errors
- `mysqld.log`: general queries and slow queries (if enabled)

To follow logs:

```sh
# tail -f /var/mysql/mysql.err
```

Query the server status:

```sql
SHOW STATUS;
SHOW VARIABLES;
```
