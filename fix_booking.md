# Fix Book Now Functionality

**Project:** Viceroy Bali Staging Site
**Issue:** Book Now button not working on staging
**Severity:** CRITICAL
**Status:** Solution documented, not yet executed
**Date:** February 1, 2026

## Problem Summary

The **Book Now** functionality on the staging site (http://34.142.200.251) redirects users to the production domain (https://www.viceroybali.com), making it impossible to test the booking flow on staging.

### Observable Symptoms:
- ✅ Book Now button appears on pages
- ✅ Clicking button navigates to `/en/reservation/` page
- ❌ Page content loads from production URLs
- ❌ Images, CSS, navigation point to `https://www.viceroybali.com`
- ❌ Cloudbeds booking widget may not load (mixed content)
- ❌ Cannot complete booking on staging environment

### User Impact:
- **HIGH** - Cannot test booking system before production deployment
- Stakeholders cannot verify booking flow changes
- Risk of deploying untested booking functionality to production

## Root Cause Analysis

### Why This Happens:

WordPress stores content in the database in two ways:

1. **Site URL Settings** (✅ Already Fixed)
   - `wp_options` table: `siteurl` and `home` values
   - These were updated during deployment to staging IP
   - Location: `vb21_options` table

2. **Content URLs** (❌ Not Fixed - This is the problem)
   - Post content (`post_content` field)
   - Post metadata (`postmeta` table)
   - Widget settings
   - Menu items
   - Custom fields
   - Shortcodes and embeds

### Technical Details:

When WordPress content is created, it stores **absolute URLs** like:
```
https://www.viceroybali.com/en/reservation/
```

These URLs are embedded in:
- Page builder content (Elementor, WPBakery, etc.)
- Custom fields (ACF)
- Serialized data structures
- Menu links
- Image src attributes

Simply updating `siteurl` and `home` in `wp_options` doesn't touch these embedded URLs.

### Why We Need WP-CLI:

Database content often contains **serialized PHP arrays**:
```php
a:3:{s:4:"url";s:32:"https://www.viceroybali.com/...";s:5:"title";s:8:"Book Now";}
```

If you do a simple MySQL `REPLACE()`, you'll corrupt serialized data because:
- String length prefix (`s:32`) becomes invalid
- Array gets corrupted
- WordPress can't unserialize data
- Pages break or show blank content

**WP-CLI handles serialized data properly** by:
1. Unserializing data
2. Replacing URLs
3. Re-serializing with correct length prefixes
4. Updating database safely

## Solution

### The Fix: WP-CLI Search-Replace

Use WP-CLI's `search-replace` command to safely update all URLs across the entire database.

### Command:

```bash
wp search-replace 'https://www.viceroybali.com' 'http://34.142.200.251' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid \
  --all-tables \
  --report-changed-only \
  --allow-root
```

### Parameter Breakdown:

| Parameter | Purpose |
|-----------|---------|
| `'https://www.viceroybali.com'` | Old URL to find |
| `'http://34.142.200.251'` | New staging IP to replace with |
| `--path=/var/www/viceroybali/public_html/` | WordPress installation directory |
| `--skip-columns=guid` | Don't change post GUIDs (WordPress best practice) |
| `--all-tables` | Search all database tables (not just wp_ prefixed) |
| `--report-changed-only` | Only show tables with changes (cleaner output) |
| `--allow-root` | Allow running as root user (required on GCP VM) |

### Why Skip GUID Column:

WordPress post GUIDs should **never** be changed:
- They're permanent unique identifiers
- Used by RSS feeds, pingbacks, trackbacks
- Changing them breaks external references
- WordPress core recommendation: leave GUIDs unchanged

## Step-by-Step Execution Plan

### Prerequisites:

1. ✅ SSH access to GCP VM (viceroy-bali)
2. ✅ WP-CLI installed and working
3. ✅ WordPress database accessible
4. ✅ Sufficient disk space for backup

### Execution Steps:

#### Step 1: Create Database Backup

```bash
# Create backup directory if not exists
mkdir -p /var/www/backups/

# Export current database
wp db export /var/www/backups/before_url_fix_$(date +%Y%m%d_%H%M%S).sql \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root

# Verify backup created
ls -lh /var/www/backups/
```

**Expected Output:**
```
Success: Exported to '/var/www/backups/before_url_fix_20260201_143022.sql'.
-rw-r--r-- 1 root root 185M Feb  1 14:30 before_url_fix_20260201_143022.sql
```

#### Step 2: Dry Run (Test Mode)

```bash
# Run with --dry-run to see what would change
wp search-replace 'https://www.viceroybali.com' 'http://34.142.200.251' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid \
  --all-tables \
  --dry-run \
  --allow-root
```

**Expected Output:**
```
+---------------------+-------+
| Table               | Count |
+---------------------+-------+
| vb21_posts          | 142   |
| vb21_postmeta       | 68    |
| vb21_options        | 12    |
| vb21_yoast_indexable| 34    |
+---------------------+-------+
Success: 256 replacements to be made.
```

**Review the output:**
- Check which tables will be affected
- Verify replacement count seems reasonable
- Look for any unexpected tables

#### Step 3: Execute Search-Replace

```bash
# Actual replacement (REMOVES --dry-run)
wp search-replace 'https://www.viceroybali.com' 'http://34.142.200.251' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid \
  --all-tables \
  --report-changed-only \
  --allow-root
```

**Expected Output:**
```
+---------------------+-------+
| Table               | Count |
+---------------------+-------+
| vb21_posts          | 142   |
| vb21_postmeta       | 68    |
| vb21_options        | 12    |
| vb21_yoast_indexable| 34    |
+---------------------+-------+
Success: Made 256 replacements.
```

**Processing time:** 5-30 seconds depending on database size

#### Step 4: Clear Caches

```bash
# Flush WordPress object cache
wp cache flush --path=/var/www/viceroybali/public_html/ --allow-root

# Flush Redis cache (if enabled)
redis-cli FLUSHALL

# Restart PHP-FPM to clear OPcache
systemctl restart php8.3-fpm

# Clear Nginx cache if using FastCGI cache
# rm -rf /var/cache/nginx/*  # Only if FastCGI cache enabled
```

#### Step 5: Verify Fix

```bash
# Check specific URL in database
wp db query "SELECT post_title, post_content FROM vb21_posts WHERE post_content LIKE '%www.viceroybali.com%' LIMIT 5;" \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root
```

**Expected:** Should return empty or very few results

```bash
# Check options table
wp option get home --path=/var/www/viceroybali/public_html/ --allow-root
wp option get siteurl --path=/var/www/viceroybali/public_html/ --allow-root
```

**Expected:**
```
http://34.142.200.251
http://34.142.200.251
```

#### Step 6: Test Book Now Functionality

1. **Open staging site in browser:**
   http://34.142.200.251

2. **Navigate to a page with Book Now button:**
   Home page, room pages, or booking-related pages

3. **Click "Book Now" button:**
   - Should stay on staging domain (34.142.200.251)
   - Should NOT redirect to www.viceroybali.com

4. **Check reservation page:**
   http://34.142.200.251/en/reservation/
   - Images should load from staging IP
   - Navigation links should point to staging IP
   - Cloudbeds booking widget should appear

5. **Test booking flow (if possible):**
   - Select dates
   - Choose room
   - Verify widget functionality

---

## Verification Checklist

After executing the fix:

- [ ] Database backup created successfully
- [ ] Dry run completed without errors
- [ ] Search-replace executed successfully
- [ ] Number of replacements seems reasonable (100-500 typical)
- [ ] Caches cleared (WordPress, Redis, PHP OPcache)
- [ ] Home page loads correctly
- [ ] Book Now button stays on staging domain
- [ ] Reservation page loads without mixed content warnings
- [ ] Cloudbeds widget appears (if configured)
- [ ] Navigation links point to staging IP
- [ ] Images load from staging domain

## Expected Results

### Before Fix:
- Book Now → redirects to `https://www.viceroybali.com/en/reservation/`
- Page content loads from production
- Cannot test booking on staging

### After Fix:
- Book Now → loads `http://34.142.200.251/en/reservation/`
- All content loads from staging IP
- Booking widget functional on staging
- Can test complete booking flow

### Database Changes:

Typical replacement count: **200-500 replacements**

Tables affected:
- `vb21_posts` - Page/post content (highest count)
- `vb21_postmeta` - Custom fields, ACF data
- `vb21_options` - Theme settings, widgets
- `vb21_yoast_indexable` - SEO data
- `vb21_polylang*` - Multi-language links
- Other plugin tables with URLs

## Rollback Plan

If something goes wrong, restore from backup:

### Rollback Procedure:

```bash
# 1. List available backups
ls -lh /var/www/backups/

# 2. Identify the backup file (before_url_fix_*.sql)
BACKUP_FILE="/var/www/backups/before_url_fix_20260201_143022.sql"

# 3. Drop current database and reimport backup
wp db reset --yes --path=/var/www/viceroybali/public_html/ --allow-root
wp db import $BACKUP_FILE --path=/var/www/viceroybali/public_html/ --allow-root

# 4. Clear caches
wp cache flush --path=/var/www/viceroybali/public_html/ --allow-root
redis-cli FLUSHALL
systemctl restart php8.3-fpm

# 5. Verify restoration
wp option get home --path=/var/www/viceroybali/public_html/ --allow-root
```

**Rollback time:** 2-5 minutes

### When to Rollback:

- Site becomes inaccessible after change
- Pages show errors or blank content
- Serialized data corruption errors in logs
- Unexpected behavior in WordPress admin

## Important Notes

### Production Deployment Consideration:

**DO NOT run this command on production!**

When deploying to production, you'll need the **reverse operation**:

```bash
# PRODUCTION ONLY - converts staging URLs back to production
wp search-replace 'http://34.142.200.251' 'https://www.viceroybali.com' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid \
  --all-tables \
  --allow-root
```

This is **CRITICAL** and documented in [port.md](port.md) Step 6.

### SSL Consideration:

Currently using `http://` on staging because:
- No SSL certificate provisioned for IP address
- SSL cert awaits DNS cutover

On production:
- Use `https://www.viceroybali.com`
- SSL certificate auto-provisions after DNS cutover
- Mixed content warnings resolved

### Multi-language Sites:

Viceroy Bali uses **Polylang** for multi-language support:
- URLs stored for each language variant
- Search-replace will update all language versions
- `/en/`, `/id/` paths preserved
- Only domain changes, not URI paths

## Risk Assessment

### Risk Level: **LOW** ⚠️

| Risk Factor | Rating | Mitigation |
|-------------|--------|------------|
| Data Loss | VERY LOW | Database backup before execution |
| Site Downtime | VERY LOW | Operation takes 5-30 seconds |
| Serialized Data Corruption | VERY LOW | WP-CLI handles serialization properly |
| Reversibility | VERY LOW | Full rollback in 2-5 minutes |
| Breaking Functionality | LOW | Staging environment, not production |

### Why This is Safe:

1. **WP-CLI is WordPress-recommended tool**
   - Used by millions of WordPress sites
   - Handles serialized data correctly
   - Well-tested and maintained

2. **Staging environment**
   - Not production site
   - Safe to test changes
   - No user impact if issues occur

3. **Full backup before change**
   - Complete database export
   - Can restore in minutes
   - No data loss risk

4. **Reversible operation**
   - Can run reverse search-replace
   - Or restore from backup
   - Both methods quick and reliable

## Performance Impact

### During Execution:
- **Duration:** 5-30 seconds
- **Resource usage:** Moderate CPU, high disk I/O
- **Site availability:** Online (users may experience brief slowdown)

### After Execution:
- **No performance degradation**
- URL length unchanged (similar character count)
- Database size unchanged
- No added overhead

## Alternative Methods (Not Recommended)

### 1. Better Search Replace Plugin (GUI)
**Pros:** User-friendly interface, no command line
**Cons:** Slower, requires WordPress admin access, less reliable
**Verdict:** Use WP-CLI instead (faster, more reliable)

### 2. Direct MySQL REPLACE()
```sql
-- DON'T USE THIS - Will corrupt serialized data!
UPDATE vb21_posts
SET post_content = REPLACE(post_content, 'https://www.viceroybali.com', 'http://34.142.200.251');
```
**Verdict:** ⚠️ **NEVER USE** - Corrupts serialized data

### 3. Manual Search-Replace in phpMyAdmin
**Pros:** Visual interface
**Cons:** Very slow, error-prone, misses serialized data
**Verdict:** Use WP-CLI instead

## Troubleshooting

### Issue: WP-CLI not found

```bash
# Install WP-CLI
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
chmod +x wp-cli.phar
mv wp-cli.phar /usr/local/bin/wp
wp --info
```

### Issue: Database connection error

```bash
# Verify wp-config.php database credentials
cat /var/www/viceroybali/public_html/wp-config.php | grep DB_

# Test database connection
wp db check --path=/var/www/viceroybali/public_html/ --allow-root
```

### Issue: "Error: The site you have requested is not installed"

```bash
# WP-CLI can't find WordPress installation
# Verify path is correct
ls -la /var/www/viceroybali/public_html/wp-config.php

# Try with explicit --path
wp search-replace --path=/var/www/viceroybali/public_html/ ...
```

### Issue: Replacement count is 0

```bash
# Check if URLs actually exist in database
wp db query "SELECT COUNT(*) FROM vb21_posts WHERE post_content LIKE '%www.viceroybali.com%';" \
  --path=/var/www/viceroybali/public_html/ --allow-root

# Try without https://
wp search-replace 'www.viceroybali.com' '34.142.200.251' --dry-run ...
```

## Commands Quick Reference

### Full Execution Sequence:

```bash
# 1. Backup
wp db export /var/www/backups/before_url_fix_$(date +%Y%m%d_%H%M%S).sql \
  --path=/var/www/viceroybali/public_html/ --allow-root

# 2. Dry run
wp search-replace 'https://www.viceroybali.com' 'http://34.142.200.251' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid --all-tables --dry-run --allow-root

# 3. Execute
wp search-replace 'https://www.viceroybali.com' 'http://34.142.200.251' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid --all-tables --report-changed-only --allow-root

# 4. Clear caches
wp cache flush --path=/var/www/viceroybali/public_html/ --allow-root
redis-cli FLUSHALL
systemctl restart php8.3-fpm

# 5. Verify
wp option get home --path=/var/www/viceroybali/public_html/ --allow-root
```

## Next Steps After Fix

Once Book Now functionality is working on staging:

1. **Complete Testing:**
   - Test all booking flows
   - Verify Cloudbeds integration
   - Check payment processing (if applicable)
   - Test confirmation emails

2. **Document Results:**
   - Update [architecture.md](features/architecture.md) with status
   - Update [optimizations_and_issues.md](optimization/optimizations_and_issues.md)
   - Mark issue as resolved

3. **Prepare for Production:**
   - Document reverse URL change in [port.md](port.md)
   - Add to production deployment checklist
   - Schedule DNS cutover

4. **Optional Optimizations:**
   - Enable Redis object cache (if not done)
   - Tune PHP-FPM pool
   - Optimize database

## Related Documentation

- **[port.md](port.md)** - Production deployment guide (includes reverse URL change)
- **[optimization/optimizations_and_issues.md](optimization/optimizations_and_issues.md)** - Full optimization guide
- **[features/architecture.md](features/architecture.md)** - System architecture
- **[setup/step02.md](setup/step02.md)** - WordPress deployment steps

---

## Status

**Current:** ❌ Not fixed - Book Now redirects to production
**After Execution:** ✅ Fixed - Book Now works on staging
**Execution Time:** ~5 minutes (including backup and verification)
**Difficulty:** Easy (copy-paste commands)
**Risk:** Low (full backup, reversible, staging environment)
