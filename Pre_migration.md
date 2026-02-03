# Pre-Migration Checklist

**Project:** Viceroy Bali - GCP Migration
**Purpose:** Tasks remaining before DNS cutover
**Last Updated:** February 2, 2026
**Status:** PHASE 1 COMPLETE - Ready for Phase 2

---

## Summary

| Phase | Tasks | When |
|-------|-------|------|
| **Phase 1: Prep Work** | 9 tasks | Do NOW |
| **Phase 2: Cutover Day** | 3 tasks | On DNS change day |

---

# Phase 1: Prep Work (Do Now)

Complete these tasks before scheduling the DNS cutover.

---

## 1. Verify Load Balancer Health

**Status:** ✅ DONE (Feb 2, 2026)
**Priority:** CRITICAL
**Result:** HEALTHY - Backend instance responding on port 80

```bash
gcloud compute backend-services get-health viceroybali-backend --global
```

**Expected Output:**
```
healthState: HEALTHY
instance: .../instances/viceroy-bali
ipAddress: 10.148.0.3
port: 80
```

**If UNHEALTHY:** Investigate and fix before proceeding

---

## 2. Fix 404 Errors on Minified Assets

**Status:** ✅ DONE (Feb 2, 2026)
**Priority:** HIGH
**Result:** Cleared WP Rocket cache - 404 errors resolved on regeneration

**Issue:** 164 console errors across pages - minified CSS/JS files returning 404:
- `/wp-content/cache/min/1/ajax/libs/jqueryui/1.12.1/themes/base/...`
- `/wp-content/cache/min/1/wp-content/themes/viceroybali-git/...`
- `/wp-content/cache/min/1/ajax/libs/font-awesome/4.7.0/css/...`

**Cause:** WP Rocket or previous caching plugin minification pointing to non-existent cache files

**Solution:**
```bash
# SSH to server
ssh -i ~/.ssh/id_ed25519_gaia root@34.158.47.112

# Clear WP Rocket minification cache
rm -rf /var/www/viceroybali/public_html/wp-content/cache/min/*

# Regenerate via WP Admin → WP Rocket → Clear Cache
# OR use WP-CLI if WP Rocket CLI is available
wp rocket clean --confirm --path=/var/www/viceroybali/public_html/ --allow-root
```

**Verify:** Refresh pages, check browser console for 404 errors

---

## 3. Fix Mixed Content on Wellness Page

**Status:** ✅ DONE (Feb 2, 2026)
**Priority:** MEDIUM
**Result:** Videos stored in vb21_postmeta - will auto-resolve in Phase 2 URL replacement

**Issue:** Videos loading from production domain on staging:
- `https://www.viceroybali.com/wp-content/uploads/2023/11/viceroy-wellness.mp4`
- `https://www.viceroybali.com/wp-content/uploads/2023/11/Akoya-.mp4`

**Page:** `/en/wellness-experiences/`

**Solution:** Find and update the video URLs in WordPress:
```bash
# Find where these URLs are stored
wp db search 'viceroy-wellness.mp4' --path=/var/www/viceroybali/public_html/ --allow-root
wp db search 'Akoya-.mp4' --path=/var/www/viceroybali/public_html/ --allow-root
```

Then update in WP Admin (likely in page content or ACF fields), OR these will auto-resolve after Phase 2 URL replacement.

---

## 4. Add Security Headers

**Status:** ✅ DONE (Feb 2, 2026)
**Priority:** MEDIUM
**Result:** Already configured! All headers present (X-Frame-Options, X-Content-Type-Options, X-XSS-Protection, Referrer-Policy, CSP, Permissions-Policy). HSTS commented out until SSL active.

**Current Score:** 86% (missing `strict-transport-security` - intentional until HTTPS)

**Edit Nginx config:**
```bash
nano /etc/nginx/sites-available/viceroybali
```

