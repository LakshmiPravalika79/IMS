# AWS EC2 Free Tier Deployment Guide - Inventory Management System

## Overview
Deploy your full-stack application on AWS using the **Free Tier** with a single EC2 instance running Docker Compose. This is the most cost-effective way to get started!

### ✅ **What's Included (FREE for 12 months):**
- **t2.micro EC2 instance** (1 vCPU, 1 GB RAM) - **FREE**
- **30 GB EBS storage** - **FREE**
- **15 GB data transfer** - **FREE**
- **Static IP (Elastic IP)** - **FREE** (if attached to running instance)

### 💰 **Total Cost: $0/month** (within free tier limits)

---

## Phase 1: AWS Account Setup

### 1.1 Create AWS Account
1. Go to [aws.amazon.com](https://aws.amazon.com)
2. Click "Create an AWS Account"
3. Follow the signup process (requires credit card for verification)
4. Choose **Basic Support Plan** (FREE)

### 1.2 Install AWS CLI (Optional)
```powershell
# Download from: https://aws.amazon.com/cli/
aws --version
aws configure
```

---

## Phase 2: Launch EC2 Instance

### 2.1 Launch Instance via AWS Console
1. **Login to AWS Console** → Navigate to **EC2 Dashboard**
2. Click **"Launch Instance"**
3. **Choose AMI**: Select **"Ubuntu Server 22.04 LTS"** (Free tier eligible)
4. **Instance Type**: Select **"t2.micro"** (Free tier eligible)
5. **Key Pair**: 
   - Create new key pair named `ims-keypair`
   - Download the `.pem` file and save it securely
6. **Security Group**: Create new with these rules:
   ```
   Type        Protocol    Port Range    Source
   SSH         TCP         22           0.0.0.0/0
   HTTP        TCP         80           0.0.0.0/0
   HTTPS       TCP         443          0.0.0.0/0
   Custom TCP  TCP         3000         0.0.0.0/0  (Frontend)
   Custom TCP  TCP         5050         0.0.0.0/0  (Backend API)
   ```
7. **Storage**: Keep default **8 GB** (can increase to 30 GB for free)
8. Click **"Launch Instance"**

### 2.2 Allocate Elastic IP (Static IP)
1. Go to **EC2 Dashboard** → **Elastic IPs**
2. Click **"Allocate Elastic IP address"**
3. Click **"Allocate"**
4. Select the new IP → **Actions** → **Associate Elastic IP address**
5. Choose your EC2 instance → **Associate**

---

## Phase 3: Connect to EC2 Instance

### 3.1 Connect via SSH (Windows)
```powershell
# Method 1: Using built-in SSH (Windows 10+)
ssh -i "path\to\ims-keypair.pem" ubuntu@<your-elastic-ip>

# Method 2: Using PuTTY (if preferred)
# Convert .pem to .ppk using PuTTYgen first
```

### 3.2 Connect via AWS Console (Easy method)
1. Go to **EC2 Dashboard** → **Instances**
2. Select your instance → **Connect**
3. Choose **"EC2 Instance Connect"** → **Connect**

---

## Phase 4: Setup Server Environment

### 4.1 Update System and Install Dependencies
```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker ubuntu
newgrp docker

# Install Docker Compose
sudo apt install docker-compose -y

# Install Git
sudo apt install git -y

# Install Nginx (for SSL and domain setup later)
sudo apt install nginx -y
```

### 4.2 Verify Installations
```bash
docker --version
docker-compose --version
git --version
nginx -v
```

---

## Phase 5: Deploy Your Application

### 5.1 Clone Your Repository
```bash
# Clone your repository
git clone https://github.com/LakshmiPravalika79/IMS.git
cd IMS

# Verify files are there
ls -la
```

### 5.2 Create Production Environment File
```bash
# Create environment file for production
nano .env.production
```

Add this content:
```env
# Database Configuration
MYSQL_ROOT_PASSWORD=your-secure-root-password
MYSQL_DATABASE=inventorydb
MYSQL_USER=ims_user
MYSQL_PASSWORD=your-secure-user-password

# Backend Configuration
SPRING_PROFILES_ACTIVE=production
DB_HOST=inventory-db
DB_PORT=3306
DB_NAME=inventorydb
DB_USERNAME=ims_user
DB_PASSWORD=your-secure-user-password
JWT_SECRET=your-super-secret-jwt-key-at-least-32-characters-long

# Frontend Configuration
REACT_APP_API_URL=http://<your-elastic-ip>:5050
```

### 5.3 Create Production Docker Compose
```bash
nano docker-compose.prod.yml
```

Add this content:
```yaml
version: '3.8'

services:
  # MySQL Database
  inventory-db:
    image: mysql:8.0
    container_name: inventory-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 20s
      retries: 10
    networks:
      - ims-network

  # Backend Service
  backend:
    build: ./backend
    container_name: backend
    restart: unless-stopped
    ports:
      - "5050:5050"
    environment:
      - SPRING_PROFILES_ACTIVE=production
      - SPRING_DATASOURCE_URL=jdbc:mysql://inventory-db:3306/${MYSQL_DATABASE}
      - SPRING_DATASOURCE_USERNAME=${MYSQL_USER}
      - SPRING_DATASOURCE_PASSWORD=${MYSQL_PASSWORD}
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      inventory-db:
        condition: service_healthy
    networks:
      - ims-network

  # Frontend Service
  frontend:
    build: ./frontend
    container_name: frontend
    restart: unless-stopped
    ports:
      - "3000:80"
    environment:
      - REACT_APP_API_URL=http://<your-elastic-ip>:5050
    depends_on:
      - backend
    networks:
      - ims-network

volumes:
  mysql_data:

networks:
  ims-network:
    driver: bridge
```

### 5.4 Update Frontend API Configuration
```bash
# Edit the API service to use production URL
nano frontend/src/service/ApiService.js
```

Update the BASE_URL:
```javascript
// Use environment variable or fallback to production IP
const BASE_URL = process.env.REACT_APP_API_URL || 'http://<your-elastic-ip>:5050';
```

### 5.5 Create Production Application Properties
```bash
nano backend/src/main/resources/application-production.properties
```

Add this content:
```properties
# Database configuration
spring.datasource.url=jdbc:mysql://${DB_HOST:localhost}:${DB_PORT:3306}/${DB_NAME:inventorydb}
spring.datasource.username=${DB_USERNAME:root}
spring.datasource.password=${DB_PASSWORD:password}
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA/Hibernate settings
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect

# Server configuration
server.port=5050

# JWT configuration
jwt.secret=${JWT_SECRET:default-secret-key}
jwt.expiration=86400000

# CORS configuration
cors.allowed.origins=http://<your-elastic-ip>:3000

# Logging
logging.level.com.phegondev=INFO
logging.level.org.springframework.security=WARN
```

---

## Phase 6: Deploy and Start Application

### 6.1 Build and Start Services
```bash
# Make sure you're in the project directory
cd ~/IMS

# Load environment variables
export $(cat .env.production | xargs)

# Build and start all services
docker-compose -f docker-compose.prod.yml up --build -d

# Check if services are running
docker-compose -f docker-compose.prod.yml ps
```

### 6.2 Check Logs
```bash
# Check all services
docker-compose -f docker-compose.prod.yml logs

# Check specific service
docker-compose -f docker-compose.prod.yml logs backend
docker-compose -f docker-compose.prod.yml logs frontend
docker-compose -f docker-compose.prod.yml logs inventory-db
```

### 6.3 Verify Application is Running
```bash
# Test backend API
curl http://localhost:5050/api/auth/test

# Test frontend
curl http://localhost:3000

# Check database connection
docker exec -it inventory-db mysql -u ims_user -p inventorydb -e "SHOW TABLES;"
```

---

## Phase 7: Access Your Application

### 7.1 Open in Browser
- **Frontend**: `http://<your-elastic-ip>:3000`
- **Backend API**: `http://<your-elastic-ip>:5050/api/auth/test`

### 7.2 Test Registration and Login
1. Go to `http://<your-elastic-ip>:3000`
2. Click **Register**
3. Fill in the form with role selection
4. Test login functionality

---

## Phase 8: Production Optimizations (Optional)

### 8.1 Setup Nginx Reverse Proxy
```bash
# Create Nginx configuration
sudo nano /etc/nginx/sites-available/ims
```

Add this configuration:
```nginx
server {
    listen 80;
    server_name <your-elastic-ip>;

    # Frontend
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Backend API
    location /api/ {
        proxy_pass http://localhost:5050/api/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/ims /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### 8.2 Setup Domain Name (Optional)
1. **Buy a domain** from providers like Namecheap, GoDaddy
2. **Point domain to your Elastic IP**:
   - Create A record: `yourdomain.com` → `<your-elastic-ip>`
   - Create A record: `www.yourdomain.com` → `<your-elastic-ip>`

### 8.3 Setup SSL Certificate (Free with Let's Encrypt)
```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Get SSL certificate (replace with your domain)
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Auto-renewal (runs automatically)
sudo systemctl status certbot.timer
```

---

## Phase 9: Monitoring and Maintenance

### 9.1 Setup Auto-start on Boot
```bash
# Create systemd service for auto-start
sudo nano /etc/systemd/system/ims-app.service
```

Add this content:
```ini
[Unit]
Description=IMS Application
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/ubuntu/IMS
ExecStart=/usr/bin/docker-compose -f docker-compose.prod.yml up -d
ExecStop=/usr/bin/docker-compose -f docker-compose.prod.yml down
TimeoutStartSec=0
User=ubuntu

[Install]
WantedBy=multi-user.target
```

Enable the service:
```bash
sudo systemctl enable ims-app.service
sudo systemctl start ims-app.service
```

### 9.2 Setup Log Rotation
```bash
# Create log rotation configuration
sudo nano /etc/logrotate.d/docker-containers
```

Add this content:
```
/var/lib/docker/containers/*/*.log {
    rotate 7
    daily
    compress
    size=1M
    missingok
    delaycompress
    copytruncate
}
```

### 9.3 Basic Monitoring Scripts
```bash
# Create monitoring script
nano ~/monitor.sh
```

Add this content:
```bash
#!/bin/bash
echo "=== System Resources ==="
free -h
df -h
echo ""
echo "=== Docker Status ==="
docker ps
echo ""
echo "=== Application Health ==="
curl -s http://localhost:5050/api/auth/test || echo "Backend DOWN"
curl -s http://localhost:3000 > /dev/null && echo "Frontend UP" || echo "Frontend DOWN"
```

Make it executable:
```bash
chmod +x ~/monitor.sh
./monitor.sh
```

---

## Phase 10: Backup and Updates

### 10.1 Database Backup
```bash
# Create backup script
nano ~/backup-db.sh
```

Add this content:
```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
docker exec inventory-db mysqldump -u ims_user -p<password> inventorydb > ~/backup_${DATE}.sql
echo "Database backup created: backup_${DATE}.sql"
```

### 10.2 Application Updates
```bash
# Update application
cd ~/IMS
git pull origin main
docker-compose -f docker-compose.prod.yml down
docker-compose -f docker-compose.prod.yml up --build -d
```

---

## Troubleshooting

### Common Issues:

#### 1. **Port Already in Use**
```bash
# Find process using port
sudo netstat -tulnp | grep :3000
sudo kill -9 <process-id>
```

#### 2. **Database Connection Issues**
```bash
# Check database logs
docker logs inventory-db

# Connect to database manually
docker exec -it inventory-db mysql -u root -p
```

#### 3. **Out of Memory**
```bash
# Check memory usage
free -h
docker stats

# Restart services if needed
docker-compose -f docker-compose.prod.yml restart
```

#### 4. **SSL Certificate Issues**
```bash
# Renew certificate
sudo certbot renew --dry-run
sudo certbot renew
```

---

## Cost Monitoring

### Stay Within Free Tier:
- **EC2 Usage**: 750 hours/month (24/7 for one t2.micro)
- **Storage**: Stay under 30 GB
- **Data Transfer**: Monitor monthly usage in billing dashboard

### Check Your Usage:
1. **AWS Console** → **Billing Dashboard**
2. **Free Tier Usage** → Monitor your limits
3. Set up **billing alerts** for $1-5 to get warnings

---

## Security Best Practices

### 10.1 Secure SSH Access
```bash
# Disable password authentication (use keys only)
sudo nano /etc/ssh/sshd_config
# Set: PasswordAuthentication no
sudo systemctl restart sshd
```

### 10.2 Setup Firewall
```bash
# Enable UFW firewall
sudo ufw enable
sudo ufw allow 22    # SSH
sudo ufw allow 80    # HTTP
sudo ufw allow 443   # HTTPS
sudo ufw allow 3000  # Frontend
sudo ufw allow 5050  # Backend
sudo ufw status
```

### 10.3 Regular Updates
```bash
# Setup automatic security updates
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

---

## 🎉 **Congratulations!**

Your full-stack application is now running on AWS EC2 **completely FREE** (within free tier limits)!

### 📱 **Access Your App:**
- **Application**: `http://<your-elastic-ip>:3000`
- **API**: `http://<your-elastic-ip>:5050/api`

### 🚀 **What You've Accomplished:**
- ✅ Deployed full-stack app on AWS
- ✅ Using Docker containerization
- ✅ MySQL database with persistent storage
- ✅ Auto-restart on server reboot
- ✅ Basic monitoring and backups
- ✅ **$0/month cost** (free tier)

### 🔄 **Next Steps:**
- Add your own domain name
- Setup SSL certificate
- Configure automated backups
- Add monitoring alerts
- Scale to multiple instances (when ready)

**Your application is now live and accessible from anywhere in the world!** 🌍