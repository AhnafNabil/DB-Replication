# MySQL Replication and ProxySQL

## Project Structure

```
DB-Replication/
├── docker-compose.yml
├── master/
│   ├── init.sql
│   └── my.cnf
├── proxysql/
│   ├── proxysql.cnf
│   └── data/
├── slave/
│   ├── init.sql
│   ├── my-slave1.cnf
│   └── my-slave2.cnf
```

## Files 

### master/init.sql

```bash
/* proxysql user */
CREATE USER IF NOT EXISTS 'monitor'@'%' IDENTIFIED BY 'monitor';

/* mysql exporter user */
CREATE USER IF NOT EXISTS 'exporter'@'%' IDENTIFIED BY 'password' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';

/* slave user */
CREATE USER IF NOT EXISTS 'slave_user'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' WITH GRANT OPTION;

FLUSH PRIVILEGES;


create table users
(
	id int auto_increment,
	name varchar(255) null,
	constraint users_pk
		primary key (id)
);

INSERT INTO users VALUES (1, 'dipanjal'), (2, 'shohan');
```

### master/my.cnf

```bash
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL
symbolic-links=0
default_authentication_plugin=mysql_native_password
lower_case_table_names = 1

bind-address            = 0.0.0.0
server-id               = 1
log_bin                 = /var/run/mysqld/mysql-bin.log
binlog_do_db            = sbtest

gtid_mode                = on
enforce_gtid_consistency = on
log_slave_updates        = on


general_log=1
general_log_file=/var/log/mysql/general.log

# Slow query settings:
slow_query_log=1
slow_query_log_file=/var/log/mysql/slow.log
long_query_time=0.5
```

### slave/init.sql

```bash
/* proxysql user */
CREATE USER IF NOT EXISTS 'monitor'@'%' IDENTIFIED BY 'monitor';

/* mysql exporter user */
CREATE USER IF NOT EXISTS 'exporter'@'%' IDENTIFIED BY 'password' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';

FLUSH PRIVILEGES;

/* start replication */
CHANGE MASTER TO MASTER_HOST='mysql-master',MASTER_USER='slave_user',MASTER_PASSWORD='password',MASTER_AUTO_POSITION=1;
START SLAVE;
```

### slave/my-slave1.cnf

```bash
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL
symbolic-links=0
default_authentication_plugin=mysql_native_password
lower_case_table_names = 1

bind-address            = 0.0.0.0
server-id               = 2
relay-log               = /var/run/mysqld/mysql-relay-bin.log
log_bin                 = /var/run/mysqld/mysql-bin.log
binlog_do_db            = sbtest

read_only = on

gtid_mode                = on
enforce_gtid_consistency = on
log_slave_updates        = on


general_log=1
general_log_file=/var/log/mysql/general.log

# Slow query settings:
slow_query_log=1
slow_query_log_file=/var/log/mysql/slow.log
long_query_time=0.5
```

### slave/my-slave2.cnf

```bash
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
secure-file-priv= NULL
symbolic-links=0
default_authentication_plugin=mysql_native_password
lower_case_table_names = 1

bind-address            = 0.0.0.0
server-id               = 3
relay-log               = /var/run/mysqld/mysql-relay-bin.log
log_bin                 = /var/run/mysqld/mysql-bin.log
binlog_do_db            = sbtest

read_only = on

gtid_mode                = on
enforce_gtid_consistency = on
log_slave_updates        = on

general_log=1
general_log_file=/var/log/mysql/general.log

# Slow query settings:
slow_query_log=1
slow_query_log_file=/var/log/mysql/slow.log
long_query_time=0.5
```

### proxysql/proxysql.cnf

```bash
datadir="/var/lib/proxysql"

admin_variables=
{
    admin_credentials="admin:admin;admin2:pass2"
    mysql_ifaces="0.0.0.0:6032"
    refresh_interval=2000
    stats_credentials="stats:admin"
}

mysql_variables=
{
    threads=4
    max_connections=2048
    default_query_delay=0
    default_query_timeout=36000000
    have_compress=true
    poll_timeout=2000
    interfaces="0.0.0.0:6033;/tmp/proxysql.sock"
    default_schema="information_schema"
    stacksize=1048576
    server_version="8.0"
    connect_timeout_server=10000
    monitor_history=60000
    monitor_connect_interval=200000
    monitor_ping_interval=200000
    ping_interval_server_msec=10000
    ping_timeout_server=200
    commands_stats=true
    sessions_sort=true
    monitor_username="monitor"
    monitor_password="monitor"
}

mysql_replication_hostgroups =
(
    { writer_hostgroup=10 , reader_hostgroup=20 , comment="host groups" }
)

mysql_servers =
(
    { address="mysql-master" , port=3306 , hostgroup=10, max_connections=100 , max_replication_lag = 5 },
    { address="mysql-slave1" , port=3306 , hostgroup=20, max_connections=100 , max_replication_lag = 5 },
    { address="mysql-slave2" , port=3306 , hostgroup=20, max_connections=100 , max_replication_lag = 5 }
)

mysql_query_rules =
(
    {
        rule_id=100
        active=1
        match_pattern="^SELECT .* FOR UPDATE"
        destination_hostgroup=10
        apply=1
    },
    {
        rule_id=200
        active=1
        match_pattern="^SELECT .*"
        destination_hostgroup=20
        apply=1
    },
    {
        rule_id=300
        active=1
        match_pattern=".*"
        destination_hostgroup=10
        apply=1
    }
)

mysql_users =
(
    { username = "root" , password = "password" , default_hostgroup = 10 , active = 1 }
)
```