**Add inside the `server` block:**
```nginx
# Security headers
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# Add HSTS after SSL is active (uncomment after DNS cutover)
# add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

**Apply changes:**
```bash
nginx -t && systemctl reload nginx
```

**Note:** HSTS should only be enabled after SSL is active (post-DNS cutover)

---

## 5. Verify Google Analytics/Tag Manager

**Status:** ✅ DONE (Feb 2, 2026)
**Priority:** MEDIUM
**Result:** GTM-W58S7SL IS present and loading on staging! Found in theme header.php and ss-gtag.js. Automated report had detection issue.

**Live Site:** GTM-W58S7SL detected
**Staging:** GTM-W58S7SL detected (verified)

**Check these locations:**
1. WP Admin → Appearance → Theme Settings (if GTM is theme-configured)
2. WP Admin → Plugins → look for analytics plugins (MonsterInsights, etc.)
3. Theme files: `header.php` or `functions.php`

```bash
# Search for GTM code
grep -r "GTM-W58S7SL" /var/www/viceroybali/public_html/wp-content/themes/
grep -r "googletagmanager" /var/www/viceroybali/public_html/wp-content/themes/
```

**Likely Cause:** GTM may have domain restriction or staging IP not included

**Action:** Note findings - GTM should work automatically after DNS cutover if domain-restricted

---

## 6. Clean Up Disk Space (~2.7 GB)

**Status:** ✅ DONE (Feb 2, 2026)
**Priority:** LOW
**Result:** Freed ~16 GB! Disk usage went from 80% (9.7 GB free) to 47% (26 GB free). Removed old backup tarball, LiteSpeed cache, Hostinger files.

```bash
# SSH to server
ssh -i ~/.ssh/id_ed25519_gaia root@34.158.47.112

# Check current disk usage
df -h /

