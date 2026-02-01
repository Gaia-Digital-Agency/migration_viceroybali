# VM & LEMP Stack Setup Documentation

**Date:** January 2026
**Project:** gda-viceroy
**Server:** viceroy-bali

---

## Part 1: GCP VM Setup

### 1.1 Set GCP Project

```bash
gcloud config set project gda-viceroy
```

### 1.2 Create VM Instance

```bash
gcloud compute instances create viceroy-bali \
  --project=gda-viceroy \
  --zone=asia-southeast1-b \
  --machine-type=e2-standard-2 \
  --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=default \
  --maintenance-policy=MIGRATE \
  --provisioning-model=STANDARD \
  --tags=http-server,https-server \
  --create-disk=auto-delete=yes,boot=yes,device-name=viceroy-bali,image=projects/ubuntu-os-cloud/global/images/ubuntu-2404-noble-amd64-v20260115,mode=rw,size=50,type=pd-balanced \
  --no-shielded-secure-boot \
  --shielded-vtpm \
  --shielded-integrity-monitoring \
  --labels=environment=production,project=viceroy \
  --reservation-affinity=any
```

### 1.3 Reserve Static External IP

```bash
gcloud compute addresses create viceroy-bali-ip \
  --project=gda-viceroy \
  --region=asia-southeast1

gcloud compute instances delete-access-config viceroy-bali \
  --zone=asia-southeast1-b \
  --access-config-name="External NAT"

gcloud compute instances add-access-config viceroy-bali \
  --zone=asia-southeast1-b \
  --address=34.124.156.73
```

### 1.4 Create Storage Bucket

```bash
gcloud storage buckets create gs://viceroybali_bucket \
  --project=gda-viceroy \
  --location=asia-southeast1 \
  --default-storage-class=STANDARD \
  --uniform-bucket-level-access
```

### 1.5 SSH Key Setup

```bash
gcloud compute os-login ssh-keys add \
  --key-file=/Users/rogerwoolie/Container/dotfolders/.ssh/id_ed25519_gaia.pub
```

### 1.6 SSH into VM

```bash
ssh -i /Users/rogerwoolie/Container/dotfolders/.ssh/id_ed25519_gaia root@34.124.156.73
```

---

## Part 2: Initial Server Setup

### 2.1 Update System

```bash
apt update && apt upgrade -y
```

### 2.2 Set Timezone

```bash
timedatectl set-timezone Asia/Singapore
timedatectl
```

### 2.3 Set Hostname

```bash
hostnamectl set-hostname viceroy-bali
```

### 2.4 Create Swap File

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
swapon --show
```

---

## Part 3: Install Nginx

### 3.1 Install Nginx

```bash
apt install nginx -y
```

### 3.2 Start and Enable

```bash
systemctl start nginx
systemctl enable nginx
systemctl status nginx
```

### 3.3 Verify

```bash
nginx -v
curl -I http://localhost
```

---

## Part 4: Install MySQL

### 4.1 Install MySQL Server

```bash
apt install mysql-server -y
```

### 4.2 Start and Enable

```bash
systemctl start mysql
systemctl enable mysql
systemctl status mysql
```

### 4.3 Secure Installation

```bash
mysql_secure_installation
```

**Interactive prompts:**
```
Would you like to setup VALIDATE PASSWORD component? Y
Please enter 0 = LOW, 1 = MEDIUM and 2 = STRONG: 2
Remove anonymous users? Y
Disallow root login remotely? Y
Remove test database and access to it? Y
Reload privilege tables now? Y
```

### 4.4 Create Database and User

```bash
mysql -u root
```

```sql
CREATE DATABASE viceroy_db_name;
CREATE USER 'viceroy_user'@'localhost' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON viceroy_db_name.* TO 'viceroy_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 4.5 Verify Database

```bash
mysql -u viceroy_user -p viceroy_db_name -e "SHOW DATABASES;"
```

---

## Part 5: Install PHP 8.3

### 5.1 Add PHP Repository

```bash
apt install software-properties-common -y
add-apt-repository ppa:ondrej/php -y
apt update
```

### 5.2 Install PHP and Extensions

```bash
apt install php8.3-fpm php8.3-mysql php8.3-curl php8.3-gd php8.3-intl \
  php8.3-mbstring php8.3-soap php8.3-xml php8.3-zip php8.3-bcmath \
  php8.3-imagick php8.3-redis php8.3-opcache -y
```

### 5.3 Start and Enable

```bash
systemctl start php8.3-fpm
systemctl enable php8.3-fpm
systemctl status php8.3-fpm
```

### 5.4 Verify

```bash
php -v
php -m | grep -E 'mysql|redis|gd|curl'
```

### 5.5 Configure PHP

```bash
nano /etc/php/8.3/fpm/php.ini
```

**Find and modify these values:**
```ini
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
max_execution_time = 300
max_input_time = 300
max_input_vars = 3000
```

### 5.6 Apply Changes

```bash
systemctl restart php8.3-fpm
```

