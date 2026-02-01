# Step 01: Project Exploration & Setup

**Date:** January 31, 2026
**Phase:** Discovery & Documentation

---

## Objective

Explore the viceroybali migration project folder and GCP server to understand current state before proceeding with WordPress deployment.

## Actions Completed

### 1. Local Project Analysis

**Command:**
```bash
ls -la /Users/rogerwoolie/Downloads/viceroybali/
```

Explored `/Users/rogerwoolie/Downloads/viceroybali/` folder:

| File | Size | Purpose |
|------|------|---------|
| `viceroy_db_snapshot.sql` | 183 MB | Full WordPress database dump (107 tables) |
| `gaia_viceroy_server_migration_plan_jan2026_v01.pdf` | 109 KB | Infrastructure migration plan |
| `viceroy_webStrat_2026_v01.pdf` | 57 KB | Web strategy document |
| `migration_plan.md` | 3.1 KB | Phase 1 migration roadmap |
| `gcp_info.md` | 291 B | GCP connection details |
| `databse_info.md` | 472 B | Database creation commands |
| `.gitignore` | 1.2 KB | Version control exclusions |

### 2. GCP Infrastructure Details

```
Project Name: GDA-COMPUTE
Project ID: gda-viceroy
Project Number: 292070531785
VM Instance: viceroy-bali
VM IP: 34.124.156.73
Region: asia-southeast1 (Singapore)
Storage Bucket: gs://viceroybali_bucket/
```

### 3. Server Stack Verification

**Command:**
```bash
ssh -i /Users/rogerwoolie/Container/dotfolders/.ssh/id_ed25519_gaia root@34.124.156.73 \
  "uname -a && \
   systemctl list-units --type=service --state=running | grep -E 'nginx|php|mysql|mariadb|redis' && \
   php -v | head -1"
```

**Output:**
```
Linux viceroy-bali.asia-southeast1-b.c.gda-viceroy.internal 6.14.0-1021-gcp #22~24.04.1-Ubuntu SMP Sat Nov 22 06:23:18 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
  mysql.service          loaded active running MySQL Community Server
  nginx.service          loaded active running A high performance web server and a reverse proxy server
  php8.3-fpm.service     loaded active running The PHP 8.3 FastCGI Process Manager
  redis-server.service   loaded active running Advanced key-value store
PHP 8.3.6 (cli) (built: Jan  7 2026 08:40:32) (NTS)
```

| Service | Status | Version |
|---------|--------|---------|
| Nginx | Running | Latest |
| PHP-FPM | Running | 8.3.6 |
| MySQL | Running | Community Server |
| Redis | Running | Latest |

### 4. Database Status

**Command:**
```bash
ssh -i /Users/rogerwoolie/Container/dotfolders/.ssh/id_ed25519_gaia root@34.124.156.73 \
  "mysql -u root -e 'SHOW DATABASES;' && \
   mysql -u root -e 'SHOW TABLES FROM viceroy_db_name;' | head -20"
```

**Output:**
```
Database
information_schema
mysql
performance_schema
sys
viceroy_db_name

Tables_in_viceroy_db_name
aP2021_users
vb21_actionscheduler_actions
vb21_actionscheduler_claims
vb21_actionscheduler_groups
vb21_actionscheduler_logs
vb21_cli_cookie_scan
vb21_cli_cookie_scan_categories
vb21_cli_cookie_scan_cookies
vb21_cli_cookie_scan_url
vb21_cli_scripts
vb21_commentmeta
vb21_comments
... (107 tables total)
```

Table prefix: `vb21_`

### 5. WordPress Files Audit

**Command:**
```bash
ssh -i /Users/rogerwoolie/Container/dotfolders/.ssh/id_ed25519_gaia root@34.124.156.73 \
  "ls -la /var/www/viceroybali/public_html/"
```

