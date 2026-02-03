# Multi-Site Staging Environment Setup

**Instance:** gda-ce01
**External IP:** 34.158.47.112
**Date:** February 3, 2026

---

## Overview

Configure path-based routing for multiple staging sites:
- `http://34.158.47.112/viceroybali` → WordPress staging site
- `http://34.158.47.112/02production` → Placeholder (future WordPress)
- `http://34.158.47.112/03production` → Placeholder (future WordPress)

---

## Step 1: Make External IP Static

First, reserve the current ephemeral IP as a static IP in GCP.

```bash
# Check current IP configuration
gcloud compute instances describe gda-ce01 \
  --zone=asia-southeast1-b \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)'

# Reserve the current IP as static (promotes ephemeral to static)
gcloud compute addresses create gda-ce01-static \
  --region=asia-southeast1 \
  --addresses=34.158.47.112

# Verify the static IP reservation
gcloud compute addresses list --filter="region:asia-southeast1"
```

**Note:** If the IP is already reserved, you'll see a message. No changes needed.

---

## Step 2: Create Production Folders

```bash
# SSH to server
ssh -i ~/.ssh/id_ed25519_gaia root@34.158.47.112

# Create directories
mkdir -p /var/www/02production
mkdir -p /var/www/03production

# Create placeholder index.html for 02production
cat > /var/www/02production/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>02 Production</title>
    <style>
        body { font-family: Arial, sans-serif; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; background: #f5f5f5; }
        .container { text-align: center; padding: 40px; background: white; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        h1 { color: #333; }
        p { color: #666; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Hello World from 02 Production</h1>
        <p>This is a placeholder page. WordPress will be installed here.</p>
    </div>
</body>
</html>
EOF

# Create placeholder index.html for 03production
cat > /var/www/03production/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>03 Production</title>
    <style>
        body { font-family: Arial, sans-serif; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; background: #f5f5f5; }
        .container { text-align: center; padding: 40px; background: white; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        h1 { color: #333; }
        p { color: #666; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Hello World from 03 Production</h1>
        <p>This is a placeholder page. WordPress will be installed here.</p>
    </div>
</body>
</html>
EOF

# Set proper ownership
chown -R www-data:www-data /var/www/02production
chown -R www-data:www-data /var/www/03production

# Set permissions
chmod 755 /var/www/02production
chmod 755 /var/www/03production
chmod 644 /var/www/02production/index.html
chmod 644 /var/www/03production/index.html

# Verify
ls -la /var/www/
```

---

## Step 3: Update Nginx Configuration

Replace the existing Nginx configuration with path-based routing.

**Backup existing config:**
```bash
cp /etc/nginx/sites-available/viceroybali /etc/nginx/sites-available/viceroybali.backup
```

**Create new multi-site configuration:**
```bash
cat > /etc/nginx/sites-available/viceroybali << 'NGINXCONF'
# Multi-Site Staging Configuration
# Viceroy Bali + Production Sites

server {
    listen 80;
    listen [::]:80;
    server_name 34.158.47.112;

    # Logging
    access_log /var/log/nginx/multisite_access.log;
    error_log /var/log/nginx/multisite_error.log;

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

    # Root redirect to viceroybali
    location = / {
        return 301 /viceroybali/;
    }

    # ==========================================
    # VICEROYBALI - WordPress Site
    # ==========================================
    location ^~ /viceroybali/ {
        alias /var/www/viceroybali/public_html/;
        index index.php index.html;

        # WordPress permalinks
        try_files $uri $uri/ @viceroybali_rewrite;

        # Static file caching
        location ~* ^/viceroybali/.*\.(jpg|jpeg|png|gif|ico|css|js|pdf|webp|woff|woff2|ttf|svg|eot)$ {
            expires 30d;
            add_header Cache-Control "public, immutable";
            access_log off;
        }

        # PHP processing for viceroybali
        location ~ ^/viceroybali/(.+\.php)$ {
            alias /var/www/viceroybali/public_html/$1;
            fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $request_filename;
            include fastcgi_params;
            fastcgi_read_timeout 300;
            fastcgi_send_timeout 300;
        }
    }

    # WordPress rewrite for viceroybali
    location @viceroybali_rewrite {
        rewrite ^/viceroybali/(.*)$ /viceroybali/index.php?$args last;
    }

    # ==========================================
    # 02PRODUCTION - Placeholder Site
    # ==========================================
    location ^~ /02production/ {
        alias /var/www/02production/;
        index index.html index.php;
        try_files $uri $uri/ =404;

        # PHP processing for 02production (future WordPress)
        location ~ ^/02production/(.+\.php)$ {
            alias /var/www/02production/$1;
            fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $request_filename;
            include fastcgi_params;
            fastcgi_read_timeout 300;
        }
    }

    # ==========================================
    # 03PRODUCTION - Placeholder Site
    # ==========================================
    location ^~ /03production/ {
        alias /var/www/03production/;
        index index.html index.php;
        try_files $uri $uri/ =404;

        # PHP processing for 03production (future WordPress)
        location ~ ^/03production/(.+\.php)$ {
            alias /var/www/03production/$1;
            fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $request_filename;
            include fastcgi_params;
            fastcgi_read_timeout 300;
        }
    }

    # ==========================================
    # Security Blocks
    # ==========================================
    location ~ /\.ht { deny all; }
    location ~ /\.git { deny all; }

    # Health check endpoint for load balancer
    location = /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
NGINXCONF
```

