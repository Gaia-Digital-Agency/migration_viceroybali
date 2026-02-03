# Files and Folders Safe to Remove

**Project:** Viceroy Bali Migration
**Purpose:** Identify non-essential files to reclaim disk space
**Current Disk Usage:** 50 GB total, 29 GB used, 20 GB free

## Summary

| Category | Total Size | Safe to Remove |
|----------|------------|----------------|
| **Backup Archives** | ~2.2 GB | ✅ Yes |
| **Cache Directories** | ~120 MB | ✅ Yes |
| **Temporary Files** | ~177 MB | ✅ Yes |
| **Log Files** | ~23 MB | ⚠️ Partial |
| **Plugin Zip** | ~76 MB | ✅ Yes |
| **Duplicate Themes** | ~50 MB | ⚠️ After verification |
| **Hostinger-Specific** | ~10 MB | ✅ Yes |
| **Development Files** | ~1.5 MB | ✅ Yes |
| **Total Removable** | **~2.7 GB** | |

---

## 1. Backup Archives (HIGH PRIORITY - 2.2 GB)

### Location: /var/www/viceroybali/misc_backups/
**Size:** 2.2 GB
**Status:** ✅ Safe to remove

```bash
# Check contents
ls -lh /var/www/viceroybali/misc_backups/

# Remove (backup already in GCP bucket)
rm -rf /var/www/viceroybali/misc_backups/*
```

**Reason:**
- Hostinger auto-generated backups
- We have complete backup in gs://viceroybali_bucket/viceroybali_backup.tar.gz
- Database snapshot separately stored in /var/www/backups/

### Location: /var/www/viceroybali/public_html/wp-content/backups-dup-lite/
**Size:** ~500 MB (estimated)
**Status:** ✅ Safe to remove

```bash
# Remove Duplicator plugin backups
rm -rf /var/www/viceroybali/public_html/wp-content/backups-dup-lite/
```

**Reason:** Old backup plugin archives no longer needed.

### Location: /var/www/viceroybali/public_html/wp-content/updraft/
**Size:** ~300 MB (estimated)
**Status:** ✅ Safe to remove

```bash
# Remove UpdraftPlus backups
rm -rf /var/www/viceroybali/public_html/wp-content/updraft/*
```

**Reason:** Old UpdraftPlus backups. Keep the plugin active but remove old archives.

### Location: /var/www/viceroybali/wordpress-backups/
**Size:** Minimal
**Status:** ✅ Safe to remove

```bash
rm -rf /var/www/viceroybali/wordpress-backups/
```

## 2. Cache Directories (MEDIUM PRIORITY - 120 MB)

### Location: /var/www/viceroybali/lscache/
**Size:** 120 MB
**Status:** ✅ Safe to remove

```bash
# Remove LiteSpeed cache (incompatible with Nginx)
rm -rf /var/www/viceroybali/lscache/*
```

**Reason:** LiteSpeed cache is Hostinger-specific, doesn't work with Nginx.

### Location: /var/www/viceroybali/public_html/wp-content/cache/
**Size:** ~50 MB (estimated)
**Status:** ⚠️ Keep - Used by WP Rocket

```bash
# DO NOT remove - WP Rocket stores page cache here
# Only clear via WP Rocket admin panel if needed
```

**Reason:** WP Rocket is now active and uses this directory for page caching.

### Location: /var/www/viceroybali/public_html/wp-content/litespeed/
**Size:** ~20 MB (estimated)
**Status:** ✅ Safe to remove

```bash
# Remove LiteSpeed plugin cache
rm -rf /var/www/viceroybali/public_html/wp-content/litespeed/*
```

**Reason:** LiteSpeed Cache plugin has been **deactivated** (Feb 2, 2026). WP Rocket is now active instead.

## 3. Temporary Files (MEDIUM PRIORITY - 177 MB)

### Location: /var/www/viceroybali/tmp/
**Size:** 177 MB
**Status:** ⚠️ Partial removal

```bash
# Check what's in tmp
ls -lah /var/www/viceroybali/tmp/

# Remove old temporary files (older than 7 days)
find /var/www/viceroybali/tmp/ -type f -mtime +7 -delete
```

**Reason:** Temporary files that may be needed by running processes. Only remove old files.

### Location: /var/www/viceroybali/public_html/wp-content/upgrade/
**Size:** ~10 MB
**Status:** ✅ Safe to remove

```bash
rm -rf /var/www/viceroybali/public_html/wp-content/upgrade/*
```

**Reason:** Temporary WordPress update files.

### Location: /var/www/viceroybali/public_html/wp-content/upgrade-temp-backup/
**Size:** ~20 MB
**Status:** ✅ Safe to remove

```bash
rm -rf /var/www/viceroybali/public_html/wp-content/upgrade-temp-backup/*
```

**Reason:** Temporary backup files from plugin/theme updates.

