# Nginx Configuration

**Project:** Viceroy Bali Migration
**Web Server:** Nginx 1.24.0
**PHP Version:** 8.3-FPM
**Date:** January 31, 2026

---

## Overview

Nginx configured as high-performance web server for WordPress, optimized for:
- Static file caching
- PHP-FPM integration
- Security headers
- Gzip compression
- Load Balancer health checks

---

## Nginx Installation

Nginx was installed via Ubuntu package manager during initial server setup.

```bash
# Install Nginx
apt update
apt install -y nginx

# Start and enable Nginx
systemctl start nginx
systemctl enable nginx

# Verify installation
nginx -v
# nginx version: nginx/1.24.0 (Ubuntu)
```

---

## Site Configuration

### Location
- **Available Sites:** `/etc/nginx/sites-available/viceroybali`
- **Enabled Sites:** `/etc/nginx/sites-enabled/viceroybali` (symlink)
- **Configuration Test:** `nginx -t`
- **Reload:** `systemctl reload nginx`

### Complete Configuration

```nginx
# Viceroy Bali WordPress Configuration
# Optimized for high-performance

server {
    listen 80;
    listen [::]:80;
    server_name www.viceroybali.com viceroybali.com 34.158.47.112;

    root /var/www/viceroybali/public_html;
    index index.php index.html index.htm;

    # Logging
    access_log /var/log/nginx/viceroybali_access.log;
    error_log /var/log/nginx/viceroybali_error.log;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json application/javascript application/rss+xml application/atom+xml image/svg+xml;

    # Client body size (for uploads)
    client_max_body_size 64M;

    # WordPress permalinks
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # PHP processing
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_read_timeout 300;
        fastcgi_send_timeout 300;
    }

    # Deny access to sensitive files
    location ~ /\.ht { deny all; }
    location ~ /\.git { deny all; }
    location = /wp-config.php { deny all; }

    # Static file caching
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|pdf|webp|woff|woff2|ttf|svg|eot)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Block xmlrpc (security)
    location = /xmlrpc.php { deny all; access_log off; log_not_found off; }

    # Health check endpoint for load balancer
    location = /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # Robots.txt and Favicon
    location = /robots.txt { allow all; log_not_found off; access_log off; }
    location = /favicon.ico { log_not_found off; access_log off; }

    # WP uploads
    location ~ ^/wp-content/uploads/ {
        expires 30d;
        add_header Cache-Control "public";
    }
}
```

---

## Configuration Breakdown

### Server Block Basics

```nginx
listen 80;
listen [::]:80;
server_name www.viceroybali.com viceroybali.com 34.158.47.112;
```

- Listens on port 80 (HTTP) for IPv4 and IPv6
- Accepts requests for domain and IP address
- SSL termination handled by Load Balancer

### Document Root

```nginx
root /var/www/viceroybali/public_html;
index index.php index.html index.htm;
```

- WordPress files located in `/var/www/viceroybali/public_html`
- Prioritizes index.php (WordPress), then index.html

### Logging

```nginx
access_log /var/log/nginx/viceroybali_access.log;
error_log /var/log/nginx/viceroybali_error.log;
```

**View Logs:**
```bash
# Access log (requests)
tail -f /var/log/nginx/viceroybali_access.log

# Error log
tail -f /var/log/nginx/viceroybali_error.log

# Filter for errors only
grep "error" /var/log/nginx/viceroybali_error.log
```

