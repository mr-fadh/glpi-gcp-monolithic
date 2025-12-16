# GLPI IT Asset Management (ITAM) – Monitoring & Reliability
This phase focuses on ensuring the system is observable, alerting administrators of failures before users notice, and securing data against catastrophic loss.

## Table of Contents 📝
1. [Architecture Overview](#1-architecture-overview)
2. [Deep Observability (Ops Agent)](#2-observability-ops-agent-%EF%B8%8F%E2%80%8D%EF%B8%8F)
3. [Uptime & Alerting](#3-uptime--alerting-)
4. [Automated Backup Strategy](#4-automated-backup-strategy-)
5. [Summary](#5-summary-)

# 1. Architecture Overview
**Before:** "Flying Blind"
- Visibility: Google Cloud Console only showed CPU usage. RAM usage (critical for the 1GB ```e2-micro```) was invisible.
- Availability: If the website crashed (NGINX failure), the VM would look "healthy," but users would face downtime.
- Data Safety: Backups were manual ```mysqldump``` commands run by hand.

**After:** "Full Observability"
- Ops Agent: Installed inside the VM to report Memory & Disk metrics.
- Uptime Checks: Global probes ping ```mr-fadh.me``` every 60 seconds.
- Automation: Database and VM disks are backed up nightly without human intervention.

# 2. Observability (Ops Agent) 🕵️‍♂️
Standard GCP metrics cannot see inside the Operating System. We installed the **Google Cloud Ops** Agent to expose RAM utilization.

### 2.1. Agent Installation
Executed the following commands via SSH on ```glpi-server```:
```bash
# 1. Download the repo script
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh

# 2. Run the script to install the agent
sudo bash add-google-cloud-ops-agent-repo.sh --also-install

# 3. Verify the service is running
sudo systemctl status google-cloud-ops-agent
```
- Outcome: The "Memory Utilization" chart in the Monitoring Dashboard is now active.

# 3. Uptime & Alerting 🚨
We configured Google Cloud Monitoring to act as a 24/7 watchman.

### 3.1. Uptime Check
Target: ```https://mr-fadh.me```
Frequency: 1 minute
Timeout: 10 seconds
Content Matching: Looks for the string ```"GCP x GLPI Project"``` to ensure the correct landing page is loaded.
3.2. Alert Policies
Notification Channels were configured to send emails to the Admin.

| Policy Name | Trigger Condition | Rationale |
|----|----|----|
| [URGENT] GLPI VM Memory > 90%	| RAM usage exceeds 90% for 5 mins | Prevents OOM (Out of Memory) crashes on the 1GB VM. |
| [CRITICAL] GLPI Site Down | Uptime Check fails (< 90% success) | Immediate notification if the NGINX server or Domain fails. |

# 4. Automated Backup Strategy 💾
We replaced manual backups with a "Set and Forget" automated schedule.

### 4.1. VM Configuration (Snapshot Schedule)
Since the VM holds the NGINX configuration and GLPI logic (but not the data), a daily snapshot is sufficient.
- Schedule Name: ```daily-web-config```
- Target: ```glpi-server``` boot disk
- Frequency: Daily at 03:00 - 04:00 (Low traffic window).
- Retention: 7 Days (Auto-delete older snapshots).
Deletion Rule: "Delete snapshots older than 7 days" (Prevents ghost billing if the VM is deleted).
### 4.2. Database Backup (Script + Cron)
Challenge: The Cloud SQL "Free Trial" instance type blocks API-based automated backups (HTTP Error 400). Solution: Implemented a script-based automation using ```mysqldump``` and ```crontab```.

The Backup Script (```~/backup_db.sh```):
```bash
#!/bin/bash

BACKUP_DIR="/home/[USER]/glpi-backups"
DATE=$(date +"%Y-%m-%d_%H-%M")
DB_HOST="[CLOUD_SQL_IP]"
DB_USER="glpi_user"
DB_PASS="[DB_PASSWORD]"
DB_NAME="glpidb"

# Dump the database
mysqldump -h $DB_HOST -u $DB_USER -p"$DB_PASS" $DB_NAME > $BACKUP_DIR/glpi_backup_$DATE.sql

# Optimization: Delete backups older than 7 days to save space
find $BACKUP_DIR -type f -name "*.sql" -mtime +7 -delete
```

The Scheduler (Crontab):
```bash
0 3 * * * /bin/bash /home/[USER]/backup_db.sh
```
- **Result**: A fresh SQL dump is saved locally every night at 3:00 AM.

# 5. Summary 📌
| Category | Status | Improvement |
|----|----|----|
| Visibility |	✅ Full |	RAM usage is now visible; blind spots removed. |
| Response |	✅ < 5 mins |	Alerts trigger immediately on downtime. |
| Recovery |	✅ 24 Hours |	RPO (Recovery Point Objective) reduced from "Never" to "24 Hours". |
| Cost |	✅ $0.00 |	Uses free Ops Agent and minimal snapshot storage. |
