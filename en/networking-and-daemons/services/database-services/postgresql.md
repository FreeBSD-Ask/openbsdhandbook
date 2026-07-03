# PostgreSQL

## Synopsis

**PostgreSQL** is a robust, open-source relational database management system (RDBMS) known for its standards compliance, extensibility, and strong consistency guarantees. It is widely used in applications that require reliable transactional data storage, structured queries via SQL, and advanced data types or functions.

PostgreSQL is available via OpenBSD’s package system and integrates with `rcctl(8)` for service management. It supports local Unix socket connections, TCP networking, TLS, fine-grained access control via `pg_hba.conf`, and flexible authentication methods.

## Installation

Install the PostgreSQL server package:

```sh
# pkg_add postgresql-server
```

This provides the PostgreSQL daemon, client tools (`psql`, `createdb`, etc.), and default configuration files.

The installed version is typically a recent, stable major release (e.g., PostgreSQL 15 or 16).

## Initialization

Create the PostgreSQL data directory and initialize the database cluster:

```sh
# su - _postgresql
$ initdb -D /var/postgresql/data -U postgres -A md5
```

- `-D /var/postgresql/data` specifies the database directory
- `-U postgres` creates the initial superuser
- `-A md5` configures password authentication

Ensure ownership and permissions are correct:

```sh
# chown -R _postgresql:_postgresql /var/postgresql/data
```

## Enabling the Service

Enable and start the PostgreSQL daemon:

```sh
# rcctl enable postgresql
# rcctl start postgresql
```

The daemon runs as `_postgresql` and listens on a Unix socket by default:

```text
/var/postgresql/data/.s.PGSQL.5432
```

## Configuration

The main configuration file is:

```text
/var/postgresql/data/postgresql.conf
```

Common adjustments:

```conf
listen_addresses = 'localhost'
port = 5432
max_connections = 100
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql.log'
```

Restart the service after changes:

```sh
# rcctl restart postgresql
```

## Access Control

Client authentication is managed by:

```text
/var/postgresql/data/pg_hba.conf
```

Default entries allow only local socket connections. To enable password-based authentication over TCP:

```
# TYPE     DATABASE  USER       ADDRESS         METHOD
host       all       all        127.0.0.1/32    md5
host       all       all        ::1/128         md5
```

Reload configuration:

```sh
# su - _postgresql -c "pg_ctl reload -D /var/postgresql/data"
```

## Creating Users and Databases

Switch to the `_postgresql` user:

```sh
# su - _postgresql
```

Create a user and database:

```sh
$ createuser appuser -P
$ createdb appdb -O appuser
```

To connect:

```sh
$ psql -U appuser -d appdb
```

List databases:

```sql
\l
```

List users:

```sql
\du
```

## TLS Support (Optional)

1. Generate server key and certificate:

```sh
# mkdir -p /var/postgresql/data/certs
# openssl req -x509 -newkey rsa:4096 -keyout /var/postgresql/data/certs/server.key \
  -out /var/postgresql/data/certs/server.crt -days 365 -nodes
# chmod 600 /var/postgresql/data/certs/server.key
# chown _postgresql:_postgresql /var/postgresql/data/certs/server.*
```

2. Update `postgresql.conf`:

```conf
ssl = on
ssl_cert_file = 'certs/server.crt'
ssl_key_file = 'certs/server.key'
```

3. Restart PostgreSQL:

```sh
# rcctl restart postgresql
```

4. Clients must connect with `sslmode=require`:

```sh
$ psql "host=127.0.0.1 dbname=appdb user=appuser sslmode=require"
```

## Remote Access (Optional)

To allow TCP connections:

1. In `postgresql.conf`:

```conf
listen_addresses = '*'
```

2. In `pg_hba.conf`:

```
host all all 192.0.2.0/24 md5
```

3. In `pf.conf`, allow port 5432:

```pf
pass in on $int_if proto tcp from 192.0.2.0/24 to port 5432
```

4. Restart the service:

```sh
# rcctl restart postgresql
```

## Logging and Diagnostics

Logs are written to `/var/postgresql/data/log/` if `logging_collector` is enabled.

To inspect logs:

```sh
# tail -f /var/postgresql/data/log/postgresql.log
```

Inspect server and connection state:

```sql
SELECT version();
SELECT * FROM pg_stat_activity;
```

Check configuration values:

```sql
SHOW config_file;
SHOW listen_addresses;
```
