# WordPress Staging Deployment Plan

**Project:** Viceroy Bali Migration
**Goal:** Deploy WordPress staging site matching https://www.viceroybali.com/
**Access:** Via IP address 34.158.47.112 (staging only, no DNS changes)

---

## Current Status

### Infrastructure ✅
- [x] GCP VM provisioned (e2-standard-2, Singapore)
- [x] Nginx + PHP 8.3 + MariaDB + Redis installed
- [x] Load Balancer + Cloud CDN configured
- [x] Firewall rules active
- [x] WordPress backup files extracted (17 GB in /var/www/viceroybali/public_html/)
- [x] Database snapshot ready (183 MB in /var/www/backups/)

### Pending Tasks
- [ ] Import database snapshot
- [ ] Update wp-config.php for new environment
- [ ] Fix file permissions
- [ ] Configure Redis cache
- [ ] Test WordPress functionality
- [ ] Verify 3rd party booking system integration

---

## Deployment Steps

### Step 1: Import Database (Priority: HIGH)

```bash
# Connect to server
ssh -i ~/.ssh/id_ed25519_gaia root@34.158.47.112

# Import database snapshot
mysql -u viceroy_user -p viceroy_db_name < /var/www/backups/viceroy_db_snapshot.sql

# Verify import
mysql -u viceroy_user -p viceroy_db_name -e "SHOW TABLES; SELECT COUNT(*) FROM vb21_posts;"
```

**Expected Result:**
- 107 tables imported
- Database populated with posts, pages, settings

---

### Step 2: Update wp-config.php (Priority: HIGH)

**Current Configuration:**
```php
define('DB_NAME', 'viceroybali_vb2021');  // OLD
```

**Required Changes:**
```php
// Database credentials
define('DB_NAME', 'viceroy_db_name');
define('DB_USER', 'viceroy_user');
define('DB_PASSWORD', 'strong_password');
define('DB_HOST', 'localhost');

// Add Redis cache configuration
define('WP_REDIS_HOST', '127.0.0.1');
define('WP_REDIS_PORT', 6379);
define('WP_CACHE_KEY_SALT', 'viceroybali_');

// Increase memory limits for media-heavy site
define('WP_MEMORY_LIMIT', '256M');
define('WP_MAX_MEMORY_LIMIT', '512M');

// Disable file editing in admin
define('DISALLOW_FILE_EDIT', true);

// Debug settings (staging only)
define('WP_DEBUG', false);
define('WP_DEBUG_LOG', true);
define('WP_DEBUG_DISPLAY', false);
```

**Execute:**
```bash
# Backup original wp-config.php
cp /var/www/viceroybali/public_html/wp-config.php /var/www/viceroybali/public_html/wp-config.php.original

# Update database credentials
sed -i "s/define('DB_NAME', 'viceroybali_vb2021');/define('DB_NAME', 'viceroy_db_name');/" /var/www/viceroybali/public_html/wp-config.php
```

---

### Step 3: Fix File Permissions (Priority: HIGH)

**Issue:** Files currently owned by `azlan:azlan` (Hostinger user)
**Required:** Files must be owned by `www-data:www-data` (Nginx user)

```bash
# Set ownership
chown -R www-data:www-data /var/www/viceroybali/public_html

# Set directory permissions (755)
find /var/www/viceroybali/public_html -type d -exec chmod 755 {} \;

# Set file permissions (644)
find /var/www/viceroybali/public_html -type f -exec chmod 644 {} \;

# Secure wp-config.php (640)
chmod 640 /var/www/viceroybali/public_html/wp-config.php

# Make uploads writable
chmod 775 /var/www/viceroybali/public_html/wp-content/uploads
chown -R www-data:www-data /var/www/viceroybali/public_html/wp-content/uploads

# Verify
ls -la /var/www/viceroybali/public_html/ | head -15
```

---

### Step 4: Clean Up Unnecessary Files (Priority: MEDIUM)

Remove files identified in [can_remove.md](can_remove.md):

```bash
# Remove backup archives
rm -rf /var/www/viceroybali/misc_backups/*
rm -rf /var/www/viceroybali/public_html/wp-content/backups-dup-lite/
rm -rf /var/www/viceroybali/public_html/wp-content/updraft/

# Remove cache directories
rm -rf /var/www/viceroybali/lscache/*
rm -rf /var/www/viceroybali/public_html/wp-content/cache/*

# Remove plugin zip file
rm /var/www/viceroybali/public_html/wp-content/plugins.zip

# Remove duplicate theme versions
# (keep only active theme - verify first)
```

**Space Savings:** ~3-4 GB

---

### Step 5: Update Site URLs in Database (Priority: HIGH)

**Important:** Update WordPress site URLs to use the staging IP address.

```bash
# Option 1: Using WP-CLI (recommended)
wp search-replace 'https://www.viceroybali.com' 'http://34.158.47.112' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid \
  --dry-run

# If dry-run looks good, remove --dry-run flag

# Option 2: Using MySQL (alternative)
mysql -u viceroy_user -p viceroy_db_name << EOF
UPDATE vb21_options SET option_value = 'http://34.158.47.112' WHERE option_name = 'siteurl';
UPDATE vb21_options SET option_value = 'http://34.158.47.112' WHERE option_name = 'home';
EOF
```

**Note:** We use HTTP for staging (no SSL yet). Production will use HTTPS.

---

### Step 6: Configure Redis Cache (Priority: MEDIUM)

```bash
# Install WP Redis plugin (if not already present)
# The object-cache.php is already in wp-content/

# Verify Redis is running
systemctl status redis

# Test Redis connection
redis-cli ping
# Should return: PONG

# Flush Redis cache
redis-cli FLUSHALL

# Monitor Redis (optional)
redis-cli MONITOR
```

