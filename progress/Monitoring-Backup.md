# GLPI IT Asset Management (ITAM) – Monitoring & Reliability
This phase focuses on ensuring the system is observable, alerting administrators of failures before users notice, and securing data against catastrophic loss.

## Table of Contents 📝
1. [Architecture Overview](#1-architecture-overview)
2. [Deep Observability (Ops Agent)](#2-observability-ops-agent-%EF%B8%8F%E2%80%8D%EF%B8%8F)
3. [Uptime & Alerting](#3-uptime--alerting-)
4. [Automated Backup Strategy](#4-automated-backup-strategy-)
5. [Summary](#5-summary-)

# 1. Architecture Overview
This phase moves the project from a "Passive" state (it works until it breaks) to an "Active" state (it tells us when it's breaking).
| Compnonent | Technology | Status |
|----|----|----|
| Agent |	Google Cloud Ops Agent |	Installed on VM to track RAM/Disk metrics. |
| Monitoring |	Uptime Checks |	Global probes pinging mr-fadh.me every 60s. |
| Alerting |	Email Notifications |	Triggers on High Memory (>90%) or Downtime. |
| Backup |	Snapshot + Script |	Automated nightly backup for both VM config and SQL data. |

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

### 2.2. Verify Installation
Ensure the service is active and running.
```bash
sudo systemctl status google-cloud-ops-agent
```


# 3. Uptime & Alerting 🚨
We configured Google Cloud Monitoring to act as a 24/7 watchman.

### 3.1. Uptime Check
First to create uptime check via **Google Cloud Console** go to Monitoring > Uptime checks > Create Uptime Check.
- Settings:
  - Protocol: HTTPS
  - Target: ```mr-fadh.me```
  - Frequency: ```1 minute```
  - Response Validation: Content matching enabled.
  - Content to match: ```"GCP x GLPI Project"``` (Ensures the landing page loads).

### 3.2. Notification Channel
Then, we need to set alert by go to Monitoring > Uptime checks > Create Uptime Check.
- Add email channel with ```[Your email address]```.
- Then, set the display name as ```Admin Alert Email```

### 3.3. Alert Policies
We created two specific policies to cover performance and availability.

**Policy A: High Memory Usage (OOM Prevention)**
- Metric: VM Instance > Memory > Memory Utilization.
- Condition: Threshold is **Above** ```90%```.
- Duration: 5 minutes.
- Significance: Prevents the server from freezing due to the 1GB RAM limit.

**Policy B: Website Down (Critical)**
- Metric: Uptime Check URL > Check passed.
- Condition: Threshold is **Below** ```90%``` (0.9).
- Significance: Triggers immediately if the NGINX server stops or the domain fails.


# 4. Automated Backup Strategy 💾
We replaced manual ```mysqldump``` operations with a fully automated "Set and Forget" strategy.

### 4.1. VM Configuration (Snapshot Schedule)
Since the VM holds the NGINX configuration and GLPI logic (but not the data), a daily snapshot is sufficient.

- Console Path: Compute Engine > Storage > Snapshots > Scheduled Snapshots.
- Schedule Name: ```daily-web-config```
- Region: ```us-west1``` (Matching VM Location)
- Target: ```glpi-server``` boot disk
- Frequency: Daily at 03:00 - 04:00 (Low traffic window).
- Retention: 7 Days (Auto-delete older snapshots).
- Deletion Rule: "Delete snapshots older than 7 days" (Prevents ghost billing if the VM is deleted).
  - _**Note:**_ This rule is critical for FinOps. It ensures that if the project is deleted, the backups are also deleted to stop billing.

### 4.2. Database Backup (Script + Cron)
Challenge: The Cloud SQL "Free Trial" instance type blocks API-based automated backups (```HTTP Error 400```). 

Solution: We implemented a local script on the VM to dump the data and manage retention.

**Step 1: Create Backup Directory**
```bash
mkdir -p ~/glpi-backups
```

**Step 2: Create the Backup Script File** (```~/backup_db.sh```):
```bash
#!/bin/bash

BACKUP_DIR="/home/[USER]/glpi-backups"
DATE=$(date +"%Y-%m-%d_%H-%M")
DB_HOST="[YOUR_CLOUD_SQL_IP]"
DB_USER="[YOUR_DB_USER_USERNAME]"
DB_PASS="[YOUR_DB_PASSWORD]"
DB_NAME="[YOUR_DB_NAME]"

# Dump the database
mysqldump -h $DB_HOST -u $DB_USER -p"$DB_PASS" $DB_NAME > $BACKUP_DIR/glpi_backup_$DATE.sql

# Optimization: Delete backups older than 7 days to save space
find $BACKUP_DIR -type f -name "*.sql" -mtime +7 -delete
```

**Step 3: Enable Automation (Cron)** 
We added the script to the system scheduler
```bash
crontab -e
```
Add this at the bottom:
```bash
0 3 * * * /bin/bash /home/[USER]/backup_db.sh
```
- **Result**: A fresh SQL dump is saved locally every night at 3:00 AM.

# 5. Summary 📌
| Feature | Implementation | Result |
|----|----|----|
| Visibility | Ops Agent |	**100%** visibility into RAM & Disk usage. |
| Reliability |	Uptime Check |	**< 1 min** detection time for downtime. |
| Recovery |	Auto-Backups |	**24-Hour** Recovery Point Objective (RPO). |
| Cost |	Optimization |	**$0.00** additional cost (Free tier & minimal storage). |