## 4. Log Files (LOW PRIORITY - 23 MB)

### Location: /var/www/viceroybali/logs/
**Size:** 23 MB
**Status:** ⚠️ Archive then remove

```bash
# Archive old logs
tar -czf /var/www/backups/hostinger_logs_$(date +%Y%m%d).tar.gz /var/www/viceroybali/logs/

# Remove after archiving
rm -rf /var/www/viceroybali/logs/*
```

**Reason:** Hostinger logs may be useful for debugging but not needed for operation.

### Location: /var/www/viceroybali/public_html/wp-content/wflogs/
**Size:** ~5 MB
**Status:** ⚠️ Keep recent

```bash
# Remove Wordfence logs older than 30 days
find /var/www/viceroybali/public_html/wp-content/wflogs/ -type f -mtime +30 -delete
```

**Reason:** Wordfence security logs. Keep recent ones for security monitoring.

## 5. Plugin Archives (HIGH PRIORITY - 76 MB)

### Location: /var/www/viceroybali/public_html/wp-content/plugins.zip
**Size:** 76 MB
**Status:** ✅ Safe to remove

```bash
# Remove plugin backup archive
rm /var/www/viceroybali/public_html/wp-content/plugins.zip
```

**Reason:** Manual backup archive. Plugins already installed and backed up in main archive.

### Location: /var/www/viceroybali/public_html/wp-content/plugins/acfpro.zip
**Size:** ~2 MB
**Status:** ✅ Safe to remove

```bash
rm /var/www/viceroybali/public_html/wp-content/plugins/acfpro.zip
```

**Reason:** Plugin installation file. Plugin already installed.

---

## 6. Duplicate Themes (MEDIUM PRIORITY - ~50 MB)

### Themes to Review:
```
/var/www/viceroybali/public_html/wp-content/themes/
├── viceroybali          ← Active theme (KEEP)
├── viceroy-git          ← Old version?
├── viceroy-git-2        ← Old version?
├── viceroy-git-3        ← Old version?
├── viceroybali-git      ← Old version?
├── viceroyblock         ← Old version?
├── twentytwentyfive     ← Default WP theme (keep as fallback)
```

**Status:** ⚠️ Verify active theme first

```bash
# Check active theme
mysql -u viceroy_user -p viceroy_db_name -e "SELECT option_value FROM vb21_options WHERE option_name = 'template';"

# After verification, remove unused theme versions
# DO NOT remove active theme or default fallback theme (twentytwentyfive)
```

**Action Required:**
1. Verify which theme is active
2. Keep active theme + one default WP theme as fallback
3. Remove all other versions

**Estimated Space Savings:** 40-50 MB

## 7. Hostinger-Specific Files (LOW PRIORITY - ~15 MB)

### Location: Various
**Status:** ✅ Safe to remove

```bash
# Remove Hostinger control panel files
rm -rf /var/www/viceroybali/.cagefs/
rm -rf /var/www/viceroybali/.cpanel/
rm -rf /var/www/viceroybali/.cphorde/
rm -rf /var/www/viceroybali/.htpasswds/
rm -rf /var/www/viceroybali/.cl.selector/
rm -rf /var/www/viceroybali/.clwpos/
rm -rf /var/www/viceroybali/.cpaddons/

# Remove LiteSpeed-specific files
rm /var/www/viceroybali/public_html/.user.ini
rm /var/www/viceroybali/lscmData/*
```

**Reason:** Hostinger/cPanel/LiteSpeed specific files not needed on GCP.

## 8. Hidden Development Files (LOW PRIORITY - 1.5 MB)

### Location: /var/www/viceroybali/viceroybali.git/
**Size:** 1.5 MB
**Status:** ⚠️ Verify first

```bash
# Check if this is needed
ls -la /var/www/viceroybali/viceroybali.git/

# If not actively used for deployment
rm -rf /var/www/viceroybali/viceroybali.git/
```

### Location: /var/www/viceroybali/public_html/.git/
**Status:** ⚠️ Consider keeping

```bash
# DO NOT remove if using Git for version control
# Only remove if certain it's not needed
```

**Recommendation:** Keep if site uses Git deployment, otherwise remove.

### Location: /var/www/viceroybali/post-receive
**Status:** ⚠️ Git hook

**Recommendation:** Remove if not using Git deployment.

## 9. Misc Unnecessary Directories (LOW PRIORITY)

### Location: /var/www/viceroybali/public_ftp/
**Status:** ✅ Safe to remove

```bash
rm -rf /var/www/viceroybali/public_ftp/
```

**Reason:** Public FTP not needed on GCP.

---

### Location: /var/www/viceroybali/cache/
**Status:** ✅ Safe to remove

```bash
rm -rf /var/www/viceroybali/cache/
```

### Location: /var/www/viceroybali/public_html/.quarantine/
**Status:** ✅ Safe to remove

