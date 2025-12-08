# GLPI IT Asset Management (ITAM) – Security (SSL)
This documentation details the development of the GLPI project on Google Cloud Platform. This phase upgrades the project from a raw IP-based application to a secure, domain-connected professional deployment.

## Table of Contents 📝
1. [Architecture Overview](#1-architecture-overview)
2. [DNS Configuration](#2-dns-configuration-)
3. [Landing Page Implementation](#3-landing-page-implementation-)
4. [NGINX & SSL Setup](#4-nginx--ssl-setup-)
5. [Operational Cost Analysis](#5-operational-cost-analysis-)

# 1. Architecture Overview
Moving to Tier 3 introduces a secure entry point for the user. We replace the direct IP connection with a Domain Name and encrypt all traffic using SSL.

| Component | Technology | Status |
|----------|--------|-------|
| Domain |	mr-fadh.me | Namecheap (GitHub Dev Student Pack) |
| Security | HTTPS (SSL) | Let's Encrypt (Certbot) |
| Web Server | NGINX | Configured for Priority Routing |

**Traffic Flow Update**
1. User visits ```https://mr-fadh.me```.
2. DNS resolves to the GCP VM Static IP (```34.187.163.217```).
3. NGINX handles the SSL Handshake (decryption).
4. Routing: Serves the "Lobby" page (```index.html```) first, protecting the GLPI login (```index.php```).

# 2. DNS Configuration 🌐
### 2.1. Provider Cleanup
Before configuring the server, the domain was cleaned to remove default GitHub Pages settings.
- Action: Removed if any A Records pointing to GitHub (```185.199...```) and the CNAME record.

### 2.2. Pointing to Google Cloud
We created new records to point the domain ```mr-fadh.me``` to the existing Tier 1 VM.
| Component | Technology | Status | TTL |
|----------|--------|-------|------|
| A Record | @ | 34.187.163.217 | Automatic |
| A Record | www | 34.187.163.217 | Automatic |

_Verification:_
```bash
ping -c 2 mr-fadh.me
# Output must show: 34.187.163.217
```

# 3. Landing Page Implementation 🎨
To prevent public visitors from hitting the GLPI login screen immediately, a lightweight "Lobby" page was created.
### 3.1. Create the File
```bash
cd /var/www/glpi/public
sudo nano index.html
```
### 3.2. Code Implementation
Paste the following HTML code to create the dark-themed entry page:
```bash
<!DOCTYPE html>
<html>
<head>
    <title>Mr-Fadh Project</title>
    <style>
        body { background-color: #1a1a1a; color: #ffffff; font-family: sans-serif; 
               display: flex; align-items: center; justify-content: center; height: 100vh; margin: 0; }
        .container { text-align: center; border: 1px solid #333; padding: 40px; 
                     border-radius: 10px; box-shadow: 0 4px 15px rgba(0,0,0,0.5); }
        h1 { margin-bottom: 10px; color: #4285F4; }
        .btn { text-decoration: none; padding: 10px 20px; background-color: #4285F4; 
               color: white; border-radius: 5px; font-weight: bold; transition: 0.3s; }
        .btn:hover { background-color: #3367d6; }
    </style>
</head>
<body>
    <div class="container">
        <h1>GCP x GLPI Project</h1>
        <p>Welcome to the IT Asset Management System Demo.</p>
        <p>Maintained by: Mr-Fadh</p>
        <br>
        <a href="index.php" class="btn">Access System</a>
    </div>
</body>
</html>
```

# 4. NGINX & SSL Setup 🔐
### 4.1. Configure NGINX
We updated the web server to recognize the domain and prioritize the HTML file. 

**File:** ```/etc/nginx/sites-available/glpi.conf```
```bash
server {
    listen 80;
    
    # 1. Update Server Name
    server_name mr-fadh.me www.mr-fadh.me;

    root /var/www/glpi/public;

    # 2. Priority: Load index.html (Lobby) BEFORE index.php (GLPI)
    index index.html index.php;

    # ... (rest of config remains the same)
}
```

### 4.2. Install Certbot (SSL)
We utilized Certbot to install a free Let's Encrypt certificate directly on the VM.
```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Request Certificate
sudo certbot --nginx -d mr-fadh.me -d www.mr-fadh.me
```
- **Redirect Option:** Selected "**2: Redirect**" to force HTTPS automatically.

# 5. Operational Cost Analysis 💰
A key decision was choosing the SSL implementation method to stick to the "Free Tier" goal.
| Feature | GCP Certificate Manager | Certbot(Chosen |
|----------|--------|-------|
| Architecture | Requires Load Balancer | Runs locally on VM | 
| Cost | ~$18.00 / month | **$0.00 (Free)** |
| Verdict | Too expensive for this project | Perfect fit |

**Conclusion:** Using Certbot aligns with the project's "Zero Operational Cost" goal while providing enterprise-grade encryption

## Visual Documents 📸
This section confirms the successful implementation of the secure domain, landing page, and application access.

### Domain & Landing Page
Status: ✅ Confirmed
- SSL Secure: The "Not Secure" warning is gone and replaced by the Padlock icon.
- Custom Branding: The "GCP x GLPI Project" lobby loads by default.
<table>
    <tr>
      <td align="center"><b>Landing Page (mr-fadh.me)</b></td>
    </tr>
    <tr>
      <td align="center">
        <img src="docs/LandingPage-GLPI.png" width="500" align="center">
      </td>
    </tr>
  </table>

### GLPI Application Interface
Status: ✅ Confirmed
- Routing: Clicking "Access System" successfully redirects to the PHP application.
- Connection: The application connects to the Cloud SQL database (Tier 2) via the secure domain.
<table>
  <tr>
    <td align="center"><b>GLPI Login Page (mr-fadh.me)</b></td>
    <td align="center"><b>GLPI Dashboard (mr-fadh.me)</b></td>
  </tr>
  <tr>
    <td align="center">
      <img src="docs/login_page_secured.png" width="500" align="center">
    </td>
    <td align="center">
      <img src="docs/dashboard_page_secured.png" width="600" align="center">
    </td>
  </tr>
</table>



# Summary 📌
| Category | Status |
|----------|--------|
| Cost | $0.00 (Free Tier + Student Pack) | 
| Performance	| High (Static Landing Page loads instantly) |
| Security	| Encrypted (SSL/HTTPS via Let's Encrypt) | 
| Use Case	| Professional presentation for Internship/Portfolio | 