### Security Headers

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
```

- **X-Frame-Options:** Prevents clickjacking attacks
- **X-Content-Type-Options:** Prevents MIME-type sniffing
- **X-XSS-Protection:** Enables XSS filter in browsers

### Gzip Compression

```nginx
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_types text/plain text/css text/xml application/json application/javascript ...;
```

- Compresses text-based files before sending to client
- Level 6 balances compression vs CPU usage
- Reduces bandwidth usage by ~60-70%

### Upload Limits

```nginx
client_max_body_size 64M;
```

- Allows file uploads up to 64 MB
- Important for media-heavy WordPress site
- Must match PHP settings

### WordPress Permalinks

```nginx
location / {
    try_files $uri $uri/ /index.php?$args;
}
```

- Handles WordPress pretty permalinks
- Falls back to index.php for dynamic routes

### PHP Processing

```nginx
location ~ \.php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
    fastcgi_read_timeout 300;
    fastcgi_send_timeout 300;
}
```

- Processes PHP files via PHP-FPM socket
- 300 second timeouts for long-running scripts
- Uses PHP 8.3 FPM

### Security Blocks

```nginx
location ~ /\.ht { deny all; }
location ~ /\.git { deny all; }
location = /wp-config.php { deny all; }
location = /xmlrpc.php { deny all; }
```

- Blocks access to sensitive files
- Prevents .htaccess exposure
- Blocks Git directory
- Prevents wp-config.php direct access
- Disables XML-RPC (common attack vector)

### Static File Caching

```nginx
location ~* \.(jpg|jpeg|png|gif|ico|css|js|pdf|webp|woff|woff2|ttf|svg|eot)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
    access_log off;
}
```

- Caches static files for 30 days
- Reduces server load
- Combined with Cloud CDN for global caching

### Health Check Endpoint

```nginx
location = /health {
    access_log off;
    return 200 "healthy\n";
    add_header Content-Type text/plain;
}
```

- Used by Google Cloud Load Balancer
- Returns 200 OK if server is running
- No logging for health checks

---

## Deployment Steps

### 1. Create Configuration File

```bash
ssh -i ~/.ssh/id_ed25519_gaia root@34.158.47.112

cat > /etc/nginx/sites-available/viceroybali << 'NGINXCONF'
[Full configuration from above]
NGINXCONF
```

### 2. Enable Site

```bash
# Create symlink
ln -sf /etc/nginx/sites-available/viceroybali /etc/nginx/sites-enabled/viceroybali

# Remove default site
rm -f /etc/nginx/sites-enabled/default

# Test configuration
nginx -t

# Reload Nginx
systemctl reload nginx
```

### 3. Verify Configuration

```bash
# Check syntax
nginx -t

# Expected output:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# Check Nginx status
systemctl status nginx

# Test HTTP response
curl -I http://localhost

# Test health endpoint
curl http://localhost/health
# Output: healthy
```

---

## PHP-FPM Integration

### PHP-FPM Configuration

**Socket:** `/var/run/php/php8.3-fpm.sock`

**PHP Settings:** `/etc/php/8.3/fpm/php.ini`

Key settings for WordPress:
```ini
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
max_execution_time = 300
max_input_time = 300
```

**Pool Configuration:** `/etc/php/8.3/fpm/pool.d/www.conf`

```ini
user = www-data
group = www-data
listen = /var/run/php/php8.3-fpm.sock
listen.owner = www-data
listen.group = www-data
pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
```

### Restart PHP-FPM

```bash
systemctl restart php8.3-fpm
systemctl status php8.3-fpm
```

---

## Performance Tuning

### Nginx Worker Processes

Edit `/etc/nginx/nginx.conf`:

```nginx
worker_processes auto;
worker_connections 1024;
```

- `auto` = number of CPU cores (2 for e2-standard-2)
- `worker_connections` = max connections per worker

### Fastcgi Cache (Optional)

For additional caching beyond WP Rocket:

```nginx
fastcgi_cache_path /var/cache/nginx levels=1:2 keys_zone=WORDPRESS:100m inactive=60m;
fastcgi_cache_key "$scheme$request_method$host$request_uri";
```

**Note:** WP Rocket handles caching, so this is optional.

---

## Common Operations

### Reload Configuration

```bash
# Test first
nginx -t