# Remove old backups (2.2 GB)
rm -rf /var/www/viceroybali/misc_backups/*
rm -rf /var/www/viceroybali/public_html/wp-content/backups-dup-lite/
rm -rf /var/www/viceroybali/public_html/wp-content/updraft/*
rm -rf /var/www/viceroybali/wordpress-backups/

# Remove LiteSpeed cache (now using WP Rocket)
rm -rf /var/www/viceroybali/lscache/*
rm -rf /var/www/viceroybali/public_html/wp-content/litespeed/*

# Remove plugin archives
rm -f /var/www/viceroybali/public_html/wp-content/plugins.zip
rm -f /var/www/viceroybali/public_html/wp-content/plugins/acfpro.zip

# Remove Hostinger-specific files
rm -rf /var/www/viceroybali/.cagefs/
rm -rf /var/www/viceroybali/.cpanel/
rm -rf /var/www/viceroybali/public_ftp/
rm -rf /var/www/viceroybali/cache/

# Check disk usage after
df -h /
```

**Expected:** Free up ~2.7 GB

**Reference:** [can_remove.md](optimization/can_remove.md) for full list

---

## 7. Fix Premium Club Pool CLS Issue

**Status:** ✅ REVIEWED (Feb 2, 2026)
**Priority:** LOW
**Result:** CLS caused by Swiper slider initialization. Requires theme CSS changes to set container height/aspect-ratio. Page functions correctly - this is a performance optimization for future.

**Issue:** CLS of 1.403 (should be < 0.1)
**Page:** `/en/premium-club-pool/`

**Cause:** Layout shift during page load - likely images without explicit dimensions

**Diagnose:**
1. Open page in Chrome DevTools
2. Run Lighthouse → Performance
3. Look for "Avoid large layout shifts" section

**Common fixes:**
- Add `width` and `height` attributes to `<img>` tags
- Add CSS `aspect-ratio` to image containers
- Ensure fonts have `font-display: swap`

**Check in WordPress:**
- WP Admin → Pages → Premium Club Pool
- Look for images without dimensions in page builder

---

## 8. Review Broken External Links

**Status:** ✅ REVIEWED (Feb 2, 2026)
**Priority:** LOW
**Result:** TravellerMade link now returns 200 OK (was temp issue). AmEx link returns 503 but not found in WordPress database - likely external/CDN content. No action needed.

**URLs checked:**
1. `https://travellermade.com/...` - ✅ Working (200 OK)
2. `https://www.americanexpress.com/...` - External issue (503), not in our DB

**Find where these links are:**
```bash
wp db search 'travellermade.com' --path=/var/www/viceroybali/public_html/ --allow-root
wp db search 'americanexpress.com' --path=/var/www/viceroybali/public_html/ --allow-root
```

**Action:** Update or remove these links in WordPress content (likely on About or Awards page)

---

## 9. Remove Unused Themes

**Status:** ✅ DONE (Feb 2, 2026)
**Priority:** LOW
**Result:** Removed 5 unused themes (~24 MB): viceroy-git, viceroy-git-2, viceroy-git-3, viceroybali, viceroyblock. Kept active theme (viceroybali-git) and fallback (twentytwentyfive).

**First, verify active theme:**
```bash
wp option get template --path=/var/www/viceroybali/public_html/ --allow-root
wp option get stylesheet --path=/var/www/viceroybali/public_html/ --allow-root
```

**Current themes:**
- `viceroybali` - Active (KEEP)
- `twentytwentyfive` - Default fallback (KEEP)
- `viceroy-git`, `viceroy-git-2`, `viceroy-git-3` - Old versions?
- `viceroybali-git`, `viceroyblock` - Old versions?

**Remove unused themes (after verifying active):**
```bash
# List all themes
ls -la /var/www/viceroybali/public_html/wp-content/themes/

# Remove old versions (adjust based on active theme)
rm -rf /var/www/viceroybali/public_html/wp-content/themes/viceroy-git
rm -rf /var/www/viceroybali/public_html/wp-content/themes/viceroy-git-2
rm -rf /var/www/viceroybali/public_html/wp-content/themes/viceroy-git-3
rm -rf /var/www/viceroybali/public_html/wp-content/themes/viceroybali-git
rm -rf /var/www/viceroybali/public_html/wp-content/themes/viceroyblock
```

**Space saved:** ~50 MB

---

# Phase 2: Cutover Day (On DNS Change Day)

Complete these tasks on the day of DNS cutover, in this exact order.

---

## 10. Run Go-Live URL Replacement

**Status:** ⬜ NOT DONE
**Priority:** CRITICAL
**When:** Immediately before DNS change

Replace all staging URLs with production URLs in database.

```bash
# SSH to server
ssh -i ~/.ssh/id_ed25519_gaia root@34.158.47.112

# Backup database first
wp db export /var/www/backups/before_production_urls_$(date +%Y%m%d_%H%M%S).sql \
  --path=/var/www/viceroybali/public_html/ --allow-root

# Dry run first
wp search-replace 'http://34.158.47.112' 'https://www.viceroybali.com' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid \
  --all-tables \
  --dry-run \
  --allow-root

# Execute if dry run looks good
wp search-replace 'http://34.158.47.112' 'https://www.viceroybali.com' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid \
  --all-tables \
  --report-changed-only \
  --allow-root
```

**Expected:** ~30 replacements (pages, nav items, ACF fields updated in Feb 2026)

**Verify:**
```bash
wp option get home --path=/var/www/viceroybali/public_html/ --allow-root
# Expected: https://www.viceroybali.com
```

---

## 11. Clear All Caches

**Status:** ⬜ NOT DONE
**Priority:** CRITICAL
**When:** Immediately after URL replacement

```bash
# WordPress object cache
wp cache flush --path=/var/www/viceroybali/public_html/ --allow-root

# Delete all transients
wp transient delete --all --path=/var/www/viceroybali/public_html/ --allow-root

# Redis cache
redis-cli FLUSHALL

# Restart PHP-FPM (clear OPcache)
systemctl restart php8.3-fpm

# If WP Rocket installed
wp rocket clean --confirm --path=/var/www/viceroybali/public_html/ --allow-root 2>/dev/null || true
```

---

## 12. Verify ACF Pro License (Post Go-Live)

**Status:** ⬜ NOT DONE
**Priority:** LOW
**When:** After DNS cutover and site is live

**Note:** ACF Pro license may need re-activation after domain change

**Check:** WP Admin → Custom Fields → Updates

If license shows inactive:
1. Go to ACF website
2. Get license key from account
3. Re-enter in WordPress

---

# Pre-Cutover Verification Checklist

Run these checks after completing Phase 1 and before starting Phase 2.

## Phase 1 Completion Checklist

### Infrastructure
- [x] Load Balancer health: HEALTHY ✅
- [x] Nginx running ✅
- [x] PHP-FPM running ✅
- [x] Redis running ✅
- [x] MariaDB running ✅

### Performance & Errors
- [x] 404 errors fixed (minified assets) ✅
- [x] Console errors reduced ✅
- [x] Mixed content warnings addressed (auto-resolve in Phase 2) ✅
- [x] Security headers already configured ✅

### Cleanup
- [x] Disk space cleaned (~16 GB freed!) ✅
- [x] Unused themes removed (~24 MB) ✅
- [x] Broken external links reviewed ✅

### Verification
- [x] GTM present and loading on staging ✅
- [x] Premium Club Pool CLS reviewed (theme change needed) ✅
- [x] All main pages load correctly on staging ✅

---

## Phase 2 Cutover Checklist

### Before URL Replacement
- [ ] Phase 1 tasks completed
- [ ] Low-traffic window selected
- [ ] Stakeholders notified
- [ ] Database backup created

### After URL Replacement
- [ ] All caches cleared
- [ ] `wp option get home` returns `https://www.viceroybali.com`
- [ ] No staging URLs remain: `wp db search '34.158.47.112'` returns empty

### After DNS Change
- [ ] DNS propagated: `dig www.viceroybali.com +short` → `34.49.188.147`
- [ ] SSL certificate: `ACTIVE` (may take 15-60 min)
- [ ] Homepage loads via HTTPS
- [ ] Booking widget functional
- [ ] GTM/Analytics firing
- [ ] ACF Pro license verified

---

## DNS Cutover Day - Quick Reference

### Hour 0:00 - URL Replacement (Phase 2, Task 10)
```bash
wp db export /var/www/backups/pre_dns_$(date +%Y%m%d_%H%M%S).sql \
  --path=/var/www/viceroybali/public_html/ --allow-root

wp search-replace 'http://34.158.47.112' 'https://www.viceroybali.com' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid --all-tables --allow-root
```

### Hour 0:05 - Clear Caches (Phase 2, Task 11)
```bash
wp cache flush --path=/var/www/viceroybali/public_html/ --allow-root
wp transient delete --all --path=/var/www/viceroybali/public_html/ --allow-root
redis-cli FLUSHALL
systemctl restart php8.3-fpm
```

### Hour 0:10 - DNS Change
1. Update `www.viceroybali.com` A record → `34.49.188.147`
2. Update `viceroybali.com` A record → `34.49.188.147`
3. Set TTL to 300 seconds
4. Save changes

### Hour 0:30 - Verify DNS
```bash
dig www.viceroybali.com +short
# Expected: 34.49.188.147
```

### Hour 1:00 - Monitor SSL
```bash
gcloud compute ssl-certificates describe viceroybali-ssl-cert \
  --global --format="value(managed.status)"
# Wait for: ACTIVE (15-60 min)
```

### Hour 2:00 - Final Verification
- [ ] Site loads via HTTPS
- [ ] All pages working
- [ ] Booking system functional
- [ ] Enable HSTS header in Nginx (uncomment line)

---

## Contacts & Resources

| Resource | Location |
|----------|----------|
| GCP Load Balancer IP | 34.49.188.147 |
| Staging Server IP | 34.158.47.112 |
| SSH Key | ~/.ssh/id_ed25519_gaia |
| Backup Bucket | gs://viceroybali_bucket/ |
| Full Migration Guide | [port.md](port.md) |
| Optimization Guide | [optimization/optimizations_and_issues.md](optimization/optimizations_and_issues.md) |
| Cleanup Guide | [optimization/can_remove.md](optimization/can_remove.md) |

---

## Status Legend

| Symbol | Meaning |
|--------|---------|
| ⬜ | Not started |
| 🔄 | In progress |
| ✅ | Completed |

---

**Current Phase:** Phase 1 COMPLETE ✅
**Next Action:** Schedule DNS cutover and execute Phase 2
**Phase 1 Completed:** February 2, 2026
**Estimated Phase 2 Time:** 30 minutes + DNS propagation
