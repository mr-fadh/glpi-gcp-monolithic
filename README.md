# GLPI IT Asset Management (ITAM) – Monolithic Deployment on GCP Free Tier

This guide documents the complete setup of the GLPI IT Service Management (ITSM) platform, running its application and database on a single Google Compute Engine (GCE) VM.

This configuration is optimized for the GCP Always Free Tier (```e2-micro``` VM) to achieve zero recurring cost.

---

## 1. Architecture Overview (The "Before" State)

### A. Monolithic Architecture
This deployment uses a **monolithic architecture**, meaning all components (Web Server, Application, and Database) are installed on one Virtual Machine (VM). This architecture places all components on a single VM. This is the cheapest method but the least performant.

| Component | Technology | Status |
|----------:|:----------:|:-----|
| Compute   | GCP e2-micro VM | 1 vCPU, 1GB RAM (FREE TIER) |
| OS        | Ubuntu 22.04 LTS | Standard, Stable Linux OS. |
| Web Stack | LEMP (NGINX, PHP 8.1-FPM) | Serves the GLPI application |
| Database  | MySQL Server 8 | Installed and running on ```localhost``` (same VM) |
### B. Critical Trade-Offs
text here

| Requirement | Status | Reason |
|----------:|:----------:|:-----|
| Performance | POOR | The application and database fight for the limited 1GB RAM. The site will be functional but slow. |
| Security(HTTPS) | NONE | Using HTTP only. HTTPS is impossible without a paid domain name. |
| Reliability | LOW | Single Point of Failure (SPOF). If the VM fails, the database is unavailable. |

---

# 2. GCP Provisioning (GCP Cloud Shell Recommended)

Run all ```gcloud``` commands from the **GCP Cloud Shell terminal**.

### A. Initial Setup & Firewall
- **Set Project and Enable API:** Replace ```[YOUR_PROJECT_ID]``` with your project ID.
  ```bash
  gcloud config set project [YOUR_PROJECT_ID]
  gcloud services enable compute.googleapis.com
  ```
- **Create SSH Key:** This key will be injected into the VM for secure access.
  ```bash
  # Creates the key pair in the Cloud Shell home directory
  ssh-keygen -t rsa -b 2048 -C "gcp-glpi-admin" -f ~/.ssh/gcp_rsa
  ```
- **Create Firewall Rules:** We allow HTTP (Port 80) and secure SSH access (Port 22).
  ```bash
  # Allow HTTP (for public access)
  gcloud compute firewall-rules create allow-http --allow=tcp:80 --target-tags=allow-web
  
  # Allow SSH only from your current Public IP (High Security)
  gcloud compute firewall-rules create allow-ssh --allow=tcp:22 --source-ranges=$(curl -s ifconfig.me)/32 --target-tags=allow-ssh
  ```

### B. Create the Free-Tier VM
This command creates the VM with the necessary tags and configuration.
```bash
# Set a free tier zone (choose one: us-west1-a, us-central1-a, us-east1-a)
export GCP_ZONE="us-west1-a"

gcloud compute instances create glpi-server \
    --machine-type=e2-micro \
    --zone=$GCP_ZONE \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud \
    --boot-disk-size=30GB \
    --boot-disk-type=pd-standard \
    --tags=allow-web,allow-ssh \
    --metadata-from-file=ssh-keys=/home/[YOUR_USERNAME]/.ssh/gcp_rsa.pub
```

---

# 3. Server Setup (LEMP Stack Installation)

After the VM is created, connect using the external IP address.
```bash
# Get the ephemeral IP address
export VM_IP=$(gcloud compute instances describe glpi-server --zone=$GCP_ZONE --format='get(networkInterfaces[0].accessConfigs[0].natIP)')

# Connect to the VM (use your Cloud Shell username)
ssh -i ~/.ssh/gcp_rsa [YOUR_USERNAME]@$VM_I
```

### A. Hardening & Swap File (Performance Fix)
This is the key step to avoid OOM errors on the 1GB VM.
```bash
sudo apt update && sudo apt upgrade -y

# 1. Create a 1GB swap file
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 2. Make the swap permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### B. Install Dependencies
```bash
# Install NGINX, MySQL, PHP-FPM, and all required GLPI extensions
sudo apt install -y nginx mysql-server php8.1-fpm \
                   php8.1-mysql php8.1-curl php8.1-gd \
                   php8.1-intl php8.1-mbstring php8.1-xml \
                   php8.1-zip php8.1-bz2 php8.1-ldap \
                   php8.1-apcu php8.1-xmlrpc
