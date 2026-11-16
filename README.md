# VProfile Infrastructure Monitoring with Zabbix

## Project Overview

This project implements centralized infrastructure monitoring for the VProfile multi-tier application using **Zabbix 7.4**.

The monitoring environment is deployed using Vagrant and VMware Workstation, with a dedicated Zabbix server responsible for collecting metrics, monitoring host availability, detecting infrastructure issues, and providing centralized visibility through a web-based dashboard.

The objective is to monitor the health and availability of the VProfile infrastructure, including its web, application, database, caching, and messaging services.

## Architecture

The monitoring architecture consists of a centralized Zabbix server and Zabbix agents installed on the VProfile infrastructure nodes.

### Infrastructure Components

| Virtual Machine | Operating System | Monitored Component |
|---|---|---|
| zabbix | CentOS Stream 9 | Zabbix Server, Frontend, Database |
| web01 | Ubuntu 22.04 | Nginx |
| app01 | CentOS Stream 9 | Apache Tomcat |
| db01 | CentOS Stream 9 | MariaDB |
| mc01 | CentOS Stream 9 | Memcached |
| rmq01 | CentOS Stream 9 | RabbitMQ |

**Note:** The intended lab network uses `192.168.56.0/24`, with the Zabbix server planned at `192.168.56.16` and application nodes at `192.168.56.11–15`. Actual IP addresses should be verified in the running VMware environment.

### Monitoring Architecture

```text
                    Zabbix Web Interface
                            |
                            |
                     Zabbix Server
                       CentOS 9
                            |
                     MariaDB Database
                            |
             -----------------------------
             |       |       |      |     |
           web01   app01   db01   mc01   rmq01
           Nginx   Tomcat  MariaDB Memcached RabbitMQ
             |       |       |      |     |
             -------- Zabbix Agents --------
                            |
                    Metrics and Alerts
```

## Technologies Used

- **Zabbix 7.4.16** – Infrastructure monitoring and alerting.
- **CentOS Stream 9** – Zabbix server and Linux application nodes.
- **Ubuntu 22.04** – Web server node.
- **MariaDB 10.5** – Zabbix database.
- **Apache / PHP** – Zabbix frontend.
- **Zabbix Agent** – Host-level monitoring.
- **Vagrant** – Virtual machine provisioning.
- **VMware Workstation 17 Player** – Virtualization platform.
- **Chrony** – Time synchronization on Linux systems.

## 1. Zabbix Server Installation

A dedicated CentOS Stream 9 virtual machine was prepared to host the Zabbix monitoring server.

### Add the Official Zabbix Repository

```bash
sudo rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/centos/9/noarch/zabbix-release-latest-7.4.el9.noarch.rpm

sudo dnf clean all
sudo dnf makecache
```

### Install Zabbix Server and Frontend Packages

The EPEL repository contained older Zabbix packages that conflicted with the official Zabbix 7.4 packages. The installation was therefore performed with EPEL disabled.

```bash
sudo dnf install -y \
zabbix-server-mysql \
zabbix-web-mysql \
zabbix-apache-conf \
zabbix-sql-scripts \
zabbix-selinux-policy \
zabbix-agent \
--disablerepo=epel
```

### Configure MariaDB

MariaDB was enabled and configured to host the Zabbix database.

```bash
sudo systemctl enable --now mariadb
```

Create the database and user:

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'YOUR_DB_PASSWORD';

GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';

FLUSH PRIVILEGES;
```

Import the initial database schema:

```bash
zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

### Configure Zabbix Server

Edit the server configuration:

```bash
sudo vi /etc/zabbix/zabbix_server.conf
```

Configure the database password:

```ini
DBPassword=YOUR_DB_PASSWORD
```

### Configure PHP Time Zone

Edit:

```bash
sudo vi /etc/php-fpm.d/zabbix.conf
```

Set:

```ini
php_value[date.timezone] = Africa/Cairo
```

### Start and Enable Services

```bash
sudo systemctl enable --now zabbix-server
sudo systemctl enable --now httpd
sudo systemctl enable --now php-fpm
```

Verify the Zabbix server:

```bash
sudo systemctl status zabbix-server
```

## 2. Accessing the Zabbix Web Interface

The Zabbix frontend is available through:

```text
http://ZABBIX_SERVER_IP/zabbix
```

During the initial setup wizard, the following configuration was used:

| Setting | Value |
|---|---|
| Database type | MySQL |
| Database host | localhost |
| Database name | zabbix |
| Database user | zabbix |
| PHP time zone | Africa/Cairo |
| Zabbix server name | VProfile-Zabbix |

The default Zabbix login credentials are:

```text
Username: Admin
Password: zabbix
```

The default password should be changed after the first login.

## 3. Zabbix Agent Installation on Monitored Hosts

Zabbix agents are installed on the VProfile nodes to collect operating system and application infrastructure metrics.

### CentOS Stream 9 Nodes

The following procedure applies to:

- app01
- db01
- mc01
- rmq01

Add the Zabbix repository:

```bash
sudo rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/centos/9/noarch/zabbix-release-latest-7.4.el9.noarch.rpm

sudo dnf clean all
```

Install the agent:

```bash
sudo dnf install -y zabbix-agent --disablerepo=epel
```

### Ubuntu 22.04 Node

The following procedure applies to web01.

Download and install the official repository package:

```bash
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu22.04_all.deb

sudo dpkg -i zabbix-release_latest_7.4+ubuntu22.04_all.deb

sudo apt update
```

Install the agent:

```bash
sudo apt install -y zabbix-agent
```

## 4. Zabbix Agent Configuration

The agent configuration file is:

```text
/etc/zabbix/zabbix_agentd.conf
```

Each monitored host has its own unique hostname.

Example configuration for db01:

```ini
Server=192.168.56.16
ServerActive=192.168.56.16
Hostname=db01
HostMetadata=linux
```

The same configuration pattern is applied to the other nodes, changing `Hostname` to the corresponding machine name.

| Host | Hostname |
|---|---|
| web01 | web01 |
| app01 | app01 |
| db01 | db01 |
| mc01 | mc01 |
| rmq01 | rmq01 |

The shared metadata value identifies hosts belonging to the VProfile Linux monitoring environment.

Enable and restart the agent:

```bash
sudo systemctl enable --now zabbix-agent
sudo systemctl restart zabbix-agent
```

Check its status:

```bash
sudo systemctl status zabbix-agent
```

## 5. Automatic Host Registration

Automatic registration is configured to allow monitored Linux hosts to register with the Zabbix server using active agent checks.

### Configure the Autoregistration Action

Navigate to:

```text
Alerts → Actions → Autoregistration actions
```

Create or configure an action named:

```text
linux Auto Registration
```

Configure the condition:

```text
Host metadata contains linux
```

Configure the operations:

1. Add host.
2. Add host to host group: `Servers`.
3. Link template: `Linux by Zabbix agent`.

The agent's `HostMetadata` value must match the action condition.

Each machine must retain its own unique hostname so Zabbix can identify it correctly.

### Verify Agent Connectivity

Install the Zabbix command-line utility on the server:

```bash
sudo dnf install -y zabbix-get --disablerepo=epel
```

Test a monitored host:

```bash
zabbix_get -s TARGET_HOST_IP -p 10050 -k agent.ping
```

Expected result:

```text
1
```

A successful response confirms that the Zabbix server can communicate with the agent using a passive agent check.

## 6. Network and Firewall Configuration

The following ports are relevant to the monitoring environment:

| Port | Protocol | Purpose |
|---|---|---|
| 80 | TCP | Zabbix web interface over HTTP |
| 10050 | TCP | Zabbix agent passive checks |
| 10051 | TCP | Zabbix server and active agent communication |

On CentOS, firewall rules can be configured using firewalld.

Example for the Zabbix server:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-port=10051/tcp
sudo firewall-cmd --reload
```

On monitored hosts, allow agent access from the Zabbix server:

```bash
sudo firewall-cmd --permanent \
--add-rich-rule='rule family="ipv4" source address="ZABBIX_SERVER_IP" port port="10050" protocol="tcp" accept'