```bash
rm -rf /var/www/viceroybali/public_html/.quarantine/
```

**Reason:** Hostinger malware quarantine folder.

### Location: /var/www/viceroybali/public_html/.tmb/
**Status:** ✅ Safe to remove

```bash
rm -rf /var/www/viceroybali/public_html/.tmb/
```

**Reason:** File manager thumbnail cache.

## 10. Specific WordPress Files

### Location: /var/www/viceroybali/public_html/.htaccess*
**Status:** ✅ Remove or archive

```bash
# Archive .htaccess files (may contain useful rules)
cp /var/www/viceroybali/public_html/.htaccess /var/www/backups/.htaccess.hostinger

# Remove from public_html (Nginx doesn't use .htaccess)
rm /var/www/viceroybali/public_html/.htaccess*
```

**Reason:** Apache .htaccess files. Nginx uses different configuration.

### Location: /var/www/viceroybali/public_html/wp-content/advanced-cache.php
**Status:** ❌ DO NOT REMOVE - Used by WP Rocket

```bash
# DO NOT remove - WP Rocket uses this for page caching
# This file is now managed by WP Rocket (not LiteSpeed)
```

**Reason:** WP Rocket is now active and uses advanced-cache.php for page caching.

## Cleanup Script

Complete cleanup script to run all removals:

```bash
#!/bin/bash
# Viceroy Bali - Safe File Cleanup Script

# Backup archives (2.2 GB)
rm -rf /var/www/viceroybali/misc_backups/*
rm -rf /var/www/viceroybali/public_html/wp-content/backups-dup-lite/
rm -rf /var/www/viceroybali/public_html/wp-content/updraft/*
rm -rf /var/www/viceroybali/wordpress-backups/

# Cache directories (120 MB)
rm -rf /var/www/viceroybali/lscache/*
# DO NOT remove wp-content/cache/ - used by WP Rocket
rm -rf /var/www/viceroybali/public_html/wp-content/litespeed/*

# Temporary files (177 MB)
find /var/www/viceroybali/tmp/ -type f -mtime +7 -delete
rm -rf /var/www/viceroybali/public_html/wp-content/upgrade/*
rm -rf /var/www/viceroybali/public_html/wp-content/upgrade-temp-backup/*

# Plugin archives (76 MB)
rm /var/www/viceroybali/public_html/wp-content/plugins.zip
rm /var/www/viceroybali/public_html/wp-content/plugins/acfpro.zip

# Hostinger-specific files
rm -rf /var/www/viceroybali/.cagefs/
rm -rf /var/www/viceroybali/.cpanel/
rm -rf /var/www/viceroybali/.cphorde/
rm -rf /var/www/viceroybali/.htpasswds/
rm /var/www/viceroybali/public_html/.user.ini
rm -rf /var/www/viceroybali/lscmData/*

# Misc directories
rm -rf /var/www/viceroybali/public_ftp/
rm -rf /var/www/viceroybali/cache/
rm -rf /var/www/viceroybali/public_html/.quarantine/
rm -rf /var/www/viceroybali/public_html/.tmb/

# LiteSpeed files (DO NOT remove advanced-cache.php - used by WP Rocket)
rm /var/www/viceroybali/public_html/.htaccess*

echo "Cleanup complete!"
du -sh /var/www/viceroybali/
```

## Files to KEEP (DO NOT REMOVE)

✅ **Keep These:**
- `/var/www/viceroybali/public_html/` - Main WordPress installation
- `/var/www/viceroybali/public_html/wp-content/uploads/` - All uploaded media files
- `/var/www/viceroybali/public_html/wp-content/plugins/` - Active plugins (remove zip files only)
- `/var/www/viceroybali/public_html/wp-content/themes/viceroybali/` - Active theme
- `/var/www/viceroybali/public_html/wp-content/themes/twentytwentyfive/` - Default fallback theme
- `/var/www/backups/` - Database snapshot and important backups
- `/var/www/viceroybali/ssl/` - SSL certificate files (if any)

## Verification After Cleanup

```bash
# Check disk usage before
df -h /

# Run cleanup script
bash cleanup.sh

# Check disk usage after
df -h /

# Verify WordPress still works
curl -I http://34.158.47.112

# Check for any errors
tail -f /var/log/nginx/viceroybali_error.log
```

## Notes

1. **Always verify before deleting** - Check active theme/plugins first
2. **Backup strategy** - Main backup is in gs://viceroybali_bucket/
3. **Rollback plan** - Can restore from GCP bucket if needed
4. **Test after cleanup** - Ensure WordPress loads correctly
5. **No Hostinger access** - Do not go back to Hostinger to fetch files

**Estimated Total Space Reclaimed:** 2.7 GB
**New Free Space:** ~23 GB (from current 20 GB)
**Risk Level:** Low (all removals are safe with proper verification)