**Output:**
```
total 48
drwxr-xr-x 12 azlan azlan 4096 Jan 29 19:41 .
drwx--x--x  5 azlan azlan 4096 Dec 31 09:10 ..
drwxr-xr-x  2 azlan azlan 4096 Dec 10  2023 .quarantine
drwxrwxr-x  2 azlan azlan 4096 Dec 10  2023 .tmb
drwxr-xr-x  4 azlan azlan 4096 Mar 30  2024 .well-known
drwxr-xr-x  3 azlan azlan 4096 Jan 31  2025 360
drwxr-xr-x  2 azlan azlan 4096 Sep 19  2021 cgi-bin
drwxr-xr-x  2 azlan azlan 4096 Aug 29  2024 documents
drwxr-xr-x  8 azlan azlan 4096 Jun 23  2024 img
drwxr-xr-x  2 azlan azlan 4096 Apr 15  2025 video
drwxr-xr-x  9 azlan azlan 4096 Dec 31 04:03 wp-admin
drwxr-xr-x 11 azlan azlan 4096 Jan 29 20:42 wp-content
```

**Command (wp-content check):**
```bash
ssh -i /Users/rogerwoolie/Container/dotfolders/.ssh/id_ed25519_gaia root@34.124.156.73 \
  "ls -la /var/www/viceroybali/public_html/wp-content/"
```

**Output:**
```
total 60
drwxr-xr-x 11 azlan azlan  4096 Jan 29 20:42 .
drwxr-xr-x 12 azlan azlan  4096 Jan 29 19:41 ..
drwxr-xr-x 14 azlan azlan  4096 Dec 27 06:41 cache
drwxr-xr-x  4 azlan azlan 20480 Dec 27 06:13 languages
drwxr-xr-x 12 azlan azlan  4096 Jan 22 08:40 litespeed
drwxr-xr-x  3 azlan azlan  4096 Jan 27  2022 maintenance
drwxr-xr-x  2 azlan azlan  4096 Aug  1  2022 smush-webp
drwxr-xr-x  9 azlan azlan  4096 Dec 31 06:08 themes
drwxr-xr-x  2 azlan azlan  4096 Jan 29 19:41 upgrade
drwxr-xr-x  3 azlan azlan  4096 Jan 28 19:40 upgrade-temp-backup
drwxr-xr-x  7 azlan azlan  4096 Dec 31 16:00 uploads
```

Location: `/var/www/viceroybali/public_html/`

| Directory/File | Status |
|----------------|--------|
| `wp-admin/` | Present |
| `wp-content/` | Present |
| `wp-content/themes/` | Present |
| `wp-content/uploads/` | Present |
| `wp-content/plugins/` | Present (via database) |
| `wp-includes/` | **MISSING** |
| `index.php` | **MISSING** |
| `wp-config.php` | **MISSING** |
| Core PHP files | **MISSING** |

### 6. Nginx Configuration Status

**Command:**
```bash
ssh -i /Users/rogerwoolie/Container/dotfolders/.ssh/id_ed25519_gaia root@34.124.156.73 \
  "ls -la /etc/nginx/sites-available/ && \
   ls -la /etc/nginx/sites-enabled/"
```

**Output:**
```
=== Sites Available ===
total 12
drwxr-xr-x 2 root root 4096 Jan 29 13:46 .
drwxr-xr-x 8 root root 4096 Jan 29 13:46 ..
-rw-r--r-- 1 root root 2412 Nov 30  2023 default

=== Sites Enabled ===
total 8
drwxr-xr-x 2 root root 4096 Jan 29 13:46 .
drwxr-xr-x 8 root root 4096 Jan 29 13:46 ..
lrwxrwxrwx 1 root root   34 Jan 29 13:46 default -> /etc/nginx/sites-available/default
```

- Only default config exists at `/etc/nginx/sites-available/default`
- No Viceroy-specific server block configured
- SSL not configured

## Files Created

| File | Purpose |
|------|---------|
| `.claude/settings.json` | Project tracking and configuration |
| `README.md` | Project documentation |

## Current State Summary

| Component | Status |
|-----------|--------|
| GCP VM | Running |
| Server Stack | Configured |
| Database | Imported |
| WordPress Core | Incomplete |
| Nginx Config | Not configured |
| SSL | Not configured |
| Load Balancer | Not configured |
| CDN | Not configured |

## Next Steps

1. Download and install WordPress core files
2. Create wp-config.php with database credentials
3. Create Nginx server block
4. Set file permissions
5. Test site via IP
6. Setup Load Balancer + CDN
7. Configure firewall

---

**Status:** Complete
**Next Document:** step02.md