sudo firewall-cmd --reload
```

Replace `ZABBIX_SERVER_IP` with the actual server address.

## 7. Custom Application Service Monitoring

In addition to standard operating system monitoring, custom Zabbix agent UserParameters were prepared to monitor application service availability.

### Nginx Monitoring – web01

Create:

```bash
sudo vi /etc/zabbix/zabbix_agentd.d/nginx.conf
```

Add:

```ini
UserParameter=service.status.nginx,/usr/bin/systemctl is-active --quiet nginx && echo 1 || echo 0
```

### RabbitMQ Monitoring – rmq01

Create:

```bash
sudo vi /etc/zabbix/zabbix_agentd.d/rabbitmq.conf
```

Add:

```ini
UserParameter=service.status.rabbitmq,/usr/bin/systemctl is-active --quiet rabbitmq-server && echo 1 || echo 0
```

Ensure the agent includes the custom configuration directory:

```ini
Include=/etc/zabbix/zabbix_agentd.d/*.conf
```

Restart the agent after configuring the UserParameters:

```bash
sudo systemctl restart zabbix-agent
```

### Test Custom Monitoring Keys

From the Zabbix server:

```bash
zabbix_get -s WEB01_IP -k service.status.nginx

zabbix_get -s RMQ01_IP -k service.status.rabbitmq
```

Expected values:

| Value | Meaning |
|---|---|
| 1 | Service is running |
| 0 | Service is not running |

**Current troubleshooting note:** The RabbitMQ check reached the agent, but execution of `/usr/bin/systemctl` was denied in the Zabbix agent's SELinux context. The returned `0` is therefore not yet a reliable indication that RabbitMQ is stopped. This requires an SELinux-compatible solution before the custom service check can be considered operational.

## 8. Real-Time Monitoring and Troubleshooting

The Zabbix frontend provides centralized monitoring through the dashboard and Problems view.

The intended monitoring workflow includes:

- Reviewing host availability.
- Monitoring CPU, memory, disk, and network metrics.
- Identifying active infrastructure problems.
- Tracking problem severity and recovery.
- Investigating service availability issues.

### Service Failure Simulation

Once the custom service checks and triggers are validated, service failures can be simulated to test alert generation and recovery.

Stop Nginx on web01:

```bash
sudo systemctl stop nginx
```

Start Nginx again:

```bash
sudo systemctl start nginx
```

Stop RabbitMQ on rmq01:

```bash
sudo systemctl stop rabbitmq-server
```

Start RabbitMQ again:

```bash
sudo systemctl start rabbitmq-server
```

Expected behavior after the service checks and triggers are configured:

1. Zabbix detects the service status change.
2. The corresponding trigger changes to a problem state.
3. The issue appears in the Problems view.
4. After the service is restarted, Zabbix detects recovery.
5. The problem is automatically resolved.

## 9. Time Synchronization

Accurate time synchronization is important for reliable monitoring, event timestamps, and log correlation.

Chrony is used to synchronize Linux system clocks.

### CentOS Stream 9

```bash
sudo systemctl enable --now chronyd
```

### Ubuntu 22.04

```bash
sudo systemctl enable --now chrony
```

Check system time synchronization:

```bash
timedatectl
```

Check Chrony tracking:

```bash
chronyc tracking
```

Check available NTP sources:

```bash
chronyc sources -v
```

A source marked with `^*` indicates the currently selected synchronization source.

If synchronization is unsuccessful, investigate NTP source availability and network access over UDP port 123.

## 10. Troubleshooting Commands

Useful commands for diagnosing Zabbix agent and server issues:

Check Zabbix server:

```bash
sudo systemctl status zabbix-server
sudo journalctl -u zabbix-server -f
```

Check agent:

```bash
sudo systemctl status zabbix-agent
sudo journalctl -u zabbix-agent -f
```

Review agent configuration:

```bash
sudo grep -E '^(Server|ServerActive|Hostname|HostMetadata|Include)=' \
/etc/zabbix/zabbix_agentd.conf
```

Test agent connectivity:

```bash
zabbix_get -s TARGET_HOST_IP -p 10050 -k agent.ping
```

Check SELinux status:

```bash
getenforce
```

Review recent SELinux denials:

```bash
sudo ausearch -m AVC -ts recent
```

Check a monitored service:

```bash
sudo systemctl status nginx
sudo systemctl status rabbitmq-server
```





The next stage is to complete end-to-end validation of all monitored hosts, resolve the SELinux restriction affecting the RabbitMQ custom check, and verify alert generation and recovery through the Zabbix frontend.
