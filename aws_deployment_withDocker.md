# Node.js Deployment with Docker on AWS EC2
### Production Deployment Guide · Ubuntu 22.04 LTS · Docker · Nginx · SSL/HTTPS

---

## Prerequisites

- ✅ AWS account with EC2 access
- ✅ Node.js / Express app ready to deploy
- ✅ Basic understanding of Docker (recommended)
- ✅ GitHub repository (optional but recommended)
- ✅ Domain name (optional, required for SSL)

---

## Table of Contents

1. [EC2 Instance Launch](#1-ec2-instance-launch)
2. [Connect to EC2](#2-connect-to-ec2)
3. [Install Docker & Dependencies](#3-install-docker--dependencies)
4. [Project Setup](#4-project-setup)
5. [Dockerize the Node.js App](#5-dockerize-the-nodejs-app)
6. [Build & Run Docker Container](#6-build--run-docker-container)
7. [Docker Compose Setup (Recommended)](#7-docker-compose-setup-recommended)
8. [Nginx Reverse Proxy (Optional)](#8-nginx-reverse-proxy-optional)
9. [SSL with Let's Encrypt](#9-ssl-with-lets-encrypt)
10. [Deployment Workflow](#10-deployment-workflow)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. EC2 Instance Launch

### Step 1.1 — Create EC2 Instance

Navigate to **AWS Console → EC2 → Launch Instance** and configure as follows:

- **AMI:** Ubuntu 22.04 LTS
- **Instance Type:** t2.micro or t2.small
- **Storage:** 20 GB gp3
- **Security Group inbound rules:**
  - SSH — port 22
  - HTTP — port 80
  - HTTPS — port 443
  - Custom TCP — port 3000 *(optional, for testing)*

---

## 2. Connect to EC2

Set correct permissions on your key file and SSH into the instance:

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@your-ec2-ip
```

---

## 3. Install Docker & Dependencies

**Update system packages**

```bash
sudo apt update && sudo apt upgrade -y
```

**Install Docker**

```bash
sudo apt install -y docker.io
```

**Start & enable Docker service**

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

**Add user to docker group**

```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

**Verify installation**

```bash
docker --version
```

---

## 4. Project Setup

Clone your repository onto the server:

```bash
cd /home/ubuntu
git clone https://github.com/yourusername/your-app.git
cd your-app
```

---

## 5. Dockerize the Node.js App

### Dockerfile

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install --production

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

### .dockerignore

```
node_modules
npm-debug.log
.env
.git
```

---

## 6. Build & Run Docker Container

**Build the image**

```bash
docker build -t nodejs-app .
```

**Run the container**

```bash
docker run -d \
  --name nodejs-app \
  -p 3000:3000 \
  --restart always \
  nodejs-app
```

**Check running containers**

```bash
docker ps
```

> 💡 **Tip:** Once running, access your app at `http://<your-ec2-ip>:3000` — make sure port 3000 is allowed in your Security Group.

---

## 7. Docker Compose Setup (Recommended)

**Install Docker Compose**

```bash
sudo apt install -y docker-compose
```

**docker-compose.yml**

```yaml
version: "3.8"

services:
  app:
    build: .
    container_name: nodejs-app
    ports:
      - "3000:3000"
    restart: always
    environment:
      - NODE_ENV=production
      - PORT=3000
    volumes:
      - .:/app
      - /app/node_modules
```

**Start services**

```bash
docker-compose up -d --build
```

**Stop services**

```bash
docker-compose down
```

> 💡 **Tip:** To hide the port and serve on port 80, map the container port to 80:
> ```yaml
> ports:
>   - "80:3000"
> ```
> Users can then access your app at `http://<your-ec2-ip>` without specifying a port number.

---

## 8. Nginx Reverse Proxy (Optional)

Install Nginx to proxy traffic from port 80 to your Node.js container:

```bash
sudo apt install -y nginx

# Edit the default site config
sudo nano /etc/nginx/sites-available/default
```

**Nginx config snippet**

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
sudo systemctl restart nginx
```

---

## 9. SSL with Let's Encrypt

**Install Certbot**

```bash
sudo apt install -y certbot python3-certbot-nginx
```

**Generate SSL certificate**

```bash
sudo certbot --nginx -d your-domain.com
```

> ⚠️ **Note:** A valid domain name pointing to your EC2 IP address is required before running Certbot. Certbot will automatically update your Nginx config for HTTPS.

---

## 10. Deployment Workflow

Typical workflow after the initial setup is complete:

1. Push changes to your GitHub repository
2. SSH into EC2: `ssh -i your-key.pem ubuntu@your-ec2-ip`
3. Pull latest code: `git pull origin main`
4. Rebuild and restart: `docker-compose up -d --build`

---

## 11. Troubleshooting

**View container logs**

```bash
docker logs nodejs-app
docker logs -f nodejs-app   # follow / live tail
```

**Enter running container**

```bash
docker exec -it nodejs-app sh
```

**Restart container**

```bash
docker restart nodejs-app
```

**Remove and rebuild**

```bash
docker-compose down
docker-compose up -d --build
```

**Check Nginx errors**

```bash
sudo nginx -t
sudo journalctl -u nginx --no-pager -n 50
```

---

*Node.js · Docker · AWS EC2 Production Guide*