---

## Part 6: Install Redis

### 6.1 Install Redis

```bash
apt install redis-server -y
```

### 6.2 Configure Redis

```bash
nano /etc/redis/redis.conf
```

**Find and modify:**
```conf
supervised systemd
maxmemory 128mb
maxmemory-policy allkeys-lru
```

### 6.3 Start and Enable

```bash
systemctl restart redis-server
systemctl enable redis-server
systemctl status redis-server
```

### 6.4 Verify

```bash
redis-cli ping
```

**Expected output:** `PONG`

---

## Part 7: Create Web Directory

### 7.1 Create Directory

```bash
mkdir -p /var/www/viceroybali/public_html
```

### 7.2 Set Ownership

```bash
chown -R www-data:www-data /var/www/viceroybali
```

### 7.3 Set Permissions

```bash
find /var/www/viceroybali -type d -exec chmod 755 {} \;
find /var/www/viceroybali -type f -exec chmod 644 {} \;
```

---

## Part 8: Transfer Files from Hostinger

### 8.1 Upload Database Snapshot

```bash
# From local machine
scp -i ~/.ssh/id_ed25519_gaia viceroy_db_snapshot.sql root@34.124.156.73:/tmp/
```

### 8.2 Import Database

```bash
mysql -u viceroy_user -p viceroy_db_name < /tmp/viceroy_db_snapshot.sql
```

### 8.3 Verify Import

```bash
mysql -u viceroy_user -p viceroy_db_name -e "SHOW TABLES;" | head -20
```

### 8.4 Upload WordPress Files

```bash
# From local machine - upload wp-admin and wp-content
scp -i ~/.ssh/id_ed25519_gaia -r wp-admin root@34.124.156.73:/var/www/viceroybali/public_html/
scp -i ~/.ssh/id_ed25519_gaia -r wp-content root@34.124.156.73:/var/www/viceroybali/public_html/
```

### 8.5 Alternative: Using GCS Bucket

```bash
# Upload to bucket from local
gsutil -m cp -r wp-admin gs://viceroybali_bucket/
gsutil -m cp -r wp-content gs://viceroybali_bucket/

# Download from bucket on server
gsutil -m cp -r gs://viceroybali_bucket/wp-admin /var/www/viceroybali/public_html/
gsutil -m cp -r gs://viceroybali_bucket/wp-content /var/www/viceroybali/public_html/
```

---

## Part 9: Configure UFW Firewall

### 9.1 Install UFW

```bash
apt install ufw -y
```

### 9.2 Configure Rules

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow 'Nginx Full'
```

### 9.3 Enable Firewall

```bash
ufw enable
ufw status
```

**Output:**
```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
Nginx Full                 ALLOW       Anywhere
```

---

## Part 10: System Optimization

### 10.1 Increase File Limits

```bash
nano /etc/security/limits.conf
```

**Add at end:**
```
www-data soft nofile 65535
www-data hard nofile 65535
```

### 10.2 Optimize Nginx

```bash
nano /etc/nginx/nginx.conf
```

**Modify:**
```nginx
worker_processes auto;

events {
    worker_connections 2048;
    multi_accept on;
    use epoll;
}

http {
    # Gzip
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json application/javascript application/rss+xml application/atom+xml image/svg+xml;

    # Buffers
    client_body_buffer_size 10K;
    client_header_buffer_size 1k;
    client_max_body_size 64m;
    large_client_header_buffers 2 1k;
}
```

### 10.3 Apply Changes

```bash
nginx -t
systemctl restart nginx
```

---

## Part 11: Verify All Services

### 11.1 Check Services

```bash
systemctl status nginx
systemctl status php8.3-fpm
systemctl status mysql
systemctl status redis-server
```

### 11.2 Check Versions

```bash
nginx -v
php -v
mysql --version
redis-cli --version
```

### 11.3 Check Listening Ports

```bash
ss -tulpn | grep -E ':80|:443|:3306|:6379'
```

---

## Summary

| Component | Version | Port | Status |
|-----------|---------|------|--------|
| Nginx | 1.24.0 | 80, 443 | Running |
| PHP-FPM | 8.3.6 | socket | Running |
| MySQL | 8.x | 3306 | Running |
| Redis | 7.x | 6379 | Running |

---

## Configuration File Locations

| Config | Path |
|--------|------|
| Nginx main | `/etc/nginx/nginx.conf` |
| Nginx sites | `/etc/nginx/sites-available/` |
| PHP FPM | `/etc/php/8.3/fpm/php.ini` |
| PHP FPM pool | `/etc/php/8.3/fpm/pool.d/www.conf` |
| MySQL | `/etc/mysql/mysql.conf.d/mysqld.cnf` |
| Redis | `/etc/redis/redis.conf` |
| UFW | `/etc/ufw/` |

---

## Next Steps

After this setup, proceed with:
1. **step02.md** - WordPress core installation and wp-config.php
2. **step03.md** - Nginx server block, Load Balancer, CDN, and Firewall rules
