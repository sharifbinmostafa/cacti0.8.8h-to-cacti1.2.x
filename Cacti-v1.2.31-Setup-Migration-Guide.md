# Cacti v1.2.31 Setup & Migration Guide

**Scope:** Old Cacti server → New Ubuntu server → SQL restore → RRD restore → Cacti v1.2.31 validation
**Use case:** Internal NMS migration SOP

---

## 1. Target Server Information

| Component | Value |
|---|---|
| OS | Ubuntu 24.04 LTS |
| Web Server | Apache 2.4 |
| Database | MariaDB 10.11 |
| PHP | 8.x |
| Cacti Version | 1.2.31 |
| Web Path | `/var/www/html/cacti` |
| RRD Path | `/var/www/html/cacti/rra` |
| Log Path | `/var/www/html/cacti/log/cacti.log` |
| Database Name | `cacti` |

---

## Part 1: New Server Preparation

### 1.1 Update Ubuntu

```bash
apt update
apt upgrade -y
```

### 1.2 Install Required Packages

```bash
apt install -y apache2 mariadb-server php php-mysql php-snmp php-gd php-xml \
php-mbstring php-ldap php-intl php-gmp php-json php-curl php-zip php-bcmath \
rrdtool snmp snmpd unzip wget
```

### 1.3 Enable & Restart Services

```bash
systemctl enable apache2
systemctl enable mariadb

systemctl restart apache2
systemctl restart mariadb
```

---

## Part 2: MariaDB Configuration

Edit the config file:

```bash
nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Under `[mysqld]`, add:

```ini
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci

max_connections=200
max_allowed_packet=64M

innodb_file_per_table=ON
innodb_buffer_pool_size=4G

tmp_table_size=512M
max_heap_table_size=512M

innodb_flush_method=O_DIRECT
```

Restart MariaDB:

```bash
systemctl restart mariadb
```

Verify:

```bash
mysql -e "
SHOW VARIABLES LIKE 'character_set_server';
SHOW VARIABLES LIKE 'collation_server';
"
```

**Expected output:**
```
utf8mb4
utf8mb4_unicode_ci
```

---

## Part 3: Create Cacti Database

Login to MySQL:

```bash
mysql -u root -p
```

Create database:

```sql
CREATE DATABASE cacti CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Create user:

```sql
CREATE USER 'cacti'@'localhost' IDENTIFIED BY 'PASSWORD';
```

Grant privileges:

```sql
GRANT ALL PRIVILEGES ON cacti.* TO 'cacti'@'localhost';
GRANT SELECT ON mysql.time_zone_name TO 'cacti'@'localhost';
FLUSH PRIVILEGES;
```

Load timezone data:

```bash
mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -u root mysql
```

Verify:

```sql
SELECT COUNT(*) FROM mysql.time_zone_name;
```

---

## Part 4: Install Cacti Files

Download:

```bash
cd /var/www/html
wget https://www.cacti.net/downloads/cacti-latest.tar.gz
```

Extract:

```bash
tar -xzf cacti-latest.tar.gz
```

Rename:

```bash
mv cacti-* cacti
```

Set permissions:

```bash
chown -R www-data:www-data /var/www/html/cacti
chmod -R 775 /var/www/html/cacti
```

---

## Part 5: Apache Configuration

Create a new site config:

```bash
nano /etc/apache2/sites-available/cacti.conf
```

Add:

```apache
<VirtualHost *:80>

    ServerName cacti-server

    DocumentRoot /var/www/html/cacti

    Alias /cacti /var/www/html/cacti

    <Directory /var/www/html/cacti>
        AllowOverride All
        Require all granted
    </Directory>

</VirtualHost>
```

Enable the site and disable the default:

```bash
a2ensite cacti.conf
a2dissite 000-default.conf
```

Restart Apache:

```bash
systemctl restart apache2
```

Access Cacti at:

```
http://SERVER-IP/cacti
```

---

## Part 6: SQL Database Migration

### 6.1 Old Server — Backup

Export the database:

```bash
mysqldump -u root -p cacti > cacti.sql
```

Copy to new server:

```bash
scp cacti.sql root@NEW-IP:/home/sharif/
```

### 6.2 New Server — Restore Database

Import:

```bash
mysql -u root -p cacti < /home/sharif/cacti.sql
```

Check:

```bash
mysql -u cacti -p cacti
```

List tables:

```sql
SHOW TABLES;
```

---

## Part 7: Restore RRD Files

RRD files contain the historical graph data.

### 7.1 Old Server — Backup

```bash
cd /var/www/html/cacti
tar czf rra-backup.tar.gz rra/
```

Copy to new server:

```bash
scp rra-backup.tar.gz root@NEW-IP:/home/sharif/
```

### 7.2 New Server — Restore

Remove the empty RRA directory contents:

```bash
rm -rf /var/www/html/cacti/rra/*
```

Extract the backup:

```bash
tar xzf /home/sharif/rra-backup.tar.gz -C /var/www/html/cacti/
```

Fix permissions:

```bash
chown -R www-data:www-data /var/www/html/cacti/rra
chmod -R 775 /var/www/html/cacti/rra
```

Verify:

```bash
ls -lh /var/www/html/cacti/rra | head
```

**Expected output:**
```
traffic_in_123.rrd
traffic_out_123.rrd
```

---

## Part 8: Update Database Paths

After migration, check current path settings:

```bash
mysql -u cacti -p cacti -e "
SELECT name,value FROM settings
WHERE name LIKE 'path%';
"
```

**Correct expected values:**

| Setting | Value |
|---|---|
| `path_webroot` | `/var/www/html/cacti` |
| `path_cactilog` | `/var/www/html/cacti/log/cacti.log` |

If the old paths are still present, update them:

```sql
UPDATE settings
SET value='/var/www/html/cacti'
WHERE name='path_webroot';

UPDATE settings
SET value='/var/www/html/cacti/log/cacti.log'
WHERE name='path_cactilog';
```

---

## Part 9: Create Log File

```bash
mkdir -p /var/www/html/cacti/log
touch /var/www/html/cacti/log/cacti.log
chown www-data:www-data /var/www/html/cacti/log/cacti.log
chmod 664 /var/www/html/cacti/log/cacti.log
```

---

## Part 10: Upgrade Database

Run the upgrade script:

```bash
php /var/www/html/cacti/cli/upgrade_database.php
```

**Expected output:**
```
Upgrading from v1.x.x to v1.2.31
```

---

## Part 11: Cacti Poller Setup

Test the poller manually:

```bash
php /var/www/html/cacti/poller.php
```

Run in debug mode if needed:

```bash
php /var/www/html/cacti/poller.php --debug
```

---

## Part 12: Cron Configuration

Create the cron file:

```bash
nano /etc/cron.d/cacti
```

Add:

```
*/5 * * * * www-data php /var/www/html/cacti/poller.php
```

Restart cron:

```bash
systemctl restart cron
```

Check status:

```bash
systemctl status cron
```

---

## Part 13: Post-Migration Checks

### 13.1 Check RRD Creation

```bash
ls -lh /var/www/html/cacti/rra | tail
```

### 13.2 Check Poller Output

In the Cacti web UI:

```
Console → System Utilities → View Poller Statistics
```

### 13.3 Check Device Status

```
Console → Devices → Device
```

**Expected:**
- Status: `UP`
- SNMP: `Success`

### 13.4 Check Graphs

```
Console → Management → Graph Management
```

---

## Part 14: Common Migration Problems

### Problem: System log file is not available for writing

**Fix:**

```bash
chown www-data:www-data /var/www/html/cacti/log/cacti.log
chmod 664 /var/www/html/cacti/log/cacti.log
```

### Problem: Graphs not creating

**Check:**

```bash
php /var/www/html/cacti/poller.php --debug
```

```bash
ls -lh /var/www/html/cacti/rra
```

### Problem: 404 Not Found on /cacti

**Check:**

```bash
apache2ctl -S
```

**Verify:**

```
DocumentRoot /var/www/html/cacti
```

**Restart:**

```bash
systemctl restart apache2
```

---

## Final Validation Checklist

| Item | Status |
|---|---|
| Apache Working | ✅ |
| MariaDB Working | ✅ |
| Cacti Login | ✅ |
| Database Restored | ✅ |
| RRD Restored | ✅ |
| Historical Graph Available | ✅ |
| New Graph Creation | ✅ |
| Poller Running | ✅ |
| Cron Running | ✅ |
| SNMP Polling | ✅ |

---

*This document matches the migration scenario: old Cacti → new Ubuntu server → SQL restore → RRD restore → Cacti v1.2.31. Use it as an internal NMS migration SOP.*
