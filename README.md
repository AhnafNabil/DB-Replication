# Setting Up MySQL Master-Slave Replication with ProxySQL

This lab provides a step-by-step guide to creating a production-ready MySQL replication cluster with ProxySQL for intelligent query routing and load balancing. You will set up a MySQL master-slave architecture with two read replicas and a ProxySQL instance to distribute read and write queries efficiently. Using Docker containers and GTID-based replication.

## Task Overview

**In this hands-on lab, you will:**
- Create a project structure for configuration files.
- Configure a MySQL master server to handle write operations.
- Set up two MySQL slave servers for read operations.
- Configure a ProxySQL instance for intelligent query routing and load balancing.
- Deploy the cluster using Docker Compose.
- Verify replication and ProxySQL functionality.
- Test the setup to ensure proper read/write query routing.

By the end of this lab, you will have a fully functional MySQL cluster with a master server, two slave servers, and a ProxySQL instance, all running in Docker containers and communicating over a bridge network.

## Key Concepts of MySQL Replication and ProxySQL

### Why MySQL Master-Slave Replication?
MySQL master-slave replication enables data to be copied from a master database to one or more slave databases. This setup improves read scalability, enhances availability, and supports failover scenarios by distributing read queries across slaves while directing all writes to the master.

### What is MySQL Master-Slave Replication?
MySQL replication is a process where data from a master database is replicated to one or more slave databases in real-time or near real-time. The master handles all write operations (INSERT, UPDATE, DELETE), while slaves can handle read operations (SELECT).

**Key features:**
- Asynchronous or semi-synchronous replication.
- Uses binary logs to track changes on the master.
- Supports Global Transaction Identifiers (GTID) for consistent and reliable replication.

### What is GTID?
GTID (Global Transaction Identifier) is a unique identifier assigned to each transaction on the master. It simplifies replication by allowing slaves to track and apply transactions in a consistent order, making failover and recovery easier.

### What is ProxySQL?
ProxySQL is a high-performance MySQL proxy that routes client queries to appropriate backend servers based on predefined rules. It provides load balancing, query caching, and failover capabilities, improving the efficiency and reliability of the database cluster.

**Key features:**
- Intelligent query routing (e.g., reads to slaves, writes to master).
- Health monitoring of backend servers.
- Support for high availability and scalability.

### How ProxySQL Works
- **Query Analysis:** ProxySQL inspects incoming queries and routes them based on user-defined rules (e.g., SELECT queries to slaves, other queries to the master).
- **Load Balancing:** Distributes read queries across multiple slaves to optimize performance.
- **Health Checks:** Monitors backend servers for availability and replication lag, rerouting queries if a server fails.

## Folder Structure

```
mysql-replication-lab/
├── docker-compose.yml
├── master/
│   ├── init.sql
│   └── my.cnf
├── proxysql/
│   ├── proxysql.cnf
├── slave/
│   ├── init.sql
│   ├── my-slave1.cnf
│   └── my-slave2.cnf
```

## Step 1: Set Up Project Structure

Create a directory structure to organize configuration files for the MySQL master, slaves, and ProxySQL.

```bash
mkdir mysql-replication-lab
cd mysql-replication-lab
```

Create the folders for the project.
```bash
mkdir -p master slave proxysql
```

## Step 2: Configure MySQL Master Server

Set up the MySQL master server to handle all write operations and enable binary logging for replication.

### 2.1 Create Master Configuration File
Create a configuration file for the master MySQL server named `my.cnf`.

```ini
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

slow_query_log=1
slow_query_log_file=/var/log/mysql/slow.log
long_query_time=0.5
```

**Key Configuration Explained:**
- `server-id=1`: Unique identifier for the master in the replication cluster.
- `log_bin`: Enables binary logging for replication.
- `binlog_do_db=sbtest`: Restricts replication to the `sbtest` database.
- `GtID_mode=on`: Enables GTID for reliable replication.
- `bind-address=0.0.0.0`: Allows connections from other containers.

### 2.2 Create Master Initialization Script
Create an initialization script to set up users and sample data named `init.sql`.

```sql
CREATE USER IF NOT EXISTS 'monitor'@'%' IDENTIFIED BY 'monitor';

CREATE USER IF NOT EXISTS 'exporter'@'%' IDENTIFIED BY 'password' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';

CREATE USER IF NOT EXISTS 'slave_user'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' WITH GRANT OPTION;

FLUSH PRIVILEGES;

CREATE TABLE users (
    id INT AUTO_INCREMENT,
    name VARCHAR(255) NULL,
    CONSTRAINT users_pk PRIMARY KEY (id)
);

INSERT INTO users VALUES (1, 'dipanjal'), (2, 'shohan');
```

