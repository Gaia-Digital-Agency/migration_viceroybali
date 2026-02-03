# Optimization & Issues Report

**Project:** Viceroy Bali Staging
**Date:** February 1, 2026
**Server:** 34.158.47.112

## Critical Issues

### 1. Book Now Functionality Not Working ⚠️

**Issue:** Hardcoded production URLs in database content preventing proper staging site function.

**Details:**
- Book Now buttons point to `/en/reservation/` page
- Page loads but contains hardcoded URLs: `https://www.viceroybali.com`
- Images, CSS, and navigation links reference production domain
- Booking widget (Cloudbeds) may not load due to mixed content

**Root Cause:**
- WordPress stores some content with absolute URLs in database
- Site URLs updated in `vb21_options` table but not in post content/meta
- Need full database search-replace for all URL instances

**Solution:**
```bash
# Use WP-CLI for safe search-replace (handles serialized data)
wp search-replace 'https://www.viceroybali.com' 'http://34.158.47.112' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid \
  --all-tables \
  --report-changed-only

# Or use Better Search Replace plugin (GUI method)
# Install and run from WordPress admin
```

**Risk:** LOW - WP-CLI handles serialized data properly

**Impact:** HIGH - Booking system non-functional on staging

### 2. Logs Configuration ✅ OPTIMIZED

**Status:** Already properly configured

**Current Setup:**
- Access log: 37 KB (232 requests logged)
- Error log: 6.9 KB (minimal errors)
- Log rotation: Daily, 14-day retention, compressed
- Old logs bloat issue: RESOLVED (logs moved to /var/www/removed_files/)

**Configuration:** `/etc/logrotate.d/nginx`
```
daily
rotate 14
compress
delaycompress
```

**Recommendation:** No changes needed - already optimized

## Optimization Opportunities

### 1. PHP-FPM Pool Tuning ⚡ MEDIUM PRIORITY

**Current Settings:**
```
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

**Issue:** Too conservative for production workload

**Recommended Settings:**
```bash
# For e2-standard-2 (2 vCPU, 8GB RAM)
pm = dynamic
pm.max_children = 50        # Up from 5
pm.start_servers = 10       # Up from 2
pm.min_spare_servers = 5    # Up from 1
pm.max_spare_servers = 20   # Up from 3
pm.max_requests = 500       # Add this
```

**How to Apply:**
```bash
# Edit PHP-FPM pool config
nano /etc/php/8.3/fpm/pool.d/www.conf

# Update the pm.* values
# Then restart PHP-FPM
systemctl restart php8.3-fpm
```

**Benefits:**
- Handle more concurrent requests
- Faster response under load
- Better resource utilization

**Risk:** LOW - Can revert if memory issues occur

### 2. PHP OPcache Tuning ⚡ LOW PRIORITY

**Current Settings:**
```
opcache.enable = On
opcache.memory_consumption = 128MB
```

**Recommended:**
```ini
# Add to /etc/php/8.3/fpm/php.ini
opcache.memory_consumption=256        # Up from 128
opcache.interned_strings_buffer=16    # Add
opcache.max_accelerated_files=10000   # Add
opcache.validate_timestamps=0         # Production only
opcache.save_comments=0               # Production only
opcache.fast_shutdown=1               # Add
```

**How to Apply:**
```bash
nano /etc/php/8.3/fpm/php.ini
# Add settings under [opcache] section
systemctl restart php8.3-fpm
```

**Benefits:**
- Faster PHP execution
- Lower CPU usage
- Better caching of WordPress core

**Risk:** LOW

### 3. Redis Object Cache Integration ⚡ HIGH PRIORITY

**Current Status:**
- Redis installed and running ✅
- wp-config.php configured with Redis settings ✅
- **Redis NOT being used** (0 hits/misses)

**Issue:** Object cache drop-in file missing

**Solution:**
```bash
# Install Redis object cache plugin
wp plugin install redis-cache --activate \
  --path=/var/www/viceroybali/public_html/

# Enable object cache
wp redis enable \
  --path=/var/www/viceroybali/public_html/

# Verify
wp redis status \
  --path=/var/www/viceroybali/public_html/
```

**OR manually:**
```bash
# Download object cache drop-in
cd /var/www/viceroybali/public_html/wp-content/
curl -O https://raw.githubusercontent.com/rhubarbgroup/redis-cache/master/includes/object-cache.php
chown www-data:www-data object-cache.php
```

**Benefits:**
- 50-80% faster page loads
- Reduced database queries
- Better performance under traffic spikes

**Risk:** LOW - Can disable if issues occur

### 4. Disable Unnecessary Plugins ⚡ MEDIUM PRIORITY

**Current Status:** 44 plugin directories installed

**Recommendations:**

**Disable on Staging:**
- **LiteSpeed Cache** - Incompatible with Nginx (Apache-only)
- **Wordfence** - Heavy resource usage on staging
- **Query Monitor** - Development tool, not needed on staging

**Keep Active:**
- WP Rocket - Page caching
- Yoast SEO Premium - SEO management
- WP Mail SMTP Pro - Email functionality
- Advanced Custom Fields Pro - Custom fields
- Polylang - Multi-language

**How to Disable:**
```bash
wp plugin deactivate litespeed-cache wordfence query-monitor \
  --path=/var/www/viceroybali/public_html/
