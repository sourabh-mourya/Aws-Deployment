# Node.js Deployment Without Docker (AWS EC2)

Complete guide for deploying Node.js/Express applications on AWS EC2 - Step by step in simple language!

---

## Prerequisites (Sabse Pehle Ye Chahiye)

✅ AWS account (free tier bhi chalega)  
✅ Node.js/Express application ready  
✅ GitHub repository (optional, but recommended)  
✅ Domain name (optional - for production)  

---

## Table of Contents

1. [EC2 Instance Launch](#1-ec2-instance-launch)
2. [Connect to EC2](#2-connect-to-ec2)
3. [Install Dependencies](#3-install-dependencies-node-git-etc)
4. [Project Setup](#4-project-setup)
5. [Environment Configuration](#5-environment-configuration)
6. [PM2 Setup (Production)](#6-pm2-setup-production-ready)
7. [Nginx Setup (Optional but Recommended)](#7-nginx-setup-reverse-proxy)
8. [MongoDB Setup (If Needed)](#8-mongodb-setup-if-needed)
9. [SSL Certificate](#9-ssl-certificate-optional)
10. [Maintenance & Troubleshooting](#10-maintenance--troubleshooting)

---

## 1. EC2 Instance Launch

### Step 1.1: AWS Console Me Jao

1. **AWS Console** login karo
2. **EC2 Dashboard** → **Launch Instance** pe click karo

---

### Step 1.2: Instance Configuration

**Name and AMI:**
- **Name:** `nodejs-production-server` (kuch bhi naam de sakte ho)
- **AMI (Operating System):**
  - **Ubuntu 22.04 LTS** (Recommended) ✅
  - Ya phir Amazon Linux 2

**Kyun Ubuntu?**  
Ubuntu zyada easy hai aur zyada tutorials available hain.

---

### Step 1.3: Instance Type Select Karo

- **Free Tier:** `t2.micro` (1 vCPU, 1GB RAM) - Testing ke liye
- **Small Project:** `t2.small` (1 vCPU, 2GB RAM)
- **Production:** `t2.medium` (2 vCPU, 4GB RAM) - Recommended

---

### Step 1.4: Key Pair Create/Select Karo

**Bahut Important Step Hai!**

1. **Create new key pair** pe click karo
2. **Key pair name:** Apna naam do (e.g., `nodejs-key`)
3. **Key pair type:** RSA
4. **Private key file format:** `.pem` (for Mac/Linux) ya `.ppk` (for Windows PuTTY)
5. **Create key pair** click karo
6. **File automatically download hogi - Isko safe jagah save karo!**

⚠️ **Warning:** Ye file dobara download nahi kar sakte, agar kho gayi to new EC2 instance banana padega!

---

### Step 1.5: Network Settings (Security Group)

**Inbound Rules Configure Karo:**

Ye ports allow karne hain:

```
Rule Type          Port    Source          Purpose
---------------------------------------------------------
SSH                22      My IP           Terminal se connect karne ke liye
HTTP               80      0.0.0.0/0       Website access (without port)
HTTPS              443     0.0.0.0/0       SSL secure access
Custom TCP         3000    0.0.0.0/0       Testing ke liye (baad me delete kar dena)
Custom TCP         8080    0.0.0.0/0       Agar 8080 port use kar rahe ho
```

**Kaise add kare:**

1. **Security Groups** section me jao
2. **Edit** pe click karo
3. **Add Rule** pe click karke upar wale rules add karo
4. **Source:** `0.0.0.0/0` (Anywhere) select karo (testing ke liye)
5. Production me SSH ke liye sirf **My IP** select karna better hai

---

### Step 1.6: Storage Configure Karo

- **Size:** 20 GB (minimum)
- **Volume Type:** gp3 (General Purpose SSD)

8GB bhi chal jayega but 20GB better hai for future.

---

### Step 1.7: Launch Instance

1. **Launch Instance** button pe click karo
2. Instance start hone me 2-3 minute lagenge
3. **Instance State** me `Running` dikhne ka wait karo
4. **Public IP address** note kar lo (instance details me milega)

Example: `18.209.248.224`

---

## 2. Connect to EC2

### Step 2.1: Windows Users (GitBash/PowerShell)

**GitBash Install Karo (Recommended):**
- Download from: https://git-scm.com/download/win
- Install karo with default settings

**Terminal Open Karo:**

```bash
# GitBash me ye commands run karo

# Step 1: Jaha .pem file hai us folder me jao
cd Downloads  # Ya jaha bhi save kiya hai

# Step 2: File permissions set karo (GitBash me)
chmod 400 nodejs-key.pem

# Step 3: EC2 se connect karo
ssh -i nodejs-key.pem ubuntu@18.209.248.224
```

**Replace karo:**
- `nodejs-key.pem` → Apni .pem file ka naam
- `18.209.248.224` → Apna EC2 Public IP
- `ubuntu` → Agar Amazon Linux use kiya to `ec2-user` likho

---

### Step 2.2: Mac/Linux Users

**Terminal Open Karo:**

```bash
# Step 1: Downloads folder me jao (ya jaha .pem file hai)
cd ~/Downloads

# Step 2: File permissions set karo
chmod 400 nodejs-key.pem

# Step 3: Connect karo
ssh -i nodejs-key.pem ubuntu@18.209.248.224
```

---

### Step 2.3: Connection Successful?

Agar successfully connect ho gaye to kuch aisa dikhega:

```
Welcome to Ubuntu 22.04 LTS
ubuntu@ip-172-31-45-123:~$
```

Congratulations! Ab tum EC2 server ke andar ho! 🎉

---

### Step 2.4: Connection Issues?

**Error: Permission denied (publickey)**
```bash
# Solution: Permissions check karo
chmod 400 nodejs-key.pem
```

**Error: Connection timed out**
```bash
# Solution: Security group check karo
# SSH port 22 allowed hai ya nahi check karo AWS console me
```

**Windows me chmod command nahi chal rahi?**
```bash
# GitBash use karo (not CMD or PowerShell)
# Ya PuTTY use karo with .ppk file
```

---

## 3. Install Dependencies (Node, Git, etc)

### Step 3.1: System Update Karo

**Ubuntu ke liye:**

```bash
# System update aur upgrade
sudo apt update && sudo apt upgrade -y

# Basic tools install karo
sudo apt install -y git curl wget vim nano build-essential
```

**Amazon Linux ke liye:**

```bash
# System update
sudo yum update -y

# Basic tools install karo
sudo yum install -y git curl wget vim
```

---

### Step 3.2: Node.js Install Karo

**Ubuntu (Recommended Method):**

```bash
# NodeSource repository add karo (Node.js 18 LTS)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -

# Node.js aur npm install karo
sudo apt install -y nodejs

# Verify installation
node --version   # Output: v18.x.x
npm --version    # Output: 9.x.x
```

**Amazon Linux:**

```bash
# NodeSource repository add karo
curl -fsSL https://rpm.nodesource.com/setup_18.x | sudo -E bash -

# Node.js install karo
sudo yum install -y nodejs git

# Verify
node --version
npm --version
```

**Different Node.js versions ke liye:**
- Node 16: `setup_16.x`
- Node 18: `setup_18.x` (LTS - Recommended)
- Node 20: `setup_20.x` (Latest)

---

### Step 3.3: Git Configure Karo (Optional)

```bash
# Git configuration (optional but good practice)
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

---

## 4. Project Setup

### Step 4.1: Clone Your Repository

**Method 1: Git Clone (Recommended)**

```bash
# Home directory me jao
cd /home/ubuntu

# Project clone karo
git clone https://github.com/yourusername/your-nodejs-project.git

# Project folder me jao
cd your-nodejs-project

# Check files
ls -la
```

**Method 2: Upload via SCP (Local Machine Se)**

Agar GitHub pe code nahi hai to local machine se upload karo:

```bash
# Local machine ke terminal me (apne computer pe)
scp -i nodejs-key.pem -r /path/to/your/project ubuntu@18.209.248.224:/home/ubuntu/
```

**Method 3: Manual Upload (FileZilla)**

1. FileZilla download karo
2. SFTP connection setup karo using .pem file
3. Files upload karo

---

### Step 4.2: Install Dependencies

```bash
# Make sure tum project folder me ho
cd /home/ubuntu/your-nodejs-project

# Dependencies install karo
npm install

# Production ke liye (optional)
npm install --production

# Check if installed successfully
ls -la node_modules/
```

**Agar error aaye:**

```bash
# Node version check karo
node --version

# Cache clear karke phir se try karo
npm cache clean --force
npm install
```

---

### Step 4.3: Test Your Application

```bash
# Application start karo (testing)
node server.js
# Or
npm start

# Application successfully start hua?
# Output kuch aisa hona chahiye:
# Server running on port 3000
```

**Browser me check karo:**
```
http://your-ec2-ip:3000
Example: http://18.209.248.224:3000
```

Agar website dikh rahi hai to sab sahi hai! 

**Ctrl + C** press karke server stop karo. Ab hum production setup karenge PM2 se.

---

## 5. Environment Configuration

### Step 5.1: .env File Create Karo

```bash
# Project folder me jao
cd /home/ubuntu/your-nodejs-project

# .env file create karo
nano .env
```

---

### Step 5.2: Environment Variables Add Karo

**.env file me ye add karo:**

```env
# Application Settings
NODE_ENV=production
PORT=3000
APP_NAME=MyNodeApp

# Database (if using MongoDB)
MONGO_URI=mongodb://localhost:27017/mydatabase
# Or for cloud MongoDB
MONGO_URI=mongodb+srv://username:password@cluster.mongodb.net/dbname

# JWT Secret (if using authentication)
JWT_SECRET=your-super-secret-jwt-key-12345

# Other API Keys
API_KEY=your-api-key
SECRET_KEY=your-secret-key

# CORS Settings (if needed)
ALLOWED_ORIGINS=http://yourdomain.com,https://yourdomain.com

# Email Configuration (if needed)
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USER=your-email@gmail.com
EMAIL_PASS=your-app-password
```

**Save kaise kare:**
1. Content type/paste karo
2. `Ctrl + X` press karo
3. `Y` type karo (yes)
4. `Enter` press karo

---

### Step 5.3: .env File ko Secure Karo

```bash
# .env file ke permissions set karo (security)
chmod 600 .env

# Check permissions
ls -la .env
```

**Output:** `-rw------- 1 ubuntu ubuntu` (ye sahi hai)

---

### Step 5.4: .gitignore Check Karo

```bash
# .gitignore file check karo
cat .gitignore

# Agar .env missing hai to add karo
echo ".env" >> .gitignore
echo "node_modules/" >> .gitignore
```

⚠️ **Important:** `.env` file kabhi bhi Git pe push nahi karni chahiye!

---

## 6. PM2 Setup (Production Ready)

PM2 ek production process manager hai jo tumhare Node.js app ko:
- Background me run karta hai
- Crash hone pe automatically restart karta hai
- Server reboot hone pe automatically start karta hai
- Multiple instances run kar sakta hai (clustering)
- Logs manage karta hai

---

### Step 6.1: PM2 Install Karo

```bash
# PM2 globally install karo
sudo npm install -g pm2

# Verify installation
pm2 --version
```

---

### Step 6.2: Application Start Karo with PM2

```bash
# Basic start
pm2 start server.js --name "nodejs-app"

# Agar npm start use karte ho
pm2 start npm --name "nodejs-app" -- start

# Agar different file hai (like index.js, app.js)
pm2 start app.js --name "nodejs-app"
```

**Check karo successfully start hua:**

```bash
# Status check karo
pm2 status

# Output kuch aisa dikhega:
# ┌────┬────────────┬─────────┬─────────┬─────────┐
# │ id │ name       │ mode    │ status  │ restart │
# ├────┼────────────┼─────────┼─────────┼─────────┤
# │ 0  │ nodejs-app │ fork    │ online  │ 0       │
# └────┴────────────┴─────────┴─────────┴─────────┘
```

**Status `online` hona chahiye!**

---

### Step 6.3: Auto-Restart on Server Reboot

```bash
# Startup script generate karo
pm2 startup systemd

# Ye command OUTPUT me ek command dikhayega
# Us command ko COPY karke run karo

# Example output:
# sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u ubuntu --hp /home/ubuntu

# Us command ko copy-paste karke run karo (example):
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u ubuntu --hp /home/ubuntu

# Ab current PM2 process list save karo
pm2 save

# Success message:
# [PM2] Saving current process list...
# [PM2] Successfully saved
```

**Kya hua:**
- Server reboot hone pe PM2 automatically start hoga
- PM2 tumhare app ko automatically start kar dega
- No manual intervention needed

---

### Step 6.4: PM2 Important Commands

```bash
# Application status dekhne ke liye
pm2 status

# Real-time monitoring
pm2 monit

# Logs dekhne ke liye
pm2 logs nodejs-app

# Last 100 lines of logs
pm2 logs nodejs-app --lines 100

# Logs continuously dekhne ke liye (Ctrl+C se exit)
pm2 logs nodejs-app --lines 0

# Application restart karne ke liye
pm2 restart nodejs-app

# Application stop karne ke liye
pm2 stop nodejs-app

# Application start karne ke liye
pm2 start nodejs-app

# Application delete karne ke liye (PM2 list se)
pm2 delete nodejs-app

# All applications restart
pm2 restart all

# Application info
pm2 info nodejs-app

# PM2 process list save karo (har change ke baad)
pm2 save
```

---

### Step 6.5: Advanced PM2 Configuration (Optional)

Agar zyada control chahiye to ecosystem file banao:

```bash
# Ecosystem file create karo
nano ecosystem.config.js
```

**Ye code paste karo:**

```javascript
module.exports = {
  apps: [{
    name: 'nodejs-app',
    script: './server.js',
    
    // Instances configuration
    instances: 1,  // Single instance
    // instances: 2,  // Multiple instances
    // instances: 'max',  // Max CPU cores use karo
    
    exec_mode: 'cluster',  // Cluster mode (multiple instances ke liye)
    
    // Environment variables
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    },
    
    // Logging
    error_file: './logs/err.log',
    out_file: './logs/out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss',
    
    // Memory and restart configuration
    max_memory_restart: '500M',  // 500MB se zyada memory use kare to restart
    autorestart: true,  // Crash pe auto-restart
    watch: false,  // File changes pe auto-restart (development me true karo)
    
    // Other settings
    max_restarts: 10,  // Maximum restart attempts
    min_uptime: '10s'  // Minimum uptime before considering stable
  }]
};
```

**Save karo:** `Ctrl+X`, `Y`, `Enter`

**Logs folder create karo:**

```bash
mkdir -p logs
```

**Ecosystem file se start karo:**

```bash
# Purana instance delete karo (agar hai to)
pm2 delete nodejs-app

# Ecosystem file se start karo
pm2 start ecosystem.config.js

# Save configuration
pm2 save

# Status check
pm2 status
```

---

## 7. Nginx Setup (Reverse Proxy)

Nginx kya karta hai:
- Port number hide karta hai URL se (3000 hide ho jayega)
- `http://domain.com:3000` ki jagah sirf `http://domain.com` se access hoga
- SSL certificate ke liye zaroori hai
- Static files efficiently serve karta hai
- Load balancing kar sakta hai

---

### Step 7.1: Nginx Install Karo

```bash
# Nginx install karo
sudo apt install -y nginx

# Nginx start karo
sudo systemctl start nginx

# Enable karo (boot pe auto-start)
sudo systemctl enable nginx

# Status check karo
sudo systemctl status nginx
```

**Test karo:**  
Browser me `http://your-ec2-ip` open karo  
Nginx welcome page dikhna chahiye

---

### Step 7.2: Nginx Configuration File Create Karo

```bash
# Configuration file create karo
sudo nano /etc/nginx/sites-available/nodejs-app
```

**Ye configuration paste karo:**

```nginx
server {
    listen 80;
    listen [::]:80;
    
    # Apna domain ya EC2 IP yaha add karo
    server_name yourdomain.com www.yourdomain.com 18.209.248.224;

    # File upload size limit
    client_max_body_size 50M;

    # Logs
    access_log /var/log/nginx/nodejs_access.log;
    error_log /var/log/nginx/nodejs_error.log;

    # Main location block - proxy to Node.js
    location / {
        # Node.js application ka address
        proxy_pass http://localhost:3000;
        
        # WebSocket support (Socket.io ke liye)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_cache_bypass $http_upgrade;
        
        # Important headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Static files (optional)
    # Agar public folder hai to ye add karo
    location /static/ {
        alias /home/ubuntu/your-nodejs-project/public/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

**Change karo:**
- `yourdomain.com` → Apna domain (ya EC2 IP)
- `18.209.248.224` → Apna EC2 IP
- `3000` → Apna application port
- `/home/ubuntu/your-nodejs-project/` → Apna project path

**Save karo:** `Ctrl+X`, `Y`, `Enter`

---

### Step 7.3: Nginx Configuration Enable Karo

```bash
# Symbolic link create karo
sudo ln -s /etc/nginx/sites-available/nodejs-app /etc/nginx/sites-enabled/

# Default configuration remove karo
sudo rm /etc/nginx/sites-enabled/default

# Configuration test karo
sudo nginx -t

# Success message dikhna chahiye:
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# Nginx restart karo
sudo systemctl restart nginx

# Status check karo
sudo systemctl status nginx
```

---

### Step 7.4: Test Karo

**Browser me open karo:**

```
http://your-ec2-ip
# Port number ki zarurat nahi hai!

Example: http://18.209.248.224
```

Agar app dikh rahi hai **WITHOUT port number** to Nginx sahi se setup ho gaya! 🎉

---

### Step 7.5: Troubleshooting Nginx

**Problem: 502 Bad Gateway**

```bash
# Check karo PM2 app running hai ya nahi
pm2 status

# Agar stopped hai to start karo
pm2 start nodejs-app

# Nginx restart karo
sudo systemctl restart nginx
```

**Problem: 404 Not Found**

```bash
# Nginx configuration check karo
sudo nginx -t

# Logs dekho
sudo tail -f /var/log/nginx/nodejs_error.log
```

**Problem: Connection Refused**

```bash
# Check port 3000 pe app chal rahi hai ya nahi
sudo lsof -i :3000

# Agar nahi chal rahi to PM2 se start karo
pm2 start nodejs-app
```

---

## 8. MongoDB Setup (If Needed)

Agar tumhare app me MongoDB use ho rahi hai to ye steps follow karo.

---

### Step 8.1: MongoDB Install Karo

**Ubuntu:**

```bash
# MongoDB GPG key import karo
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# MongoDB repository add karo
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Update karo
sudo apt update

# MongoDB install karo
sudo apt install -y mongodb-org

# Start karo
sudo systemctl start mongod

# Enable karo (auto-start on boot)
sudo systemctl enable mongod

# Status check karo
sudo systemctl status mongod
```

**Status `active (running)` hona chahiye!**

---

### Step 8.2: MongoDB Security Setup

```bash
# MongoDB shell open karo
mongosh

# Admin user create karo
use admin

db.createUser({
  user: "admin",
  pwd: "StrongPassword123!",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    "readWriteAnyDatabase"
  ]
})

# Application user create karo
use myapp_db

db.createUser({
  user: "app_user",
  pwd: "AppPassword456!",
  roles: [
    { role: "readWrite", db: "myapp_db" }
  ]
})

# Exit karo
exit
```

---

### Step 8.3: Authentication Enable Karo

```bash
# MongoDB config file edit karo
sudo nano /etc/mongod.conf
```

**Find karo `#security:` line aur uncomment/edit karo:**

```yaml
security:
  authorization: enabled
```

**Save karo:** `Ctrl+X`, `Y`, `Enter`

**MongoDB restart karo:**

```bash
sudo systemctl restart mongod

# Test authentication
mongosh -u admin -p StrongPassword123! --authenticationDatabase admin
```

Successfully login ho gaya? Great! `exit` type karke niklo.

---

### Step 8.4: .env File Update Karo

```bash
# .env file edit karo
nano .env
```

**MongoDB URI add/update karo:**

```env
MONGO_URI=mongodb://app_user:AppPassword456!@localhost:27017/myapp_db?authSource=myapp_db
```

**Save karo aur app restart karo:**

```bash
pm2 restart nodejs-app

# Logs check karo
pm2 logs nodejs-app
```

"MongoDB connected" jaise message dikhna chahiye!

---

## 9. SSL Certificate (Optional)

HTTPS enable karne ke liye SSL certificate install karo - **FREE with Let's Encrypt!**

**Prerequisite:** Tumhare paas domain name hona chahiye (GoDaddy, Namecheap, etc se)

---

### Step 9.1: Domain Point Karo EC2 IP Pe

1. Domain provider (GoDaddy/Namecheap) me jao
2. DNS settings me jao
3. A Record add karo:
   - **Type:** A
   - **Name:** @ (for root domain) aur www (for www subdomain)
   - **Value:** Your EC2 Public IP
   - **TTL:** Automatic ya 3600

4. Save karo aur 5-30 minutes wait karo DNS propagate hone ke liye

**Test karo:**
```bash
# Local machine pe
ping yourdomain.com
```

---

### Step 9.2: Certbot Install Karo

```bash
# Certbot aur Nginx plugin install karo
sudo apt install -y certbot python3-certbot-nginx
```

---

### Step 9.3: SSL Certificate Obtain Karo

```bash
# SSL certificate generate karo
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

**Follow the prompts:**

1. **Email address:** Apna email enter karo (renewal notifications ke liye)
2. **Terms of Service:** `Y` type karo (agree)
3. **Share email:** `N` type karo (optional)
4. **Redirect HTTP to HTTPS:** `2` select karo (Recommended)

**Wait karo... certificate generate ho raha hai...**

**Success message:**
```
Congratulations! You have successfully enabled HTTPS on https://yourdomain.com
```

---

### Step 9.4: Test HTTPS

**Browser me open karo:**
```
https://yourdomain.com
```

Padlock icon 🔒 dikhna chahiye address bar me!

---

### Step 9.5: Auto-Renewal Test

Certificates 90 days me expire hote hain, but Certbot automatically renew kar deta hai.

```bash
# Dry run test karo
sudo certbot renew --dry-run

# Success message aana chahiye
```

**Manual renewal (if needed):**

```bash
sudo certbot renew
sudo systemctl reload nginx
```

---

## 10. Maintenance & Troubleshooting

### 10.1: Code Update Kaise Kare

```bash
# Project folder me jao
cd /home/ubuntu/your-nodejs-project

# Latest code pull karo
git pull origin main

# Dependencies update karo (if package.json changed)
npm install --production

# Application restart karo
pm2 restart nodejs-app

# Logs check karo
pm2 logs nodejs-app --lines 50
```

---

### 10.2: Logs Kaise Dekhe

```bash
# PM2 logs (real-time)
pm2 logs nodejs-app

# PM2 logs (last 100 lines)
pm2 logs nodejs-app --lines 100

# Nginx access logs
sudo tail -f /var/log/nginx/nodejs_access.log

# Nginx error logs
sudo tail -f /var/log/nginx/nodejs_error.log

# MongoDB logs (if installed)
sudo tail -f /var/log/mongodb/mongod.log

# System logs
sudo journalctl -u nginx -f
```

---

### 10.3: Common Problems & Solutions

#### Problem 1: Application crash ho raha hai

```bash
# Logs dekho error ke liye
pm2 logs nodejs-app --err

# Application restart karo
pm2 restart nodejs-app

# Agar phir bhi crash ho to detailed logs dekho
pm2 logs nodejs-app --lines 200
```

---

#### Problem 2: Port already in use

```bash
# Dekhho kon port 3000 use kar raha hai
sudo lsof -i :3000

# Process kill karo
sudo kill -9 <PID>

# Ya sabhi node processes kill karo
sudo pkill node

# PM2 se start karo
pm2 start nodejs-app
```

---

#### Problem 3: MongoDB connection error

```bash
# MongoDB status check karo
sudo systemctl status mongod

# Agar stopped hai to start karo
sudo systemctl start mongod

# Logs check karo
sudo tail -50 /var/log/mongodb/mongod.log

# .env file check karo
cat .env | grep MONGO_URI
```

---

#### Problem 4: 502 Bad Gateway (Nginx)

```bash
# PM2 app running hai ya nahi
pm2 status

# Agar stopped hai to start karo
pm2 start nodejs-app

# Nginx restart karo
sudo systemctl restart nginx

# Nginx logs dekho
sudo tail -f /var/log/nginx/nodejs_error.log
```

---

#### Problem 5: Permission denied errors

```bash
# Project folder ka ownership change karo
sudo chown -R ubuntu:ubuntu /home/ubuntu/your-nodejs-project

# Permissions fix karo
chmod -R 755 /home/ubuntu/your-nodejs-project

# .env file permissions
chmod 600 /home/ubuntu/your-nodejs-project/.env
```

---

### 10.4: Database Backup

```bash
# Backup directory create karo
mkdir -p /home/ubuntu/backups

# MongoDB backup (if using MongoDB)
mongodump --uri="mongodb://app_user:AppPassword456!@localhost:27017/myapp_db?authSource=myapp_db" --out=/home/ubuntu/backups/$(date +%Y%m%d)

# Backup compressed file me
tar -czf /home/ubuntu/backups/backup-$(date +%Y%m%d).tar.gz /home/ubuntu/backups/$(date +%Y%m%d)
```

**Automated backup with cron:**

```bash
# Crontab edit karo
crontab -e

# Daily 2 AM backup (ye line add karo)
0 2 * * * mongodump --uri="mongodb://app_user:AppPassword456!@localhost:27017/myapp_db?authSource=myapp_db" --out=/home/ubuntu/backups/$(date +\%Y\%m\%d) >> /home/ubuntu/backup.log 2>&1
```

---

### 10.5: System Monitoring

```bash
# Disk space check karo
df -h

# Memory usage
free -h

# CPU usage (top command)
top

# PM2 monitoring
pm2 monit

# Check open ports
sudo netstat -tulpn

# Service status check
sudo systemctl status nginx
sudo systemctl status mongod
```

---

## Quick Reference Commands

Ye important commands yaad rakho:

```bash
# SSH Connect
ssh -i nodejs-key.pem ubuntu@your-ec2-ip

# PM2 Commands
pm2 status
pm2 restart nodejs-app
pm2 logs nodejs-app
pm2 monit

# Nginx Commands
sudo systemctl restart nginx
sudo nginx -t
sudo systemctl status nginx

# MongoDB Commands (if installed)
sudo systemctl restart mongod
mongosh -u admin -p --authenticationDatabase admin

# System Commands
df -h          # Disk space
free -h        # Memory
top            # CPU usage

# Code Update
cd /home/ubuntu/your-nodejs-project
git pull
npm install --production
pm2 restart nodejs-app

# Logs
pm2 logs nodejs-app
sudo tail -f /var/log/nginx/nodejs_error.log
```

---

## Security Checklist

✅ **Firewall setup karo:**

```bash
# UFW install aur configure karo
sudo apt install ufw
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable
sudo ufw status
```

✅ **SSH key authentication use karo (password disable karo):**

```bash
sudo nano /etc/ssh/sshd_config

# Ye set karo:
PasswordAuthentication no
PermitRootLogin no

sudo systemctl restart ssh
```

✅ **Regular updates:**

```bash
sudo apt update && sudo apt upgrade -y
```

✅ **Environment variables properly set:**
- `.env` file use karo
- Git me commit mat karo
- Permissions 600 rakho

✅ **MongoDB authentication enabled:**
- User credentials use karo
- Strong passwords use karo

---

## Congratulations! 🎉

Tumhara Node.js application successfully deploy ho gaya hai AWS EC2 pe!

**Final Checklist:**

✅ EC2 instance running  
✅ Node.js installed  
✅ Application code deployed  
✅ PM2 running application  
✅ Nginx configured (port hiding)  
✅ SSL certificate (if domain hai)  
✅ MongoDB setup (if needed)  
✅ Auto-restart on server reboot  

**Ab kya karo:**
1. Application test karo thoroughly
2. Monitoring setup karo (PM2 Plus, etc)
3. Regular backups lo
4. Security hardening karo
5. CDN setup karo (CloudFlare - free)

---

## Need Help?

**Common Resources:**
- Node.js Docs: https://nodejs.org/docs/
- PM2 Docs: https://pm2.keymetrics.io/
- Nginx Docs: https://nginx.org/en/docs/
- MongoDB Docs: https://docs.mongodb.com/

**YouTube channels for Hindi tutorials:**
- Code Step By Step
- Thapa Technical
- CodeWithHarry

---

**Last Updated:** February 2026  
**Version:** 2.0  

**Happy Coding! 🚀**