**Script Purpose:**
- Creates `monitor` and `exporter` users for ProxySQL monitoring.
- Creates `slave_user` for replication.
- Sets up a sample `users` table with initial data that will be replicated to slaves.

## Step 3: Configure MySQL Slave Servers

Set up two MySQL slave servers to handle read queries, configured to replicate data from the master.

### 3.1 Create Slave Initialization Script
Create an initialization script for both slaves to configure replication named `init.sql`.

```sql
CREATE USER IF NOT EXISTS 'monitor'@'%' IDENTIFIED BY 'monitor';

CREATE USER IF NOT EXISTS 'exporter'@'%' IDENTIFIED BY 'password' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';

FLUSH PRIVILEGES;

CHANGE MASTER TO 
    MASTER_HOST='mysql-master',
    MASTER_USER='slave_user',
    MASTER_PASSWORD='password',
    MASTER_AUTO_POSITION=1;

START SLAVE;
```

**Replication Setup Explained:**
- `MASTER_HOST='mysql-master'`: Specifies the master container’s name.
- `MASTER_AUTO_POSITION=1`: Uses GTID for automatic positioning.
- `START SLAVE`: Initiates the replication process.

### 3.2 Create Slave Configuration Files
Create configuration files for the two slave servers with unique `server-id` values.

**Slave 1 Configuration File `my-slave1.cnf`:**
```ini
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

slow_query_log=1
slow_query_log_file=/var/log/mysql/slow.log
long_query_time=0.5
```

**Slave 2 Configuration File `my-slave2.cnf`:**
```ini
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

slow_query_log=1
slow_query_log_file=/var/log/mysql/slow.log
long_query_time=0.5
```

**Slave Configuration Key Points:**
- `server-id`: Unique for each slave (`2` for slave1, `3` for slave2).
- `read_only=on`: Prevents direct writes to slaves.
- `relay-log`: Stores replication events before applying them.

## Step 4: Configure ProxySQL

Set up ProxySQL to route queries intelligently between the master and slaves.

### 4.1 Create ProxySQL Configuration File
Create a configuration file named `proxysql.cnf` to define hostgroups, query rules, and monitoring settings.

```ini
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
    { 
        writer_hostgroup=10,
        reader_hostgroup=20,
        comment="host groups" 
    }
)

mysql_servers =
(
    { 
        address="mysql-master",
        port=3306, 
        hostgroup=10,
        max_connections=100, 
        max_replication_lag = 5
    },
    { 
        address="mysql-slave1",
        port=3306, 
        hostgroup=20,
        max_connections=100, 
        max_replication_lag = 5 
    },
    { 
        address="mysql-slave2",
        port=3306, 
        hostgroup=20,
        max_connections=100, 
        max_replication_lag = 5 
    }
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
    { 
        username = "root",
        password = "password",
        default_hostgroup = 10,
        active = 1
    }
)
```

**ProxySQL Configuration Explained:**
- **Hostgroups:**
  - `10`: Master server for writes.
  - `20`: Slave servers for reads.
- **Query Rules:**
  - `SELECT ... FOR UPDATE` queries go to the master (hostgroup 10).
  - `SELECT` queries go to slaves (hostgroup 20).
  - All other queries (INSERT, UPDATE, DELETE) go to the master.
- **Monitoring:**
  - Health checks every 200 seconds.
  - Maximum replication lag of 5 seconds for slaves.
  - Automatic failover if a server becomes unavailable.

## Step 5: Create Docker Compose File

Define the services for the MySQL master, slaves, and ProxySQL using Docker Compose.

**Docker Compose File `docker-compose.yml`:**
```yaml
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
      - ./master/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "3306:3306"
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
    volumes:
      - ./slave/my-slave1.cnf:/etc/mysql/my.cnf
      - ./slave/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "3307:3306"
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
      - ./slave/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "3308:3306"
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
      - "6032:6032"
      - "6033:6033"
    volumes:
      - ./proxysql/proxysql.cnf:/etc/proxysql.cnf
      - ./proxysql/data:/var/lib/proxysql
    networks:
      - mysql_cluster_net
    depends_on:
      - mysql-master
      - mysql-slave1
      - mysql-slave2
    healthcheck:
      test: ["CMD", "proxysql_admin", "-u", "admin2", "-ppass2", "-h", "127.0.0.1", "-P", "6032", "--execute", "SELECT 1"]
      timeout: 20s
      retries: 5

networks:
  mysql_cluster_net:
    driver: bridge
```

