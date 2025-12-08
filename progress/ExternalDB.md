# GLPI IT Asset Management (ITAM) – External Database Migration
This documentation details the architectural upgrade of GLPI from a Monolithic setup (Single VM) to a Two-Tier Cloud Architecture using Google Cloud SQL.

This upgrade decouples the storage layer from the compute layer, solving memory bottlenecks and eliminating Single Points of Failure (SPOF), while adhering to a strict internship budget ($300 Credit).

## Table of Contents 📝
1. [Architecture Overview](#1-architecture-overview)
2. [Infrastructure Provisioning](#2-infrastructure-provisioning)
3. [Data Migration (ETL)](#3-data-migration-etl-process-)
4. [Application Reconfiguration](#4-application-reconfiguration-)
5. [Operational Management](#5-operational-management)

# 1. Architecture Overview
**Before:** Monolithic Architecture
- Structure: Single Compute Engine VM (```e2-micro```).
- Components: NGINX Web Server + PHP Application + MySQL Database all running on ```localhost```.
- Bottleneck: High RAM contention (1GB total) between the Web Server and Database processes.
- Risk: If the VM is deleted or corrupted, all data is lost.

**After:** Two-Tier Architecture
- Structure: Decoupled Compute and Storage layers.
- Web Tier: Compute Engine VM (```e2-micro```) handling only NGINX and PHP.
- Data Tier: Cloud SQL Instance (```MySQL 8.0 Enterprise Plus```) handling data persistence.
- Benefit: Dedicated resources for each tier, improved fault tolerance, and simplified scaling.

# 2. Infrastructure Provisioning
We provision a managed MySQL to replace the local database.
### 2.1. Cloud SQL Instance Setup
  - Instance Name: ```glpi-sql-trial```
  - Region: ```us-west1``` (Same as VM for low latency)
  - Connectivity: Public IP enabled (Cost-efficient for this phase)
  - Security Rule: Whitelisted the VM's Static External IP ```[YOUR_STATIC_IP]``` in the Cloud SQL "Authorized Networks" settings to permit secure traffic.
### 2.1. Web Server Preparation
We utilized the existing Ubuntu 22.04 VM running the LEMP stack.
- Optimization: Stopped the local MySQL service to free up ~400MB of RAM immediately.
  ```bash
  sudo systemctl stop mysql
  sudo systemctl disable mysql
  ```

# 3. Data Migration (ETL Process) 📦
We performed a manual migration to move existing production data from the local VM to the new Cloud infrastructure.
### 3.1. Maintenance Mode
Prevent data inconsistencies by stopping the web server during migration.
```bash
sudo systemctl stop nginx
```
### 3.2. Data Export (Extract)
Extract the SQL schema and data from the local VM database.
```bash
# Export the 'glpidb' database to a file
sudo mysqldump -u root -p glpidb > migration_backup.sql
```

### 3.3. Cloud Database Initialization
Connect to the remote Cloud SQL instance to create the empty database structure.
```bash
# Connect to Cloud SQL via Public IP
mysql -h [YOUR_CLOUD_SQL_IP] -u root -p

# SQL Commands executed inside the prompt:
> CREATE DATABASE IF NOT EXISTS glpidb;
> CREATE USER 'glpi_user'@'%' IDENTIFIED BY '[YOUR_STRONG_PASSWORD]';
> GRANT ALL PRIVILEGES ON glpidb.* TO 'glpi_user'@'%';
> FLUSH PRIVILEGES;
> EXIT;
```

### 3.4. Data Import (Load)
Upload the local backup file into the Cloud SQL instance.
```bash
# Pipe the backup file directly into the remote database
mysql -h [YOUR_STRONG_PASSWORD] -u glpi_user -p glpidb < migration_backup.sql
```

# 4. Application Reconfiguration 🔧
We updated GLPI's configuration to point to the new database host.
File Path: ```/var/www/glpi/config/config_db.php```

Updated Configuration:
```bash
<?php
class DB extends DBmysql {
   public $dbhost = '[YOUR_STRONG_PASSWORD]';  // Updated from 'localhost' to Cloud SQL IP
   public $dbuser = 'glpi_user';     // Application-specific user
   public $dbpassword = '[YOUR_STRONG_PASSWORD]';
   public $dbdefault = 'glpidb';
   public $use_utf8mb4 = true;
}
```

**Final Verification:** 
1. Restarted Web Server: ```sudo systemctl start nginx```
2. Cleared GLPI Internal Cache: ```sudo php /var/www/glpi/bin/console glpi:cache:clear```
3. Verified login at ```http://[YOUR_STATIC_IP]/```


# 5. Operational Management
Since the project operates on a limited $300 Free Trial Credit, strict cost controls were implemented for the Enterprise Plus Cloud SQL instance. Unlike the VM (which is free/cheap), the Database is billed heavily by the hour.

**Start of Day Routine**

**Objective:** Enable the application for development work.
Action:
1. Navigate to **Google Cloud Console** > **SQL**.
2. Select the instance ```glpi-sql-trial```.
3. Click the **Start** button.


**End of Day Routine**

**Objective:** Halt billing for CPU and RAM usage immediately.
Action:
1. Navigate to Google **Cloud Console** > **SQL**.
2. Select the instance ```glpi-sql-trial```.
3. Click the **Stop** button.

   Crucial Note: We intentionally **do not stop the VM**. The VM has a reserved Static IP address. If the VM is stopped, Google charges an "Unused Static IP Fee" (~$0.01/hr). Leaving the free-tier VM running effectively makes the IP address free.

# Summary 📌
| Category | Status |
|----------|--------|
| Architecture | Two-Tier (VM + Cloud SQL) |
| Reliability | High (Database is independent of VM) |
| Performance | Optimized (VM RAM dedicated to App) |
| Cost | Managed (Manual Start/Stop Routine) |
