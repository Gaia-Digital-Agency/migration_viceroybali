# Step 03: Server Configuration & Infrastructure Deployment

**Date:** January 31, 2026
**Phase:** Infrastructure Deployment
**Status:** Complete

---

## Objective

Configure web server, setup GCP Global Load Balancer with CDN, and configure firewall rules for production-ready WordPress hosting.

---

## Components Deployed

| Component | Status | Documentation |
|-----------|--------|---------------|
| **Nginx Web Server** | ✅ Active | [nginx.md](nginx.md) |
| **Load Balancer** | ✅ Active | [loadbalancer.md](loadbalancer.md) |
| **Cloud CDN** | ✅ Enabled | [cdn.md](cdn.md) |
| **Firewall Rules** | ✅ Active | [firewall.md](firewall.md) |
| **SSL Certificate** | ⏳ Provisioning | [loadbalancer.md](loadbalancer.md) |

---

## Quick Reference

### Server Access

```bash
ssh -i ~/.ssh/id_ed25519_gaia root@34.142.200.251
```

### Important IP Addresses

| Resource | IP Address |
|----------|------------|
| **VM Instance** | 34.142.200.251 |
| **Load Balancer** | 34.49.188.147 |

### Service Status

```bash
# Check all services
systemctl status nginx
systemctl status php8.3-fpm
systemctl status redis
systemctl status mariadb
```

---

## Configuration Summary

### 1. Nginx Configuration

**Status:** ✅ Configured and Running

**Key Features:**
- WordPress-optimized configuration
- PHP 8.3-FPM integration
- Static file caching (30-day expiry)
- Security headers enabled
- Gzip compression active
- Health check endpoint (/health)

**Full Documentation:** [nginx.md](nginx.md)

**Quick Test:**
```bash
curl -I http://34.142.200.251/health
# Expected: HTTP/1.1 200 OK
```

---

### 2. File Permissions

**Status:** ✅ Applied

```bash
# Ownership
chown -R www-data:www-data /var/www/viceroybali/public_html

# Directory permissions: 755
find /var/www/viceroybali/public_html -type d -exec chmod 755 {} \;

# File permissions: 644
find /var/www/viceroybali/public_html -type f -exec chmod 644 {} \;

# wp-config.php: 640 (restricted)
chmod 640 /var/www/viceroybali/public_html/wp-config.php
```

**Verification:**
```bash
ls -la /var/www/viceroybali/public_html/ | head -15
```

---

### 3. Global Load Balancer

**Status:** ✅ Deployed

**Components:**
- Health Check (viceroybali-health-check)
- Unmanaged Instance Group (viceroybali-umig)
- Backend Service with CDN (viceroybali-backend)
- URL Maps (HTTP redirect + HTTPS)
- SSL Certificate (Google-managed, provisioning)
- Global IP Address: **34.49.188.147**

**Full Documentation:** [loadbalancer.md](loadbalancer.md)

**Verification:**
```bash
gcloud compute backend-services get-health viceroybali-backend --global
# Expected: HEALTHY
```

---

### 4. Cloud CDN

**Status:** ✅ Enabled

**Configuration:**
- Cache Mode: CACHE_ALL_STATIC
- Default TTL: 3600 seconds (1 hour)
- Edge Locations: Global (200+ worldwide)

**Cached Content:**
- Images (.jpg, .png, .gif, .webp, .svg)
- Stylesheets (.css)
- Scripts (.js)
- Fonts (.woff, .woff2, .ttf)
- Documents (.pdf)

**Full Documentation:** [cdn.md](cdn.md)

---

### 5. Firewall Rules

**Status:** ✅ Active

**Rules Created:**

| Rule Name | Ports | Source | Purpose |
|-----------|-------|--------|---------|
| viceroybali-allow-lb-health | 80 | 130.211.0.0/22<br>35.191.0.0/16 | Health checks |
| viceroybali-allow-http-https | 80, 443 | 0.0.0.0/0 | Public web traffic |

**Network Tags Applied:** `http-server`, `https-server`

**Full Documentation:** [firewall.md](firewall.md)

**Verification:**
```bash
gcloud compute firewall-rules list --filter="name~viceroybali"
```

---

## Architecture Overview

```
                         Internet
                             |
                    +--------v--------+
                    |  Global LB IP   |
                    |  34.49.188.147  |
                    +--------+--------+
                             |
              +--------------+--------------+
              |                             |
    +---------v---------+         +---------v---------+
    |   HTTP (Port 80)  |         |  HTTPS (Port 443) |
    |   Redirect Rule   |         |    SSL Termination|
    +---------+---------+         +---------+---------+
              |                             |
              +-------------+---------------+
                            |
                   +--------v--------+
                   |    Cloud CDN    |
                   | (Static Cache)  |
                   +--------+--------+
                            |
                   +--------v--------+
                   |   Backend Svc   |
                   | viceroybali-umig|
                   +--------+--------+
                            |
                   +--------v--------+
                   |  viceroy-bali   |
                   |  34.142.200.251 |
                   |  Nginx + PHP    |
                   +--------+--------+
                            |
                   +--------v--------+
                   |     MariaDB     |
                   +------------------+
```

