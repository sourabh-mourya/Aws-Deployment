# Node.js + MongoDB Deployment Guide on AWS EC2

A complete step-by-step guide for deploying a Node.js application with MongoDB on AWS EC2 without Docker.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: EC2 Instance Setup](#step-1-ec2-instance-setup)
- [Step 2: Connect to Server](#step-2-connect-to-server)
- [Step 3: System Update & Basic Tools](#step-3-system-update--basic-tools)
- [Step 4: Node.js Installation](#step-4-nodejs-installation)
- [Step 5: MongoDB Installation & Configuration](#step-5-mongodb-installation--configuration)
- [Step 6: Project Setup](#step-6-project-setup)
- [Step 7: PM2 Process Manager](#step-7-pm2-process-manager)
- [Step 8: Nginx Configuration](#step-8-nginx-configuration)
- [Step 9: SSL Certificate (Optional but Recommended)](#step-9-ssl-certificate-optional-but-recommended)
- [Step 10: Maintenance & Monitoring](#step-10-maintenance--monitoring)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- AWS Account
- Domain name (optional, but recommended for production)
- Basic knowledge of Linux commands
- Your Node.js application repository

---

## Step 1: EC2 Instance Setup

### Launch Instance from AWS Console

1. **Navigate to EC2 Dashboard**
   - Login to AWS Console
   - Go to EC2 Dashboard
   - Click **Launch Instance**

2. **Configure Instance Settings:**

   **Name and OS:**
   - Name: `nodejs-production-server`
   - **AMI:** Ubuntu 22.04 LTS (Free tier eligible)

   **Instance Type:**
   - **Recommended:** `t2.medium` (2 vCPU, 4GB RAM) - for production
   - **Minimum:** `t2.small` (1 vCPU, 2GB RAM) - for testing

3. **Key Pair:**
   - Create new key pair or select existing
   - Download and save the `.pem` file securely
   - **Important:** You cannot download it again!

4. **Network Settings - Security Group:**

   ```
   Port 22  (SSH)   - Source: Your IP only (e.g., 203.0.113.0/32)
   Port 80  (HTTP)  - Source: 0.0.0.0/0, ::/0
   Port 443 (HTTPS) - Source: 0.0.0.0/0, ::/0
   Port 8000        - Source: 0.0.0.0/0 (only for testing, remove later)
   ```

5. **Configure Storage:**
   - **Volume Size:** 20GB (minimum)
   - **Volume Type:** gp3 (General Purpose SSD)

6. Click **Launch Instance**

7. Note down your **Public IP Address** from the instance details

---

## Step 2: Connect to Server

### SSH Connection from Local Machine

**On Linux/Mac:**

```bash
# Set correct permissions for key file
chmod 400 your-key.pem

# Connect to EC2 instance
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```

**On Windows (using PowerShell):**

```powershell
# Connect to EC2 instance
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```

**Example:**

```bash
ssh -i mykey.pem ubuntu@13.233.45.67
```

**Alternative: Using PuTTY (Windows)**
1. Convert `.pem` to `.ppk` using PuTTYgen
2. Use the `.ppk` file in PuTTY for authentication

---

## Step 3: System Update & Basic Tools

```bash
# Update package lists and upgrade all packages
sudo apt update && sudo apt upgrade -y

# Install essential development tools
sudo apt install -y git curl wget vim nano software-properties-common build-essential

# Install Nginx
sudo apt install -y nginx

# Start and enable Nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

**Verify Nginx:**
Visit `http://your-ec2-public-ip` in browser - you should see Nginx welcome page.

---

## Step 4: Node.js Installation

### Install Node.js via NodeSource Repository

```bash
# Add NodeSource repository for Node.js 18.x LTS
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -

# Install Node.js and npm
sudo apt install -y nodejs

# Verify installation
node --version    # Should display v18.x.x
npm --version     # Should display npm version
```

**For different Node.js versions:**
- Node.js 20.x: Replace `setup_18.x` with `setup_20.x`
- Node.js 16.x: Replace `setup_18.x` with `setup_16.x`

---

## Step 5: MongoDB Installation & Configuration

### 5.1 Install MongoDB 7.0

```bash
# Import MongoDB public GPG key
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# Add MongoDB repository to sources list
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Update package database
sudo apt update

# Install MongoDB
sudo apt install -y mongodb-org

# Start MongoDB service
sudo systemctl start mongod

# Enable MongoDB to start on boot
sudo systemctl enable mongod

# Check MongoDB status
sudo systemctl status mongod
```

**Expected Output:** You should see `active (running)` in green.

---

### 5.2 MongoDB Security Configuration

#### Create Admin User

```bash
# Open MongoDB shell
mongosh

# Switch to admin database
use admin

# Create admin user with strong password
db.createUser({
  user: "admin",
  pwd: "YourStrongAdminPassword123!",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    "readWriteAnyDatabase"
  ]
})
```

#### Create Application User

```bash
# Still in mongosh, switch to your application database
use myapp_db

# Create application-specific user
db.createUser({
  user: "app_user",
  pwd: "YourStrongAppPassword456!",
  roles: [
    { role: "readWrite", db: "myapp_db" }
  ]
})

# Exit MongoDB shell
exit
```

**Important Security Notes:**
- Replace `YourStrongAdminPassword123!` with a strong password
- Replace `YourStrongAppPassword456!` with a different strong password
- Replace `myapp_db` with your actual database name
- Use a password generator for production environments

---

### 5.3 Enable MongoDB Authentication

```bash
# Edit MongoDB configuration file
sudo nano /etc/mongod.conf
```

**Find the `#security:` section and modify it:**

```yaml
security:
  authorization: enabled
```

**Save and exit:**
- Press `Ctrl + X`
- Press `Y` to confirm
- Press `Enter`

**Restart MongoDB:**

```bash
# Restart MongoDB to apply changes
sudo systemctl restart mongod

# Test authentication with admin user
mongosh -u admin -p YourStrongAdminPassword123! --authenticationDatabase admin
```

If login is successful, authentication is working correctly. Type `exit` to logout.

---

## Step 6: Project Setup

### 6.1 Clone Your Repository

```bash
# Navigate to home directory
cd /home/ubuntu

# Clone your Node.js project
git clone https://github.com/yourusername/nodejs-project.git

# Navigate to project directory
cd nodejs-project
```

**Alternative: Upload project via SCP**

```bash
# From your local machine
scp -i your-key.pem -r /path/to/your/project ubuntu@your-ec2-ip:/home/ubuntu/
```

---

### 6.2 Install Dependencies

```bash
# Install all dependencies
npm install

# For production (installs only production dependencies)
npm install --production
```

---

### 6.3 Environment Configuration

Create environment variables file:

```bash
# Create .env file
nano .env
```

**Add the following configuration:**

```env
# Application Environment
NODE_ENV=production
PORT=3000

# MongoDB Connection
MONGO_URI=mongodb://app_user:YourStrongAppPassword456!@localhost:27017/myapp_db?authSource=myapp_db

# Application Settings
APP_NAME=MyNodeApp
SECRET_KEY=your-secret-jwt-key-here

# CORS Settings (if needed)
ALLOWED_ORIGINS=https://yourdomain.com,https://www.yourdomain.com

# Other configurations as per your app
```

**Save and exit:** `Ctrl + X`, then `Y`, then `Enter`

**Security Best Practice:**

```bash
# Add .env to .gitignore to prevent committing secrets
echo ".env" >> .gitignore

# Set proper permissions
chmod 600 .env
```

---

### 6.4 Test Your Application

```bash
# Test if application starts correctly
node server.js
# or
npm start
```

**Access your app:** `http://your-ec2-ip:3000`

If it works, press `Ctrl + C` to stop the server. We'll use PM2 next for production.

---

## Step 7: PM2 Process Manager

PM2 is a production process manager for Node.js applications. It keeps your app running in the background, restarts it on crashes, and starts automatically on server reboot.

### 7.1 Install PM2 Globally

```bash
# Install PM2 globally
sudo npm install -g pm2
```

---

### 7.2 Start Application with PM2

```bash
# Start your application
pm2 start server.js --name "nodejs-app"

# If you use npm start, use this instead:
pm2 start npm --name "nodejs-app" -- start
```

---

### 7.3 Configure PM2 for Auto-Restart on Server Reboot

```bash
# Generate startup script
pm2 startup systemd

# Copy and run the command shown in output
# It will look something like this:
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u ubuntu --hp /home/ubuntu

# Save current PM2 process list
pm2 save
```

**What this does:**
- Creates a systemd service for PM2
- PM2 will start automatically when server reboots
- All saved applications will restart automatically

---

### 7.4 PM2 Useful Commands

```bash
# Check application status
pm2 status

# View logs
pm2 logs nodejs-app

# View logs in real-time
pm2 logs nodejs-app --lines 100

# Monitor CPU and memory usage
pm2 monit

# Restart application
pm2 restart nodejs-app

# Stop application
pm2 stop nodejs-app

# Delete application from PM2
pm2 delete nodejs-app

# Restart all applications
pm2 restart all

# Show detailed information
pm2 info nodejs-app
```

---

### 7.5 PM2 Ecosystem File (Advanced Configuration)

Create `ecosystem.config.js` for advanced configuration:

```bash
nano ecosystem.config.js
```

```javascript
module.exports = {
  apps: [{
    name: 'nodejs-app',
    script: './server.js',
    instances: 2,  // or 'max' for all CPU cores
    exec_mode: 'cluster',
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    },
    error_file: './logs/err.log',
    out_file: './logs/out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    max_memory_restart: '1G',
    autorestart: true,
    watch: false
  }]
};
```

**Use ecosystem file:**

```bash
# Create logs directory
mkdir -p logs

# Start with ecosystem file
pm2 start ecosystem.config.js

# Save configuration
pm2 save
```

---

## Step 8: Nginx Configuration

Nginx acts as a reverse proxy, forwarding requests to your Node.js application and hiding the port number from URLs.

### 8.1 Create Nginx Configuration

```bash
# Create new Nginx configuration file
sudo nano /etc/nginx/sites-available/nodejs_app
```

**Add the following configuration:**

```nginx
server {
    listen 80;
    listen [::]:80;
    
    server_name yourdomain.com www.yourdomain.com your-ec2-ip;

    # Request size limit
    client_max_body_size 50M;

    # Logging
    access_log /var/log/nginx/nodejs_app_access.log;
    error_log /var/log/nginx/nodejs_app_error.log;

    location / {
        # Proxy to Node.js application
        proxy_pass http://localhost:3000;
        
        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_cache_bypass $http_upgrade;
        
        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Optional: Serve static files directly from Nginx
    location /static/ {
        alias /home/ubuntu/nodejs-project/public/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

**Replace:**
- `yourdomain.com` with your actual domain name
- `your-ec2-ip` with your EC2 public IP
- `3000` with your application port
- `/home/ubuntu/nodejs-project/` with your project path

**Save and exit:** `Ctrl + X`, then `Y`, then `Enter`

---

### 8.2 Enable Configuration

```bash
# Create symbolic link to enable the site
sudo ln -s /etc/nginx/sites-available/nodejs_app /etc/nginx/sites-enabled/

# Remove default Nginx configuration
sudo rm /etc/nginx/sites-enabled/default

# Test Nginx configuration for syntax errors
sudo nginx -t

# If test is successful, restart Nginx
sudo systemctl restart nginx

# Check Nginx status
sudo systemctl status nginx
```

**Expected output from `nginx -t`:**
```
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

### 8.3 Verify Deployment

Visit your website:
- `http://your-ec2-ip` (without port number)
- `http://yourdomain.com` (if domain is configured)

You should see your Node.js application running!

---

## Step 9: SSL Certificate (Optional but Recommended)

Secure your website with free SSL certificate from Let's Encrypt.

### 9.1 Install Certbot

```bash
# Install Certbot and Nginx plugin
sudo apt install -y certbot python3-certbot-nginx
```

---

### 9.2 Obtain SSL Certificate

**Prerequisites:**
- Domain name pointing to your EC2 IP address
- Nginx already configured (from Step 8)

```bash
# Obtain and install SSL certificate
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

**Follow the prompts:**
1. Enter email address (for renewal notifications)
2. Agree to Terms of Service
3. Choose whether to redirect HTTP to HTTPS (recommended: Yes)

Certbot will automatically:
- Obtain SSL certificate
- Update Nginx configuration
- Set up auto-renewal

---

### 9.3 Auto-Renewal Test

```bash
# Test auto-renewal
sudo certbot renew --dry-run
```

If successful, your certificates will auto-renew before expiration.

**Manual renewal (if needed):**

```bash
sudo certbot renew
```

---

### 9.4 Updated Nginx Configuration (After SSL)

After running Certbot, your Nginx config will look like:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name yourdomain.com www.yourdomain.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    
    server_name yourdomain.com www.yourdomain.com;

    # SSL certificates (managed by Certbot)
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    client_max_body_size 50M;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## Step 10: Maintenance & Monitoring

### 10.1 Application Updates

```bash
# Navigate to project directory
cd /home/ubuntu/nodejs-project

# Pull latest changes from Git
git pull origin main

# Install any new dependencies
npm install --production

# Restart application with PM2
pm2 restart nodejs-app

# Alternative: Reload with zero downtime
pm2 reload nodejs-app
```

---

### 10.2 View Logs

```bash
# PM2 logs
pm2 logs nodejs-app

# PM2 logs (last 100 lines)
pm2 logs nodejs-app --lines 100

# Nginx access logs
sudo tail -f /var/log/nginx/nodejs_app_access.log

# Nginx error logs
sudo tail -f /var/log/nginx/nodejs_app_error.log

# MongoDB logs
sudo tail -f /var/log/mongodb/mongod.log

# System logs
sudo journalctl -u mongod -f
```

---

### 10.3 Database Backup

```bash
# Create backup directory
mkdir -p /home/ubuntu/backups

# Backup MongoDB database
mongodump --uri="mongodb://app_user:YourStrongAppPassword456!@localhost:27017/myapp_db?authSource=myapp_db" --out=/home/ubuntu/backups/$(date +%Y%m%d)

# Automated backup with cron
crontab -e
```

**Add this line for daily backup at 2 AM:**

```cron
0 2 * * * mongodump --uri="mongodb://app_user:YourStrongAppPassword456!@localhost:27017/myapp_db?authSource=myapp_db" --out=/home/ubuntu/backups/$(date +\%Y\%m\%d) >> /home/ubuntu/backup.log 2>&1
```

**Restore database:**

```bash
mongorestore --uri="mongodb://app_user:YourStrongAppPassword456!@localhost:27017/myapp_db?authSource=myapp_db" /home/ubuntu/backups/20250207/myapp_db
```

---

### 10.4 System Monitoring

```bash
# Check disk space
df -h

# Check memory usage
free -h

# Check CPU usage
top

# Check running processes
htop  # Install with: sudo apt install htop

# PM2 monitoring dashboard
pm2 monit

# Check open ports
sudo netstat -tulpn

# Check Nginx status
sudo systemctl status nginx

# Check MongoDB status
sudo systemctl status mongod
```

---

## Troubleshooting

### Application Not Starting

```bash
# Check PM2 logs
pm2 logs nodejs-app --err

# Check if port is already in use
sudo lsof -i :3000

# Kill process on port 3000
sudo kill -9 $(sudo lsof -t -i:3000)

# Restart application
pm2 restart nodejs-app
```

---

### MongoDB Connection Issues

```bash
# Check MongoDB status
sudo systemctl status mongod

# Check MongoDB logs
sudo tail -50 /var/log/mongodb/mongod.log

# Test MongoDB connection
mongosh -u admin -p YourStrongAdminPassword123! --authenticationDatabase admin

# Restart MongoDB
sudo systemctl restart mongod

# Check if MongoDB is listening
sudo netstat -tulpn | grep mongod
```

---

### Nginx Issues

```bash
# Test Nginx configuration
sudo nginx -t

# Check Nginx status
sudo systemctl status nginx

# Restart Nginx
sudo systemctl restart nginx

# Check Nginx error logs
sudo tail -50 /var/log/nginx/error.log

# Check if port 80/443 is open
sudo netstat -tulpn | grep nginx
```

---

### Port Already in Use

```bash
# Find process using port 3000
sudo lsof -i :3000

# Kill the process
sudo kill -9 <PID>

# Or kill all node processes
sudo pkill node

# Restart with PM2
pm2 restart nodejs-app
```

---

### High Memory Usage

```bash
# Check memory
free -h

# Limit PM2 memory
pm2 restart nodejs-app --max-memory-restart 500M

# Or update ecosystem.config.js
max_memory_restart: '500M'
```

---

### SSL Certificate Issues

```bash
# Check certificate expiry
sudo certbot certificates

# Renew certificates manually
sudo certbot renew

# Restart Nginx after renewal
sudo systemctl restart nginx
```

---

### Permission Denied Errors

```bash
# Fix ownership of project files
sudo chown -R ubuntu:ubuntu /home/ubuntu/nodejs-project

# Fix permissions
chmod -R 755 /home/ubuntu/nodejs-project

# For .env file
chmod 600 /home/ubuntu/nodejs-project/.env
```

---

### Application Crashes on Boot

```bash
# Check PM2 startup status
pm2 status

# Regenerate startup script
pm2 unstartup systemd
pm2 startup systemd

# Run the command shown, then save
pm2 save

# Reboot and verify
sudo reboot
```

---

## Security Best Practices

1. **Firewall Configuration:**

```bash
# Install UFW (Uncomplicated Firewall)
sudo apt install ufw

# Allow SSH (change 22 to your custom port if changed)
sudo ufw allow 22/tcp

# Allow HTTP and HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status
```

2. **Keep System Updated:**

```bash
# Regular updates
sudo apt update && sudo apt upgrade -y

# Auto-updates
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

3. **Disable Root Login:**

```bash
sudo nano /etc/ssh/sshd_config

# Set:
PermitRootLogin no
PasswordAuthentication no

sudo systemctl restart ssh
```

4. **Use Environment Variables:**
   - Never hardcode credentials in code
   - Use `.env` file for secrets
   - Add `.env` to `.gitignore`

5. **MongoDB Security:**
   - Always enable authentication
   - Use strong passwords
   - Create specific users with limited permissions
   - Bind MongoDB to localhost only

---

## Performance Optimization

### 1. Enable Gzip Compression in Nginx

```bash
sudo nano /etc/nginx/nginx.conf
```

**Add inside http block:**

```nginx
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml+rss application/json;
```

### 2. Enable PM2 Cluster Mode

```javascript
// In ecosystem.config.js
instances: 'max',  // Use all CPU cores
exec_mode: 'cluster'
```

### 3. MongoDB Indexes

```javascript
// In your Node.js application
// Create indexes on frequently queried fields
db.collection.createIndex({ email: 1 }, { unique: true });
db.collection.createIndex({ createdAt: -1 });
```

---

## Conclusion

Your Node.js application with MongoDB is now deployed on AWS EC2! 

**Key Points to Remember:**
- Keep your system and packages updated
- Monitor logs regularly
- Backup database frequently
- Use strong passwords
- Enable SSL for production
- Monitor resource usage

**Next Steps:**
- Set up monitoring (e.g., PM2 Plus, New Relic)
- Configure CDN for static assets
- Set up CI/CD pipeline
- Implement load balancing for scaling

---

## Quick Reference Commands

```bash
# PM2
pm2 status
pm2 restart nodejs-app
pm2 logs nodejs-app

# Nginx
sudo systemctl restart nginx
sudo nginx -t

# MongoDB
sudo systemctl restart mongod
mongosh -u admin -p --authenticationDatabase admin

# System
sudo systemctl status nginx
sudo systemctl status mongod
df -h
free -h

# Updates
cd /home/ubuntu/nodejs-project
git pull
npm install --production
pm2 restart nodejs-app
```

---

**For questions or issues, refer to:**
- [Node.js Documentation](https://nodejs.org/docs/)
- [MongoDB Documentation](https://docs.mongodb.com/)
- [PM2 Documentation](https://pm2.keymetrics.io/)
- [Nginx Documentation](https://nginx.org/en/docs/)

---

**Last Updated:** February 2026
**Version:** 1.0