---

### Step 7: Disable Hostinger-Specific Features (Priority: MEDIUM)

```bash
# Remove LiteSpeed cache (not compatible with Nginx)
rm /var/www/viceroybali/public_html/wp-content/advanced-cache.php
rm /var/www/viceroybali/public_html/.htaccess

# Disable LiteSpeed Cache plugin in database
mysql -u viceroy_user -p viceroy_db_name << EOF
UPDATE vb21_options SET option_value = '' WHERE option_name = 'active_plugins' AND option_value LIKE '%litespeed%';
EOF
```

**Note:** LiteSpeed Cache is incompatible with Nginx. Use WP Rocket instead.

---

### Step 8: Test WordPress Site (Priority: HIGH)

```bash
# Test basic WordPress load
curl -I http://34.158.47.112

# Test WordPress admin
curl -I http://34.158.47.112/wp-admin/

# Check PHP errors
tail -f /var/log/nginx/viceroybali_error.log

# Check PHP-FPM errors
tail -f /var/log/php8.3-fpm.log
```

**Manual Testing Checklist:**
- [ ] Homepage loads correctly
- [ ] Villa pages display properly
- [ ] Images load correctly
- [ ] Booking system integration works
- [ ] Contact forms functional
- [ ] Admin dashboard accessible
- [ ] Plugins active and working
- [ ] Theme displays correctly

---

### Step 9: Re-enable WordPress (Remove Hello World)

```bash
# Restore WordPress index.php as primary
mv /var/www/viceroybali/public_html/index.php.bak /var/www/viceroybali/public_html/index.php

# Remove or rename Hello World page
mv /var/www/viceroybali/public_html/index.html /var/www/viceroybali/public_html/index.html.bak

# Reload Nginx
systemctl reload nginx
```

---

### Step 10: Configure WordPress Caching (Priority: LOW)

**WP Rocket Configuration:**
- Enable page caching
- Enable file optimization (minify CSS/JS)
- Enable lazy loading for images
- Configure Redis as object cache
- Disable database optimization (already optimized)

**Note:** Cloud CDN handles static asset caching globally.

---

## WordPress Configuration Summary

### Active Plugins (Key)
- **WP Rocket** - Page caching & optimization
- **Wordfence** - Security & firewall
- **Yoast SEO Premium** - SEO optimization
- **WP Mail SMTP Pro** - Email delivery
- **WPForms Lite** - Contact forms
- **Advanced Custom Fields Pro** - Custom fields
- **UpdraftPlus** - Backup system
- **Polylang** - Multi-language support
- **Redirection** - URL management

### Active Theme
- **viceroybali** (custom theme)

### Database
- **Name:** viceroy_db_name
- **User:** viceroy_user
- **Tables:** 107
- **Prefix:** vb21_

---

## Critical Considerations

### 1. 3rd Party Booking System
- **Current:** Integrated external booking system
- **Action Required:** Verify booking integration works on staging
- **Test:** Check if booking widgets/iframes load correctly

### 2. Email Functionality
- **Plugin:** WP Mail SMTP Pro
- **Action Required:** Verify SMTP credentials still valid
- **Test:** Send test email from admin

### 3. SEO Protection
- **Important:** Staging site must be hidden from search engines
- **Action:** Add to wp-config.php:
  ```php
  define('WP_ENVIRONMENT_TYPE', 'staging');
  ```
- **Verify:** Check robots.txt blocks indexing

### 4. SSL Certificate
- **Status:** Google-managed SSL provisioning (for Load Balancer)
- **Staging:** Use HTTP only (no SSL on staging)
- **Production:** Will auto-activate once DNS points to Load Balancer

---

## Risk Mitigation

### Backup Strategy
✅ Original files in GCP bucket (gs://viceroybali_bucket/viceroybali_backup.tar.gz)
✅ Database snapshot in /var/www/backups/
✅ Live site on Hostinger untouched

### Rollback Plan
If issues occur:
1. Live site continues running on Hostinger (zero impact)
2. Can rebuild staging from backup files
3. Can re-import database from snapshot

---

## Success Criteria

Site is ready when:
- [  ] WordPress loads without errors
- [ ] All pages render correctly
- [ ] Images display properly
- [ ] Booking system integration functional
- [ ] Admin dashboard accessible
- [ ] No PHP/MySQL errors in logs
- [ ] Performance acceptable (TTFB < 500ms)

---

## Timeline Estimate

| Task | Priority | Status |
|------|----------|--------|
| Import database | HIGH | Pending |
| Update wp-config.php | HIGH | Pending |
| Fix permissions | HIGH | Pending |
| Update site URLs | HIGH | Pending |
| Test basic functionality | HIGH | Pending |
| Configure Redis | MEDIUM | Pending |
| Clean up files | MEDIUM | Pending |
| Disable LiteSpeed | MEDIUM | Pending |
| Full testing | HIGH | Pending |
| Documentation | LOW | Pending |

---

## Next Actions

**Immediate (Do Now):**
1. Import database snapshot
2. Update wp-config.php database credentials
3. Fix file permissions
4. Update site URLs to staging IP
5. Test WordPress loads

**Follow-up (After Basic Testing):**
1. Clean up unnecessary files
2. Configure Redis caching
3. Disable LiteSpeed features
4. Comprehensive testing
5. Document any issues

---

**Status:** Ready to execute
**Staging URL:** http://34.158.47.112 (after deployment)
**Production URL:** https://www.viceroybali.com/ (untouched)