**Docker Compose Key Points:**
- **Port Mapping:** Unique external ports for each service (3306 for master, 3307 for slave1, 3308 for slave2, 6032/6033 for ProxySQL).
- **Dependencies:** Slaves depend on the master, and ProxySQL depends on all MySQL services.
- **Health Checks:** Ensure services are running before proceeding.
- **Volumes:** Mount configuration files and persistent data directories.

## Step 6: Deploy the Cluster

Deploy the MySQL cluster and ProxySQL using Docker Compose.

### 6.1 Start All Services
Launch the services in the background and verify their status.

```bash
# Start all services in background
docker-compose up -d

# Check status
docker-compose ps
```

**Expected Output:**
```
Name                                   Command               State                    Ports
----------------------------------------------------------------------------------------------------
proxysql-mysql-replication-master     docker-entrypoint.sh mysqld  Up      0.0.0.0:3306->3306/tcp
proxysql-mysql-replication-proxysql   proxysql -c /etc/proxysql.cnf Up      0.0.0.0:6032->6032/tcp, 0.0.0.0:6033->6033/tcp
proxysql-mysql-replication-slave1     docker-entrypoint.sh mysqld  Up      0.0.0.0:3307->3306/tcp
proxysql-mysql-replication-slave2     docker-entrypoint.sh mysqld  Up      0.0.0.0:3308->3306/tcp
```

### 6.2 Monitor Startup Process
Check logs to ensure services start without errors.

```bash
docker-compose logs mysql-master
docker-compose logs mysql-slave1
docker-compose logs proxysql
```

## Step 7: Verify Replication Status

Confirm that replication is working correctly between the master and slaves.

### 7.1 Check Master Status
Verify the master’s binary log and GTID status.

```bash
docker-compose exec mysql-master sh -c "export MYSQL_PWD=password; mysql -u root sbtest -e 'SHOW MASTER STATUS\G'"
```

**Expected Output:**
```
*************************** 1. row ***************************
             File: mysql-bin.000003
         Position: 154
     Binlog_Do_DB: sbtest
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 12345678-1234-1234-1234-123456789012:1-5
```

### 7.2 Check Slave Status
Verify that both slaves are replicating correctly.

**Check Slave 1**
```bash
docker-compose exec mysql-slave1 sh -c "export MYSQL_PWD=password; mysql -u root sbtest -e 'SHOW SLAVE STATUS\G'"
```

**Check Slave 2**
```bash
docker-compose exec mysql-slave2 sh -c "export MYSQL_PWD=password; mysql -u root sbtest -e 'SHOW SLAVE STATUS\G'"
```

**Key Indicators to Check:**
- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`
- `Seconds_Behind_Master: 0` (or very low)

## Step 8: Verify ProxySQL Configuration

Ensure ProxySQL is routing queries correctly and all servers are online.

### 8.1 Install MySQL Client
Install a MySQL client to interact with ProxySQL.

```bash
sudo apt update
sudo apt install mysql-client -y
```

### 8.2 Check ProxySQL Server Status
Verify that ProxySQL recognizes all MySQL servers.

```bash
mysql -h 0.0.0.0 -P 6032 -u admin2 -ppass2 -e 'SELECT * FROM mysql_servers;'
```

**Expected Output:**
```
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname     | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 10           | mysql-master | 3306 | 0         | ONLINE | 1      | 0           | 100             | 5                   | 0       | 0              |         |
| 20           | mysql-slave1 | 3306 | 0         | ONLINE | 1      | 0           | 100             | 5                   | 0       | 0              |         |
| 20           | mysql-slave2 | 3306 | 0         | ONLINE | 1      | 0           | 100             | 5                   | 0       | 0              |         |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
```

## Conclusion

This lab demonstrated how to set up a MySQL master-slave replication cluster with ProxySQL for intelligent query routing and load balancing. By using Docker Compose, you created a scalable, high-availability database architecture with one master for writes, two slaves for reads, and ProxySQL for efficient query distribution. The setup includes automated replication with GTID, health checks, and monitoring, making it suitable for production environments.