---

## Testing Instructions

### Option 1: Direct VM Access (Current)

```bash
# Test via VM IP
curl -I http://34.142.200.251
```

**Note:** Currently showing "Hello World" test page. WordPress deployment pending.

### Option 2: Via Load Balancer (Production)

```bash
# Test via Load Balancer IP
curl -I http://34.49.188.147
```

### Option 3: Local hosts file (Pre-DNS Testing)

Add to `/etc/hosts` on your local machine:
```
34.49.188.147 www.viceroybali.com viceroybali.com
```

Then browse to: `http://www.viceroybali.com`

**Remove this entry before DNS cutover!**

---

## SSL Certificate Status

**Type:** Google-managed SSL certificate
**Domains:** www.viceroybali.com, viceroybali.com
**Status:** PROVISIONING

**Check Status:**
```bash
gcloud compute ssl-certificates describe viceroybali-ssl-cert \
  --global \
  --format="get(managed.status)"
```

**Activation:**
- Certificate will auto-activate once DNS points to 34.49.188.147
- Provisioning typically takes 15-60 minutes after DNS update
- No manual certificate management required

---

## Resource Summary

### GCP Resources Created

| Resource Type | Name | Configuration |
|--------------|------|---------------|
| VM Instance | viceroy-bali | e2-standard-2, Singapore |
| Health Check | viceroybali-health-check | HTTP, /health endpoint |
| Instance Group | viceroybali-umig | Unmanaged, 1 instance |
| Backend Service | viceroybali-backend | HTTP, CDN enabled |
| URL Map (HTTPS) | viceroybali-url-map | Routes to backend |
| URL Map (HTTP) | viceroybali-http-redirect | Redirects to HTTPS |
| SSL Certificate | viceroybali-ssl-cert | Google-managed |
| HTTPS Proxy | viceroybali-https-proxy | SSL termination |
| HTTP Proxy | viceroybali-http-proxy | HTTP redirect |
| Global IP | viceroybali-ip | 34.49.188.147 |
| Forwarding Rule (HTTPS) | viceroybali-https-rule | Port 443 |
| Forwarding Rule (HTTP) | viceroybali-http-rule | Port 80 |
| Firewall Rule (Health) | viceroybali-allow-lb-health | LB health checks |
| Firewall Rule (Web) | viceroybali-allow-http-https | Public traffic |

---

## Next Steps

1. **Deploy WordPress** - See [plan.md](plan.md) for deployment steps
2. **Import Database** - See [databse_info.md](databse_info.md)
3. **Configure wp-config.php** - Update database credentials
4. **Test Staging Site** - Verify functionality via IP address
5. **DNS Cutover** - Point domain to 34.49.188.147 (when ready)

---

## Important Notes

### Infrastructure Status
✅ **Completed:**
- VM provisioned and configured
- Nginx + PHP 8.3 + Redis + MariaDB installed
- Load Balancer with Cloud CDN deployed
- Firewall rules configured
- Backup files transferred and extracted

⏳ **Pending:**
- WordPress database import
- Site configuration and testing
- SSL certificate activation (awaiting DNS)
- DNS cutover to production

### Security
- All sensitive files blocked (.git, wp-config.php, etc.)
- Security headers enabled
- XML-RPC disabled
- Firewall rules restrict access appropriately
- SSL certificate will auto-provision after DNS update

### Performance
- Cloud CDN enabled for global static asset delivery
- Nginx configured for optimal WordPress performance
- Gzip compression active
- Static file caching (30-day expiry)
- Redis cache ready for object caching

### Monitoring
- Health checks every 10 seconds
- Nginx access and error logs active
- Load Balancer metrics available in GCP Console

---

## Documentation Links

- **Nginx Configuration:** [nginx.md](nginx.md)
- **Load Balancer Setup:** [loadbalancer.md](loadbalancer.md)
- **Cloud CDN Configuration:** [cdn.md](cdn.md)
- **Firewall Rules:** [firewall.md](firewall.md)
- **Database Setup:** [databse_info.md](databse_info.md)
- **Deployment Plan:** [plan.md](plan.md)
- **Files to Remove:** [can_remove.md](can_remove.md)

---

**Status:** Infrastructure deployment complete
**Current State:** Ready for WordPress deployment
**Live Production Site:** https://www.viceroybali.com/ (untouched on Hostinger)
**Staging Site:** http://34.142.200.251 (Hello World test page)