# If OK, reload
systemctl reload nginx
```

### Restart Nginx

```bash
systemctl restart nginx
```

### View Active Configuration

```bash
cat /etc/nginx/sites-enabled/viceroybali
```

### Check Error Logs

```bash
# Real-time
tail -f /var/log/nginx/viceroybali_error.log

# Last 50 lines
tail -50 /var/log/nginx/viceroybali_error.log

# Search for specific error
grep "PHP" /var/log/nginx/viceroybali_error.log
```

### Check Access Logs

```bash
# Real-time
tail -f /var/log/nginx/viceroybali_access.log

# Count requests
wc -l /var/log/nginx/viceroybali_access.log

# Find most accessed pages
awk '{print $7}' /var/log/nginx/viceroybali_access.log | sort | uniq -c | sort -rn | head -10
```

---

## Troubleshooting

### 502 Bad Gateway

**Cause:** PHP-FPM not running or socket issue

**Fix:**
```bash
systemctl status php8.3-fpm
systemctl restart php8.3-fpm
systemctl reload nginx
```

### 403 Forbidden

**Cause:** Permission issues

**Fix:**
```bash
chown -R www-data:www-data /var/www/viceroybali/public_html
chmod 755 /var/www/viceroybali/public_html
```

### 404 Not Found

**Cause:** Permalinks not working

**Fix:**
- Verify `try_files` directive in nginx config
- Check WordPress permalink settings

### Slow Response

**Check:**
```bash
# PHP-FPM slow log
tail -f /var/log/php8.3-fpm.log

# Nginx status
curl http://localhost/nginx_status
```

---

## Security Best Practices

✅ **Implemented:**
- Security headers (X-Frame-Options, X-Content-Type-Options, XSS Protection)
- Block sensitive files (.git, wp-config.php, .htaccess)
- Disable XML-RPC
- Limit upload size

⚠️ **Recommended:**
- Enable fail2ban for brute-force protection
- Rate limiting for wp-login.php
- ModSecurity WAF rules
- Regular security updates

---

## Integration with Cloud CDN

Nginx serves origin content to Google Cloud CDN:

1. **Request Flow:**
   - User → Load Balancer → Cloud CDN → Nginx → PHP-FPM

2. **Static Files:**
   - Cached by CDN (30-day expiry)
   - Nginx serves from disk on cache miss

3. **Dynamic Content:**
   - WP Rocket caches on server
   - Nginx serves cached HTML
   - PHP-FPM generates uncached pages

---

## Status

- [x] Nginx installed and configured
- [x] PHP-FPM integration working
- [x] Security headers enabled
- [x] Gzip compression active
- [x] Health check endpoint functional
- [x] Logging configured
- [x] WordPress permalinks working
- [x] Multi-site path-based routing configured

---

## Multi-Site Configuration (Staging)

As of February 3, 2026, Nginx is configured for path-based routing to support multiple staging sites.

### URL Routing

| Path | Document Root | Content |
|------|---------------|---------|
| `/` | Redirect | → `/viceroybali/` |
| `/viceroybali/` | `/var/www/viceroybali/public_html/` | WordPress staging |
| `/02production/` | `/var/www/02production/` | Placeholder |
| `/03production/` | `/var/www/03production/` | Placeholder |
| `/health` | - | Load balancer health check |

### Key Configuration Changes

The multi-site config uses `location ^~` with `alias` directives for path-based routing:

```nginx
# Viceroybali WordPress
location ^~ /viceroybali/ {
    alias /var/www/viceroybali/public_html/;
    # ... WordPress handling
}

# Production placeholders
location ^~ /02production/ {
    alias /var/www/02production/;
    # ...
}

location ^~ /03production/ {
    alias /var/www/03production/;
    # ...
}
```

For complete configuration, see [multisite_staging.md](../setup/multisite_staging.md).

---

**Configuration File:** `/etc/nginx/sites-available/viceroybali`
**Status:** Active and Running
**Performance:** Optimized for WordPress + Cloud CDN
