# Staging to Production Migration Guide

**Project:** Viceroy Bali
**Purpose:** Document the complete process for moving staging site to production
**IMPORTANT:** This is documentation only - DO NOT execute without explicit approval

---

## Overview

This document outlines the complete process for transitioning the staging WordPress site (currently on http://34.142.200.251) to production (https://www.viceroybali.com) via DNS cutover.

**Key Principle:** Zero downtime migration with instant rollback capability

---

## Pre-Migration Checklist

### 1. Staging Site Verification ✅

**Must be completed before proceeding:**

- [ ] WordPress loads correctly on http://34.142.200.251
- [ ] All pages render properly (villas, facilities, about)
- [ ] Images display correctly
- [ ] Booking system functional (after URL fix)
- [ ] Contact forms working
- [ ] Multi-language switching works (EN/JA/KO)
- [ ] Admin dashboard accessible
- [ ] No PHP/MySQL errors in logs
- [ ] Performance acceptable (TTFB < 500ms)
- [ ] Mobile responsive design works

### 2. SSL Certificate Status 🔒

**Check Certificate Provisioning:**
```bash
gcloud compute ssl-certificates describe viceroybali-ssl-cert \
  --global \
  --format="value(managed.status,managed.domainStatus)"
```

**Expected Status:**
- **PROVISIONING** - Certificate not yet active (DNS not pointing)
- **ACTIVE** - Certificate ready (only after DNS points to GCP)

**Important:** Certificate auto-activates AFTER DNS cutover

### 3. Load Balancer Health Check ✅

**Verify Backend Healthy:**
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

**If UNHEALTHY:** Do NOT proceed - investigate and fix first

### 4. Backup Production Site (Hostinger) 📦

**CRITICAL: Full backup of current live site**

```bash
# This should already exist, but verify:
gs://viceroybali_bucket/viceroybali_backup.tar.gz (22GB)
gs://viceroybali_bucket/viceroy_db_snapshot.sql (183MB)
```

**Verify backup integrity:**
- Confirm files exist in bucket
- Check file sizes are correct
- Note backup timestamp

**Purpose:** Rollback capability if GCP site has issues

---

## Pre-Cutover: Update Database URLs

### Step 1: Change from Staging to Production URLs

**Current State:** Database contains `http://34.142.200.251`
**Target State:** Database should contain `https://www.viceroybali.com`

**Method 1: WP-CLI (Recommended)**

```bash
# SSH to server
ssh -i ~/.ssh/id_ed25519_gaia root@34.142.200.251

# Backup database first
wp db export /var/www/backups/before_production_urls.sql \
  --path=/var/www/viceroybali/public_html/

# Search and replace URLs
wp search-replace 'http://34.142.200.251' 'https://www.viceroybali.com' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid \
  --all-tables \
  --report-changed-only \
  --dry-run

# If dry-run looks good, run without --dry-run
wp search-replace 'http://34.142.200.251' 'https://www.viceroybali.com' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid \
  --all-tables
```

**Method 2: Direct Database Update (Alternative)**

```bash
mysql -u viceroy_user -pstrong_password viceroy_db_name << EOF
UPDATE vb21_options SET option_value = 'https://www.viceroybali.com' WHERE option_name = 'siteurl';
UPDATE vb21_options SET option_value = 'https://www.viceroybali.com' WHERE option_name = 'home';

-- Update post content (be careful with serialized data)
UPDATE vb21_posts SET post_content = REPLACE(post_content, 'http://34.142.200.251', 'https://www.viceroybali.com');
UPDATE vb21_posts SET guid = REPLACE(guid, 'http://34.142.200.251', 'https://www.viceroybali.com');

-- Update postmeta
UPDATE vb21_postmeta SET meta_value = REPLACE(meta_value, 'http://34.142.200.251', 'https://www.viceroybali.com');
EOF
```

**IMPORTANT:** WP-CLI method is safer for serialized data

### Step 2: Verify URL Updates

```bash
# Check site URL
wp option get siteurl --path=/var/www/viceroybali/public_html/
# Should return: https://www.viceroybali.com

# Check home URL
wp option get home --path=/var/www/viceroybali/public_html/
# Should return: https://www.viceroybali.com

# Search for any remaining staging URLs
wp db search '34.142.200.251' \
  --path=/var/www/viceroybali/public_html/
# Should return minimal or no results
```

### Step 3: Update wp-config.php (Optional)

Remove staging-specific settings:

```bash
# Edit wp-config.php
nano /var/www/viceroybali/public_html/wp-config.php

# Change or remove:
define('WP_ENVIRONMENT_TYPE', 'staging');  // Remove this line

# Optionally add:
define('WP_ENVIRONMENT_TYPE', 'production');
```

---

## DNS Cutover Process

### Understanding Current DNS

**Before Migration:**
```
www.viceroybali.com → Hostinger IP (e.g., 1.2.3.4)
viceroybali.com → Hostinger IP (e.g., 1.2.3.4)
```

**After Migration:**
```
www.viceroybali.com → GCP Load Balancer (34.49.188.147)
viceroybali.com → GCP Load Balancer (34.49.188.147)
```

### Method 1: Gradual Cutover (Lower Risk, Recommended)

**Step 1: Update WWW subdomain first**

1. Login to DNS provider (wherever viceroybali.com is managed)
2. Find the A record for `www.viceroybali.com`
3. Change IP from Hostinger to `34.49.188.147`
4. Set TTL to 300 seconds (5 minutes)
5. Save changes

**Step 2: Monitor WWW subdomain**

```bash
# Check DNS propagation
dig www.viceroybali.com +short
# Should return: 34.49.188.147 (may take 5-60 min)

# Test WWW site
curl -I https://www.viceroybali.com
# Should return: HTTP/2 200 (once DNS propagates)
```

**Step 3: Update apex domain**

Once WWW is working:
1. Update A record for `viceroybali.com` to `34.49.188.147`
2. Save changes

### Method 2: Simultaneous Cutover (Faster, Higher Risk)

Update both www and apex domain simultaneously to `34.49.188.147`

**Advantage:** Faster complete migration
**Disadvantage:** Cannot test www first before apex

---

## Post-Cutover Verification

### 1. DNS Propagation Check

```bash
# Check DNS from multiple locations
dig www.viceroybali.com +short
dig viceroybali.com +short
# Both should return: 34.49.188.147

# Check from different DNS servers
dig @8.8.8.8 www.viceroybali.com +short  # Google DNS
dig @1.1.1.1 www.viceroybali.com +short  # Cloudflare DNS
```

### 2. SSL Certificate Activation

**Monitor Certificate Status:**
```bash
gcloud compute ssl-certificates describe viceroybali-ssl-cert \
  --global \
  --format="value(managed.status)"
```

**Timeline:**
- Immediately after DNS: PROVISIONING
- 15-30 minutes: Still PROVISIONING (Google verifying domain)
- 30-60 minutes: ACTIVE (Certificate issued and active)

**During Provisioning:**
- Site may show "Connection not secure" warnings
- This is normal and temporary
- Inform stakeholders of this transition period

### 3. Site Functionality Tests

**Critical Tests:**
- [ ] Homepage loads via https://www.viceroybali.com
- [ ] Apex domain redirects to www properly
- [ ] All pages accessible (villas, facilities, about, contact)
- [ ] Images loading correctly
- [ ] Booking widget functional (Cloudbeds)
- [ ] Contact forms working
- [ ] Language switching (EN/JA/KO)
- [ ] Admin dashboard accessible
- [ ] No mixed content warnings in browser console

### 4. Performance Verification

```bash
# Test Time to First Byte (TTFB)
curl -w "@curl-format.txt" -o /dev/null -s https://www.viceroybali.com

# Check Load Balancer metrics in GCP Console
# Navigate to: Network Services → Load Balancing → viceroybali-url-map
# Verify: Request count, Latency, Error rate
```

### 5. Monitor Logs

```bash
# SSH to server
ssh -i ~/.ssh/id_ed25519_gaia root@34.142.200.251

# Watch for errors
tail -f /var/log/nginx/viceroybali_error.log

# Monitor access patterns
tail -f /var/log/nginx/viceroybali_access.log
```

---

## SEO Considerations

### 1. Verify No URL Structure Changes ✅

**Critical:** All URLs must remain identical
```
https://www.viceroybali.com/en/villas/
https://www.viceroybali.com/en/facilities/
https://www.viceroybali.com/en/about-ubud/
```

**Test:** Every major URL from old site should work on new site

### 2. Submit to Google Search Console

After DNS cutover and SSL active:

1. Login to Google Search Console
2. Verify property ownership (if needed)
3. Submit sitemap: `https://www.viceroybali.com/sitemap_index.xml`
4. Request re-indexing of key pages

### 3. Monitor Rankings

**First 72 hours:** Watch for any ranking drops
**If drops occur:** Check for:
- 404 errors
- Redirect loops
- Mixed content issues
- Broken internal links

---

## Rollback Procedure

### If Critical Issues Discovered:

**Immediate Rollback (Minutes):**

```bash
# Option 1: Quick DNS Rollback
# Change DNS back to Hostinger IP
# Previous A records for www.viceroybali.com and viceroybali.com

# Option 2: Temporarily Point to Hostinger while fixing
```

**Database Rollback (If URLs were changed):**

```bash
# Restore database from pre-cutover backup
mysql -u viceroy_user -pstrong_password viceroy_db_name < /var/www/backups/before_production_urls.sql

# Or run reverse search-replace
wp search-replace 'https://www.viceroybali.com' 'http://34.142.200.251' \
  --path=/var/www/viceroybali/public_html/ \
  --skip-columns=guid \
  --all-tables
```

**Important:** Hostinger site remains untouched during entire migration, so rollback is always possible

---

## Post-Migration Tasks

### 1. Update DNS TTL

After 24-48 hours of stable operation:
```
# Increase TTL from 300s (5 min) to 3600s (1 hour)
# This reduces DNS query load
```

### 2. Disable Old Site (Later)

**DO NOT** disable Hostinger immediately:
- Keep live for 7-14 days as backup
- Monitor GCP site for any issues
- After confidence period, can cancel Hostinger

### 3. Setup Monitoring

**GCP Uptime Checks:**
```bash
gcloud monitoring uptime-check-configs create viceroybali-uptime \
  --display-name="Viceroy Bali Uptime" \
  --http-check-path=/ \
  --period=60 \
  --timeout=10s \
  --host=www.viceroybali.com \
  --port=443
```

**Alert Policies:**
- SSL certificate expiring (GCP auto-renews, but monitor)
- Backend unhealthy
- High error rate (4xx/5xx)
- Response time > 2s

### 4. Enable Production Optimizations

After stable for 1 week:

```bash
# Set OPcache for production
nano /etc/php/8.3/fpm/php.ini
# opcache.validate_timestamps=0
# opcache.save_comments=0

systemctl restart php8.3-fpm
```

### 5. Schedule Backups

```bash
# Create weekly snapshot schedule (if not already)
gcloud compute resource-policies create snapshot-schedule viceroybali-weekly \
  --region=asia-southeast1 \
  --max-retention-days=28 \
  --start-time=03:00 \
  --weekly-schedule=sunday

# Attach to disk
gcloud compute disks add-resource-policies viceroy-bali \
  --zone=asia-southeast1-b \
  --resource-policies=viceroybali-weekly
```

### 6. Documentation Updates

Update all documentation with:
- Live site now on GCP
- Remove staging IP references
- Update team access credentials if changed
- Document any production-specific settings

---

## Timeline & Coordination

### Pre-Migration Phase (2-3 days before)

**Day -2:**
- Complete all optimizations
- Fix Book Now URLs
- Enable Redis cache
- Final staging tests

**Day -1:**
- Stakeholder notification (migration planned)
- Final backup verification
- Pre-flight checks complete

### Migration Day

**Low-Traffic Window Recommended:**
- Best time: Sunday 2:00 AM - 6:00 AM Bali time
- Reason: Minimal guest impact if issues occur

**Hour 0:00** - Start Migration
- Update database URLs (staging → production)
- Final staging verification with production URLs

**Hour 0:15** - DNS Cutover
- Update WWW subdomain
- Monitor propagation

**Hour 0:30** - Apex Domain Update
- Update apex domain
- Begin full monitoring

**Hour 1:00-2:00** - SSL Certificate Provisioning
- Wait for Google to issue certificate
- Site may show warnings during this period

**Hour 2:00** - Verification
- SSL active
- All tests passing
- Monitor for 2-4 hours

**Hour 6:00** - Migration Complete
- Notify stakeholders
- Continue monitoring for 24-48 hours

---

## Communication Template

### Pre-Migration Notice (24 hours before)

**Subject:** Viceroy Bali Website Migration - Scheduled Maintenance

**Body:**
```
Dear Team,

We will be migrating the Viceroy Bali website to a new high-performance
cloud infrastructure on [DATE] at [TIME] Bali time.

Expected duration: 2-3 hours
Expected impact: Minimal - website will remain accessible

During migration:
- Website will remain online
- Temporary SSL certificate warnings possible (15-60 minutes)
- Full functionality restored within 2 hours

We have full rollback capability if any issues occur.

Please report any issues immediately to [CONTACT].

Thank you,
IT Team
```

### Post-Migration Notice

**Subject:** Viceroy Bali Website Migration - Complete

**Body:**
```
Dear Team,

The website migration to Google Cloud Platform is now complete.

New Infrastructure:
- Faster global performance
- Enhanced security
- Improved reliability
- Cloud CDN for worldwide visitors

The website remains at: https://www.viceroybali.com
All functionality has been verified and tested.

Please report any issues to [CONTACT].

Thank you,
IT Team
```

---

## Troubleshooting

### Issue: Site not loading after DNS change

**Check:**
1. DNS propagation: `dig www.viceroybali.com +short`
2. Load Balancer health: `gcloud compute backend-services get-health viceroybali-backend --global`
3. Nginx running: `systemctl status nginx`
4. Firewall rules: `gcloud compute firewall-rules list --filter="viceroybali"`

### Issue: SSL certificate stuck in PROVISIONING

**Causes:**
- DNS not fully propagated
- Firewall blocking port 80/443
- Health check failing

**Solution:**
- Wait 60-90 minutes
- Verify DNS points to correct IP
- Check firewall allows 0.0.0.0/0 on ports 80/443

### Issue: Mixed content warnings

**Cause:** Some resources loading via HTTP instead of HTTPS

**Solution:**
```bash
# Find HTTP references
wp db search 'http://www.viceroybali.com' \
  --path=/var/www/viceroybali/public_html/

# Replace with HTTPS
wp search-replace 'http://www.viceroybali.com' 'https://www.viceroybali.com' \
  --path=/var/www/viceroybali/public_html/ \
  --all-tables
```

### Issue: Images not loading

**Check:**
1. File permissions: `ls -la /var/www/viceroybali/public_html/wp-content/uploads/`
2. Nginx serving files: `curl -I https://www.viceroybali.com/wp-content/uploads/2023/12/Viceroy-bali-logo.png`
3. CDN caching properly: Check response headers for X-Cache

### Issue: Booking widget not working

**Check:**
1. Cloudbeds script loading: View page source, search for "cloudbeds"
2. No JavaScript errors: Open browser console
3. URLs all HTTPS: Check for mixed content
4. Contact Cloudbeds if widget not displaying

---

## Success Criteria

Migration considered successful when:

- [ ] DNS points to GCP Load Balancer (34.49.188.147)
- [ ] SSL certificate status: ACTIVE
- [ ] Homepage loads via HTTPS
- [ ] All major pages accessible
- [ ] Images display correctly
- [ ] Booking system functional
- [ ] No 404 errors for key pages
- [ ] No mixed content warnings
- [ ] Performance meets expectations (TTFB < 500ms)
- [ ] No spike in error logs
- [ ] Google Search Console shows no critical errors
- [ ] Stakeholders approve go-live

---

## Final Notes

**Important Reminders:**

1. ⚠️ **Never delete Hostinger** until 14+ days of stable GCP operation
2. 📊 **Monitor for 72 hours** - First 3 days are critical
3. 🔒 **SSL takes time** - 15-60 minutes for certificate activation is normal
4. 📱 **Test on mobile** - Verify responsive design works
5. 🔍 **Watch SEO** - Monitor rankings for first 2 weeks
6. 💾 **Backups exist** - Full rollback capability maintained
7. 🎯 **Zero downtime** - Site remains accessible throughout migration
8. 📞 **Communication** - Keep stakeholders informed at each step

**This is a low-risk migration** because:
- Old site remains untouched (instant rollback)
- Infrastructure pre-tested on staging
- DNS cutover is instant and reversible
- No content or structure changes
- Monitoring in place

---

**Status:** Documentation Complete - Ready for execution upon approval
**Estimated Migration Time:** 2-3 hours (mostly waiting for SSL)
**Risk Level:** LOW (with proper preparation and backups)
**Rollback Time:** < 15 minutes (DNS revert)