```

**Benefits:**
- Lower memory usage
- Faster admin dashboard
- Fewer potential conflicts

### 5. Nginx FastCGI Cache (Optional) ⚡ LOW PRIORITY

**Status:** Not currently implemented

**Note:** WP Rocket already provides page caching, so this is optional.

**If needed:**
```nginx
# Add to nginx config
fastcgi_cache_path /var/cache/nginx levels=1:2 keys_zone=WORDPRESS:100m inactive=60m;
fastcgi_cache_key "$scheme$request_method$host$request_uri";

# In location ~ \.php$ block
fastcgi_cache WORDPRESS;
fastcgi_cache_valid 200 60m;
fastcgi_cache_bypass $skip_cache;
fastcgi_no_cache $skip_cache;
```

**Recommendation:** Skip for now, WP Rocket sufficient

### 6. Database Optimization ⚡ LOW PRIORITY

**Current:** 107 tables, 10,220 posts

**Recommendations:**
```bash
# Clean up post revisions (can grow large)
wp post delete $(wp post list --post_type='revision' --format=ids) \
  --path=/var/www/viceroybali/public_html/

# Clean up transients
wp transient delete --all \
  --path=/var/www/viceroybali/public_html/

# Optimize all tables
wp db optimize \
  --path=/var/www/viceroybali/public_html/
```

**Benefits:**
- Smaller database size
- Faster queries
- Reduced backup size

**Risk:** VERY LOW

### 7. Image Optimization (Already Done) ✅

**Status:** Site uses WebP format and lazy loading

**Observed:**
- Images in `.webp` format
- Lazy loading implemented
- CDN caching configured (30-day expiry)

**No action needed**

## Architecture Optimizations (Already Implemented) ✅

### What's Already Optimized:

1. **Cloud CDN** ✅
   - Global edge caching enabled
   - 1-hour TTL for static assets
   - CACHE_ALL_STATIC mode

2. **Nginx Configuration** ✅
   - Gzip compression (level 6)
   - Static file caching (30 days)
   - Security headers enabled
   - 64 MB upload limit

3. **PHP 8.3** ✅
   - Latest stable version
   - OPcache enabled
   - Proper memory limits (256M)

4. **MariaDB** ✅
   - Modern database engine
   - Properly configured

5. **File Structure** ✅
   - Proper permissions (www-data)
   - Clean directory structure
   - 3.1 GB unnecessary files removed

## Implementation Priority

### Do Immediately (Critical):
1. **Fix Book Now URLs** - Run WP-CLI search-replace
2. **Enable Redis Object Cache** - Install plugin/drop-in

### Do Soon (High Impact):
3. **Tune PHP-FPM Pool** - Update pm.* settings
4. **Disable Unnecessary Plugins** - LiteSpeed, Wordfence on staging

### Do When Time Permits (Low Impact):
5. **Optimize Database** - Clean revisions and transients
6. **Tune PHP OPcache** - Increase memory and settings
7. **Consider FastCGI Cache** - Only if WP Rocket insufficient

## Performance Metrics (Estimated)

**Current Performance:**
- TTFB: ~500ms (estimated)
- Page Load: 2-3 seconds
- Database Queries: 50-100 per page

**After Optimizations:**
- TTFB: ~200ms (60% improvement)
- Page Load: 1-1.5 seconds (50% improvement)
- Database Queries: 10-20 per page (80% reduction with Redis)

## Safety Notes

### Backups Before Changes:
```bash
# Backup database
wp db export /var/www/backups/before_optimization.sql \
  --path=/var/www/viceroybali/public_html/

# Backup wp-config.php
cp /var/www/viceroybali/public_html/wp-config.php \
   /var/www/backups/wp-config.php.backup
```

### Monitoring After Changes:
```bash
# Watch error logs
tail -f /var/log/nginx/viceroybali_error.log

# Monitor PHP-FPM
systemctl status php8.3-fpm

# Check Redis
redis-cli info stats
```

## Commands Summary

### 1. Fix Book Now (CRITICAL)
```bash
wp search-replace 'https://www.viceroybali.com' 'http://34.158.47.112' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid \
  --all-tables
```

### 2. Enable Redis
```bash
wp plugin install redis-cache --activate \
  --path=/var/www/viceroybali/public_html/
wp redis enable --path=/var/www/viceroybali/public_html/
```

### 3. Optimize PHP-FPM
```bash
# Edit /etc/php/8.3/fpm/pool.d/www.conf
# Update pm.* values as recommended
systemctl restart php8.3-fpm
```

### 4. Disable Unnecessary Plugins
```bash
wp plugin deactivate litespeed-cache wordfence query-monitor \
  --path=/var/www/viceroybali/public_html/
```

### 5. Clean Database
```bash
wp transient delete --all --path=/var/www/viceroybali/public_html/
wp db optimize --path=/var/www/viceroybali/public_html/
```

## Risks & Rollback

All optimizations are low-risk and easily reversible:

| Change | Risk | Rollback Method |
|--------|------|-----------------|
| URL Search-Replace | LOW | Restore database from backup |
| Redis Cache | VERY LOW | `wp redis disable` |
| PHP-FPM Tuning | LOW | Revert config, restart PHP-FPM |
| Plugin Disable | VERY LOW | Re-activate plugins |
| Database Optimize | VERY LOW | Restore from backup |

