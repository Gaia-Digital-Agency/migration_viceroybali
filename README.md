# Viceroy Bali - WordPress Migration Project

Migration of [www.viceroybali.com](https://www.viceroybali.com/) from Hostinger shared hosting to Google Cloud Platform.

## Current Status

| Phase | Status | Date |
|-------|--------|------|
| **Phase 1: Prep Work** | ✅ COMPLETE | Feb 2, 2026 |
| **Phase 1.5: Multi-Site Staging** | ✅ COMPLETE | Feb 3, 2026 |
| **Phase 2: DNS Cutover** | ⬜ PENDING | TBD |

## Multi-Site Staging URLs

| Site | URL | Purpose |
|------|-----|---------|
| **Viceroy Bali** | http://34.158.47.112/viceroybali/ | WordPress staging |
| **02 Production** | http://34.158.47.112/02production/ | Placeholder (future WordPress) |
| **03 Production** | http://34.158.47.112/03production/ | Placeholder (future WordPress) |

## Project Overview

| Field | Value |
|-------|-------|
| **Client** | Viceroy Bali Resort |
| **Domain** | viceroybali.com |
| **Current Host** | Hostinger (Shared Hosting) |
| **Target Host** | Google Cloud Platform (GCP) |
| **Migration Start** | February 2, 2026 |

## GCP Infrastructure

| Component | Configuration |
|-----------|---------------|
| **Project ID** | gda-viceroy |
| **Project Name** | GDA-COMPUTE |
| **VM Instance** | gda-ce01 |
| **VM IP (Static)** | 34.158.47.112 |
| **Load Balancer IP** | 34.49.188.147 |
| **Region** | asia-southeast1 (Singapore) |
| **Machine Type** | e2-standard-2 (2 vCPU / 8 GB RAM) |
| **Disk Size** | 50 GB (~20 GB free, 59% used) |
| **Storage Bucket** | gs://viceroybali_bucket/ |

## Folder Structure

```
/var/www/
├── viceroybali/
│   └── public_html/    → WordPress (http://34.158.47.112/viceroybali/)
├── 02production/
│   └── index.html      → Placeholder (http://34.158.47.112/02production/)
└── 03production/
    └── index.html      → Placeholder (http://34.158.47.112/03production/)
```

## Tech Stack

| Component | Version/Details |
|-----------|-----------------|
| **Web Server** | Nginx 1.24.0 |
| **PHP** | 8.3 |
| **Cache** | Redis + WP Rocket |
| **Database** | MySQL (MariaDB compatible) |
| **CMS** | WordPress 6.9 |
| **Theme** | viceroybali-git (Active) |

## Server Access

```bash
# SSH into the server
ssh -i ~/.ssh/id_ed25519_gaia root@34.158.47.112

# WordPress location
cd /var/www/viceroybali/public_html/
```

## Database

| Field | Value |
|-------|-------|
| **Database Name** | viceroy_db_name |
| **Database User** | viceroy_user |
| **Original DB Name** | viceroybali_vb2021 |
| **Table Prefix** | vb21_ |
| **Snapshot Size** | ~182 MB (107 tables) |

### Database Access

```bash
mysql -u viceroy_user -p viceroy_db_name
```

### Database Backup

```bash
# Latest backup location
/var/www/backups/viceroy_db_name_20260202_170446.sql

# Create new backup
wp db export /var/www/backups/viceroy_db_name_$(date +%Y%m%d_%H%M%S).sql \
  --path=/var/www/viceroybali/public_html/ --allow-root
```

## Migration Phases

### Phase 1: Infrastructure & Prep Work ✅ COMPLETE

- [x] GCP VM provisioned (e2-standard-2)
- [x] Configure Nginx + PHP 8.3 + Redis
- [x] Create database and import snapshot
- [x] Setup Global HTTP/HTTPS Load Balancer
- [x] Enable Google Cloud CDN
- [x] Configure firewall rules
- [x] Transfer WordPress backup files to GCP
- [x] Deploy WordPress files to staging
- [x] Configure wp-config.php
- [x] Enable Redis object cache
- [x] Enable WP Rocket page cache
- [x] Configure security headers
- [x] Verify GTM/Analytics working
- [x] Clean up disk space (~16 GB freed)
- [x] Remove unused themes
- [x] Verify all pages working

### Phase 1.5: Multi-Site Staging ✅ COMPLETE (Feb 3, 2026)

- [x] Reserve static IP (gda-ce01-static: 34.158.47.112)
- [x] Create 02production folder with placeholder
- [x] Create 03production folder with placeholder
- [x] Configure Nginx path-based routing
- [x] Update WordPress home/siteurl for /viceroybali/ path
- [x] Verify all staging sites accessible

### Phase 2: DNS Cutover ⬜ PENDING

- [ ] Run URL replacement (staging → production)
- [ ] Clear all caches
- [ ] Update DNS A records to 34.49.188.147
- [ ] SSL certificate activation (Google-managed)
- [ ] Verify live site functionality
- [ ] Enable HSTS header
- [ ] Verify ACF Pro license

## Architecture

```
                    +------------------+
                    |   Cloud DNS      |
                    +--------+---------+
                             |
                    +--------v---------+
                    | Global LB + CDN  |
                    | IP: 34.49.188.147|
                    | (SSL Termination)|
                    +--------+---------+
                             |
                    +--------v---------+
                    |      UMIG        |
                    | (Instance Group) |
                    +--------+---------+
                             |
                    +--------v---------+
                    |     gda-ce01     |
                    | IP: 34.158.47.112|
                    |    (Static)      |
                    |  Nginx+PHP+Redis |
                    +--------+---------+
                             |
              +--------------+--------------+
              |              |              |
     /viceroybali/    /02production/  /03production/
      WordPress        Placeholder     Placeholder
```

## Caching Stack

| Layer | Tool | Purpose |
|-------|------|---------|
| Object Cache | Redis | Database query caching |
| Page Cache | WP Rocket | Full HTML page caching |
| Static Assets | Google Cloud CDN | Global edge caching |

## Key Documentation

| File | Description |
|------|-------------|
| `Pre_migration.md` | Phase 1 & 2 checklist (main reference) |
| `gda-ce01.md` | Staging instance details & multi-site URLs |
| `setup/multisite_staging.md` | Multi-site staging setup guide |
| `features/nginx.md` | Nginx configuration (includes path-based routing) |
| `features/redis.md` | Redis configuration details |
| `features/architecture.md` | System architecture |
| `optimization/can_remove.md` | Disk cleanup guide |
| `.claude/progress.md` | Session progress notes |

## Commands Reference

```bash
# Connect to server
gcloud compute ssh gda-ce01 --zone=asia-southeast1-b --project=gda-viceroy

# Check all services
systemctl status nginx php8.3-fpm mysql redis-server

# Check staging sites
curl -sI http://34.158.47.112/viceroybali/
curl -sI http://34.158.47.112/02production/
curl -sI http://34.158.47.112/03production/

# Check disk usage
df -h /

# Restart services
systemctl restart nginx php8.3-fpm

# Clear Redis cache
redis-cli FLUSHALL

# Clear WordPress cache
wp cache flush --path=/var/www/viceroybali/public_html/ --allow-root

# View error logs
tail -f /var/log/nginx/multisite_error.log
```

## Important Notes

1. **Booking System:** Site integrates with Cloudbeds - verify widget loads after DNS cutover (domain whitelist required)

2. **SEO Protocol:** Using "Mirror & Verify" strategy:
   - Zero URL changes
   - Zero content changes
   - Private testing before DNS switch

3. **Backup Strategy:**
   - Database backup: `/var/www/backups/`
   - Full backup in GCP bucket: `gs://viceroybali_bucket/`

4. **SSL Certificate:**
   - Google-managed certificate pre-configured
   - Will auto-activate after DNS points to Load Balancer IP
   - Activation time: 15-60 minutes after DNS change

---

**Project managed by:** Gaia Digital Agency
**Last Updated:** February 3, 2026
