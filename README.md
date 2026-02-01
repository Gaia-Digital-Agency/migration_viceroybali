# Viceroy Bali - WordPress Migration Project

Migration of [www.viceroybali.com](https://www.viceroybali.com/) from Hostinger shared hosting to Google Cloud Platform.

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
| **VM Instance** | viceroy-bali |
| **VM IP** | 34.142.200.251 |
| **Load Balancer IP** | 34.49.188.147 |
| **Region** | asia-southeast1 (Singapore) |
| **Machine Type** | e2-standard-2 (2 vCPU / 8 GB RAM) |
| **Disk Size** | 50 GB (20 GB free, 60% used) |
| **Storage Bucket** | gs://viceroybali_bucket/ |

## Tech Stack

- **Web Server:** Nginx
- **PHP:** 8.3
- **Cache:** Redis
- **Database:** MariaDB 10.11.15
- **CMS:** WordPress

## Server Access

```bash
# SSH into the server
ssh -i ~/.ssh/id_ed25519_gaia root@34.142.200.251

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
| **Snapshot Size** | ~183 MB (107 tables) |

### Database Access

```bash
mysql -u viceroy_user -p viceroy_db_name
```

### Database Import

```bash
# Database snapshot location
/var/www/backups/viceroy_db_snapshot.sql

# Import command
mysql -u viceroy_user -p viceroy_db_name < /var/www/backups/viceroy_db_snapshot.sql
```

## Migration Phases

### Phase 1: Infrastructure Upgrade (Current)

**Objective:** Move to dedicated cloud environment for improved speed while preserving SEO.

- [x] GCP VM provisioned (e2-standard-2)
- [x] Configure Nginx + PHP 8.3 + Redis
- [x] Create database and import snapshot
- [x] Setup Global HTTP/HTTPS Load Balancer
- [x] Enable Google Cloud CDN
- [x] Configure firewall rules
- [x] Transfer WordPress backup files to GCP
- [x] Database snapshot stored in /var/www/backups/
- [ ] Deploy WordPress files to staging
- [ ] Configure wp-config.php
- [ ] SSL certificate (Google-managed) - Provisioning
- [ ] Private testing via hosts file
- [ ] DNS cutover (zero downtime)

## Architecture

```
                    +------------------+
                    |   Cloud DNS      |
                    +--------+---------+
                             |
                    +--------v---------+
                    | Global LB + CDN  |
                    | (SSL Termination)|
                    +--------+---------+
                             |
                    +--------v---------+
                    |      UMIG        |
                    | (Instance Group) |
                    +--------+---------+
                             |
                    +--------v---------+
                    |   viceroy-bali   |
                    |  e2-standard-2   |
                    |  Nginx+PHP+Redis |
                    +--------+---------+
                             |
                    +--------v---------+
                    |    MariaDB       |
                    +------------------+
```

## WordPress Plugins (Key)

| Plugin | Purpose |
|--------|---------|
| WP Rocket | Caching & Performance |
| LiteSpeed | Server-side caching |
| Yoast SEO | SEO optimization |
| Rank Math | SEO management |
| Wordfence | Security & Firewall |
| WP Mail SMTP | Email handling |
| WPForms | Form builder |

## Important Notes

1. **3rd Party Booking System:** The site integrates with an external booking system - must verify functionality after migration.

2. **SEO Protocol:** Using "Mirror & Verify" strategy:
   - Zero URL changes
   - Zero content changes
   - Private testing before DNS switch

3. **Backup Schedule:**
   - Weekly automated snapshots (Sunday 3:00 AM Bali time)
   - 4-week retention policy
   - "Gold Master" snapshot after verified migration

## Files in This Repository

| File | Description |
|------|-------------|
| `viceroy_db_snapshot.sql` | Full database dump (183 MB) |
| `migration_plan.md` | Phase 1 migration documentation |
| `gcp_info.md` | GCP connection details |
| `databse_info.md` | Database creation commands |
| `*.pdf` | Strategy and planning documents |

## Commands Reference

```bash
# Connect to server
ssh -i ~/.ssh/id_ed25519_gaia root@34.142.200.251

# Check Nginx status
systemctl status nginx

# Check PHP-FPM status
systemctl status php8.3-fpm

# Check Redis status
systemctl status redis

# Restart services
systemctl restart nginx php8.3-fpm redis

# Check WordPress files
ls -la /var/www/viceroy/

# View Nginx config
cat /etc/nginx/sites-available/viceroy

# Test Nginx config
nginx -t

# View error logs
tail -f /var/log/nginx/error.log
```

---

**Project managed by:** Gaia Digital Agency