### docker-compose.yml

```bash
version: '3'

services:

  mysql-master:
    image: mysql:5.7
    container_name: proxysql-mysql-replication-master
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: sbtest
    volumes:
      - ./master/my.cnf:/etc/mysql/my.cnf
      # - ./master/data:/var/lib/mysql
      - ./master/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - 3306:3306
    networks:
      - mysql_cluster_net
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-u", "root", "-ppassword"]
      timeout: 20s
      retries: 10

  mysql-slave1:
    image: mysql:5.7
    container_name: proxysql-mysql-replication-slave1
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: sbtest
    volumes: #
      - ./slave/my-slave1.cnf:/etc/mysql/my.cnf
      # - ./slave/data/slave1:/var/lib/mysql
      - ./slave/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - 3307:3306
    depends_on:
      - mysql-master
    networks:
      - mysql_cluster_net
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-u", "root", "-ppassword"]
      timeout: 20s
      retries: 10

  mysql-slave2:
    image: mysql:5.7
    container_name: proxysql-mysql-replication-slave2
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: sbtest
    volumes:
      - ./slave/my-slave2.cnf:/etc/mysql/my.cnf
      # - ./slave/data/slave2:/var/lib/mysql
      - ./slave/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - 3308:3306
    depends_on:
      - mysql-master
    networks:
      - mysql_cluster_net
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-u", "root", "-ppassword"]
      timeout: 20s
      retries: 10

  proxysql:
    image: proxysql/proxysql:2.0.12
    container_name: proxysql-mysql-replication-proxysql
    ports:
      - 6032:6032
      - 6033:6033
    volumes:
      - ./proxysql/proxysql.cnf:/etc/proxysql.cnf
      - ./proxysql/data:/var/lib/proxysql
    networks:
      - mysql_cluster_net
    depends_on: # the proxy will start after all the mysql instance has started
      - mysql-master
      - mysql-slave1
      - mysql-slave2
    healthcheck:
      # test: ["CMD", "proxysql_admin", "-u", "admin", "-padmin", "-h", "127.0.0.1", "-P", "6032", "--execute", "SELECT 1"]
      # mysql -h 0.0.0.0 -P 6032 -u admin2 -p -e 'select * from mysql_servers'
      test: ["CMD", "proxysql_admin", "-u", "admin2", "-ppass2", "-h", "127.0.0.1", "-P", "6032", "--execute", "SELECT 1"]
      timeout: 20s
      retries: 5

#creating a common L2 Bridge network for the Docker containers internal communication
networks:
  mysql_cluster_net:
    driver: bridge
```

## Verification Commands

### Check Master's Status

```bash
docker-compose exec mysql-master sh -c "export MYSQL_PWD=password; mysql -u root sbtest -e 'show master status\G'"
```

### Check Slave 1 Status

```bash
docker-compose exec mysql-slave1 sh -c "export MYSQL_PWD=password; mysql -u root sbtest -e 'show slave status\G'"
```

### Check Slave 2 Status

```bash
docker-compose exec mysql-slave2 sh -c "export MYSQL_PWD=password; mysql -u root sbtest -e 'show slave status\G'"
```

### Check Replication States

Install Mysql:

```bash
Sudo apt update
sudo apt install mysql-client -y
```

Now, check the replication state:

```bash
mysql -h 0.0.0.0 -P 6032 -u admin2 -p -e 'select * from mysql_servers'
```

**Expected Output:**

```bash
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname     | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 10           | mysql-master | 3306 | 0         | ONLINE | 1      | 0           | 100             | 5                   | 0       | 0              |         |
| 20           | mysql-slave2 | 3306 | 0         | ONLINE | 1      | 0           | 100             | 5                   | 0       | 0              |         |
| 20           | mysql-slave1 | 3306 | 0         | ONLINE | 1      | 0           | 100             | 5                   | 0       | 0              |         |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
``` 

All the Master and Slaves are Online and Synced up.