**Test and reload:**
```bash
# Test configuration
nginx -t

# If OK, reload
systemctl reload nginx
```

---

## Step 4: Update WordPress URLs

Update WordPress to use the new `/viceroybali` path.

### Option A: Using WP-CLI (Recommended)

```bash
# Dry run first
wp search-replace 'http://34.158.47.112' 'http://34.158.47.112/viceroybali' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid \
  --dry-run

# If dry run looks good, execute
wp search-replace 'http://34.158.47.112' 'http://34.158.47.112/viceroybali' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid \
  --allow-root
```

### Option B: Direct MySQL Updates

```bash
mysql -u viceroy_user -p viceroy_db_name << 'EOF'
-- Update site URL
UPDATE vb21_options SET option_value = 'http://34.158.47.112/viceroybali' WHERE option_name = 'siteurl';
UPDATE vb21_options SET option_value = 'http://34.158.47.112/viceroybali' WHERE option_name = 'home';

-- Verify
SELECT option_name, option_value FROM vb21_options WHERE option_name IN ('siteurl', 'home');
EOF
```

---

## Step 5: Update wp-config.php

Add the subdirectory configuration to wp-config.php:

```bash
# Edit wp-config.php
nano /var/www/viceroybali/public_html/wp-config.php
```

Add these lines before `/* That's all, stop editing! */`:

```php
// Subdirectory configuration for staging
define('WP_HOME', 'http://34.158.47.112/viceroybali');
define('WP_SITEURL', 'http://34.158.47.112/viceroybali');

// Fix for running WordPress from subdirectory
$_SERVER['REQUEST_URI'] = str_replace('/viceroybali', '', $_SERVER['REQUEST_URI']);
```

**Note:** The REQUEST_URI fix may be needed for some WordPress configurations.

---

## Step 6: Clear Caches

```bash
# Clear WordPress cache
wp cache flush --path=/var/www/viceroybali/public_html/ --allow-root

# Clear Redis cache
redis-cli FLUSHALL

# Restart PHP-FPM
systemctl restart php8.3-fpm

# Reload Nginx
systemctl reload nginx
```

---

## Step 7: Test All Sites

```bash
# Test root redirect
curl -I http://34.158.47.112/

# Test viceroybali
curl -I http://34.158.47.112/viceroybali/

# Test 02production
curl -I http://34.158.47.112/02production/

# Test 03production
curl -I http://34.158.47.112/03production/

# Test health endpoint
curl http://34.158.47.112/health
```

---

## Final Structure

```
/var/www/
├── viceroybali/
│   └── public_html/    → WordPress (http://34.158.47.112/viceroybali/)
├── 02production/
│   └── index.html      → Placeholder (http://34.158.47.112/02production/)
└── 03production/
    └── index.html      → Placeholder (http://34.158.47.112/03production/)
```

---

## URL Summary

| Path | Location | Content |
|------|----------|---------|
| `/` | Redirect | → `/viceroybali/` |
| `/viceroybali/` | `/var/www/viceroybali/public_html/` | WordPress staging |
| `/02production/` | `/var/www/02production/` | Placeholder |
| `/03production/` | `/var/www/03production/` | Placeholder |
| `/health` | - | Load balancer health check |

---

## Rollback

If issues occur:

```bash
# Restore original Nginx config
cp /etc/nginx/sites-available/viceroybali.backup /etc/nginx/sites-available/viceroybali
nginx -t && systemctl reload nginx

# Restore WordPress URLs
mysql -u viceroy_user -p viceroy_db_name << 'EOF'
UPDATE vb21_options SET option_value = 'http://34.158.47.112' WHERE option_name = 'siteurl';
UPDATE vb21_options SET option_value = 'http://34.158.47.112' WHERE option_name = 'home';
EOF

# Remove wp-config.php changes if added
# Clear caches
redis-cli FLUSHALL
systemctl restart php8.3-fpm
```

---

## Notes

- The Load Balancer IP (34.49.188.147) remains unchanged for production
- Domain viceroybali.com continues to point to production
- This staging setup is IP-only access for testing
- Future: Can convert 02production/03production to full WordPress installs