```

### C. Configure Local MySQL
We set up the database and the required ```glpi_user``` for the application.
- Secure MySQL: Set the root password (needed for future maintenance).
  ```bash
  sudo mysql_secure_installation
  ```

- Create GLPI Database and User: Replace ```YOUR_GLPI_DB_PASSWORD```.
  - Note: We connect via ```sudo``` to bypass the root password prompt and use the local socket for security.
  ```bash
  sudo mysql -u root
  # You will see the mysql> prompt immediately
  
  # Inside the MySQL prompt, run these commands:
  CREATE DATABASE glpidb;
  CREATE USER 'glpi_user'@'localhost' IDENTIFIED BY 'YOUR_GLPI_DB_PASSWORD';
  GRANT ALL PRIVILEGES ON glpidb.* TO 'glpi_user'@'localhost';
  FLUSH PRIVILEGES;
  EXIT;
  ```

---

# 4. GLPI Application Setup

### A. Download and Permissions
```bash
cd /tmp
# Download the latest stable version (using 10.0.18 for this guide)
wget [https://github.com/glpi-project/glpi/releases/download/10.0.18/glpi-10.0.18.tgz](https://github.com/glpi-project/glpi/releases/download/10.0.18/glpi-10.0.18.tgz)

tar -xzvf glpi-10.0.18.tgz
sudo mv glpi /var/www/glpi

# Set ownership to the web server user
sudo chown -R www-data:www-data /var/www/glpi
sudo chmod -R 755 /var/www/glpi
```

### B. Configure NGINX (Routing Fix)
This configuration block is essential and corrects the routing errors (404 Not Found after login) faced in earlier attempts.
- Optimize PHP-FPM: Adjust for low RAM.
  ```bash
  sudo nano /etc/php/8.1/fpm/pool.d/www.conf
  # Find pm = dynamic and change to pm = ondemand
  # Set pm.max_children = 5
  ```
- Create NGINX Configuration File:
  ```bash
  sudo nano /etc/nginx/sites-available/glpi.conf
  ```

  Paste the following, using the ```server_name_``` directive to accept the IP address:
  ```bash
  server {
    listen 80;
    server_name _; # Accepts requests to the IP address

    # Set the root for the main application
    root /var/www/glpi/public;
    index index.php;

    # --- Main Application Block ---
    location / {
        # This handles all application URLs and redirects them to the front controller
        try_files $uri $uri/ /index.php$is_args$args;
    }

    # --- Installer/Internal PHP Handler ---
    location ~ \.php$ {
        # try_files $uri =404; # Remove this if using 'include snippets'

        # This is the path to the original GLPI files (outside /public)
        # This section specifically allows the installer to run if needed
        location ~ ^/install {
            alias /var/www/glpi/install;
        }

        # Global PHP Processing Block
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_read_timeout 300;
    }

    # --- Security Blocks (CRITICAL) ---
    location ~ /(config|files|locales|sql|vendor|inc) {
        deny all;
    }
    location ~ /\.env {
        deny all;
    }
  }
  ```
- Activate and Restart NGINX:
  ```bash
  sudo ln -s /etc/nginx/sites-available/glpi.conf /etc/nginx/sites-enabled/
  sudo rm /etc/nginx/sites-enabled/default
  
  sudo nginx -t
  sudo systemctl restart nginx
  sudo systemctl restart php8.1-fpm
  ```

### C. Set Permissions and Cron
```bash
# Set ownership to the web server user
sudo chown -R www-data:www-data /var/www/glpi
sudo chmod -R 755 /var/www/glpi

# Set up cron job for GLPI tasks (mail, actions)
sudo nano /etc/cron.d/glpi
# Paste: */2 * * * * www-data /usr/bin/php /var/www/glpi/front/cron.php &>/dev/null
```

---

# 5.  Final Web Installation and Security
- **Access the Installer:** Open your web browser and navigate to http://[YOUR_VM_IP_ADDRESS].
- Follow the steps until you reach the **Database Connection** page.
- **Enter Credentials:**
  - SQL server: localhost
  - SQL user: glpi_user
  - SQL password: YOUR_GLPI_DB_PASSWORD (from Section 3.C.)
- Finish the installation and immediately change the default passwords (glpi/glpi, tech/tech, etc.).

### Final Security and Documentation Step
- Remove Installer File: (Prevents security vulnerability)
  ```bash
  sudo rm /var/www/glpi/install/install.php
  ```
- Record Final IP: Note the external IP address of your VM for future access.

