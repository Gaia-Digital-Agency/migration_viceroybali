# Navigation Link Fixes

**Date:** February 3, 2026
**Issue:** 8 menu navigation URLs returning 404 errors
**Resolution:** Added Nginx 301 redirects

---

## Problem

The WordPress navigation menu used incorrect URL paths that did not match the actual WordPress page slugs. Users clicking menu items got 404 errors.

## Solution Applied

Added 301 redirects to Nginx configuration to map broken URLs to correct pages.

### Redirects Added

| Broken URL | Redirects To | Section |
|------------|--------------|---------|
| `/viceroybali/en/villas/` | `/viceroybali/en/room/` | Villas |
| `/viceroybali/en/offers/` | `/viceroybali/en/hotel-offers/` | Offers |
| `/viceroybali/en/dining/` | `/viceroybali/en/bali-restaurants/` | Dining |
| `/viceroybali/en/experiences/activities/` | `/viceroybali/en/bali-activities/` | Experiences |
| `/viceroybali/en/experiences/destination/` | `/viceroybali/en/ubud/` | Experiences |
| `/viceroybali/en/media/galleries/` | `/viceroybali/en/gallery/` | Media |
| `/viceroybali/en/media/viceroy-story/` | `/viceroybali/en/about/` | Media |
| `/viceroybali/en/destination/` | `/viceroybali/en/ubud/` | Experiences |

---

## Technical Details

### Nginx Configuration

**File:** `/etc/nginx/sites-available/viceroybali`
**Backup:** `/etc/nginx/sites-available/viceroybali.bak.20260203`

Added after "Root redirect to viceroybali" section:
```nginx
# ==========================================
# FIX: Broken navigation URL redirects
# Added Feb 3, 2026
# ==========================================
location = /viceroybali/en/villas/ { return 301 /viceroybali/en/room/; }
location = /viceroybali/en/offers/ { return 301 /viceroybali/en/hotel-offers/; }
location = /viceroybali/en/dining/ { return 301 /viceroybali/en/bali-restaurants/; }
location = /viceroybali/en/experiences/activities/ { return 301 /viceroybali/en/bali-activities/; }
location = /viceroybali/en/experiences/destination/ { return 301 /viceroybali/en/ubud/; }
location = /viceroybali/en/media/galleries/ { return 301 /viceroybali/en/gallery/; }
location = /viceroybali/en/media/viceroy-story/ { return 301 /viceroybali/en/about/; }
location = /viceroybali/en/destination/ { return 301 /viceroybali/en/ubud/; }
```

### Fix 2: Language Path Redirects (Added Later)

Also added redirects so `/en/...` paths work (redirect to `/viceroybali/en/...`):

```nginx
# Redirect /en/ paths to /viceroybali/en/
location ^~ /en/ {
    rewrite ^/en/(.*)$ /viceroybali/en/$1 redirect;
}
location ^~ /ja/ {
    rewrite ^/ja/(.*)$ /viceroybali/ja/$1 redirect;
}
location ^~ /ko/ {
    rewrite ^/ko/(.*)$ /viceroybali/ko/$1 redirect;
}
```

### Commands Used

```bash
# Backup config
sudo cp /etc/nginx/sites-available/viceroybali /etc/nginx/sites-available/viceroybali.bak.20260203

# Test configuration
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx
```

---

## Verification

All 8 URLs now redirect (301) and resolve to working pages (200):

```
/viceroybali/en/villas/                    → 301 → /viceroybali/en/room/ → 200 ✅
/viceroybali/en/offers/                    → 301 → /viceroybali/en/hotel-offers/ → 200 ✅
/viceroybali/en/dining/                    → 301 → /viceroybali/en/bali-restaurants/ → 200 ✅
/viceroybali/en/experiences/activities/    → 301 → /viceroybali/en/bali-activities/ → 200 ✅
/viceroybali/en/experiences/destination/   → 301 → /viceroybali/en/ubud/ → 200 ✅
/viceroybali/en/media/galleries/           → 301 → /viceroybali/en/gallery/ → 200 ✅
/viceroybali/en/media/viceroy-story/       → 301 → /viceroybali/en/about/ → 200 ✅
/viceroybali/en/destination/               → 301 → /viceroybali/en/ubud/ → 200 ✅
```

---

## Go-Live Consideration

When migrating to production (`https://www.viceroybali.com`), these redirects need to be updated:

**Option A:** Keep redirects but change paths from `/viceroybali/` prefix to root `/`

**Option B:** Update WordPress menu items to use correct URLs directly:
- WP Admin → Appearance → Menus
- Update each menu item's URL

**Recommended:** Keep Nginx redirects for SEO (301 preserves link equity) and also fix menu items in WordPress for clean URLs.

---

---

## Full Page Verification (30 URLs)

All pages verified working on February 3, 2026.

**Both URL formats now work:**
- `http://34.158.47.112/en/...` (redirects to /viceroybali/en/...)
- `http://34.158.47.112/viceroybali/en/...` (direct)

### Direct Pages (22)
| # | URL | Status |
|---|-----|--------|
| 1 | `/en/` | 200 |
| 2 | `/en/room/` | 200 |
| 3 | `/en/room/pool-suite/` | 200 |
| 4 | `/en/room/terrace-villas/` | 200 |
| 5 | `/en/room/deluxe-terrace-villa/` | 200 |
| 6 | `/en/room/premium-club-pool-villa/` | 200 |
| 7 | `/en/room/elephant-villa/` | 200 |
| 8 | `/en/room/vice-regal-villa/` | 200 |
| 9 | `/en/room/viceroy-bali/` | 200 |
| 10 | `/en/room/garden-villa/` | 200 |
| 11 | `/en/hotel-offers/` | 200 |
| 12 | `/en/packages/` | 200 |
| 13 | `/en/bali-restaurants/` | 200 |
| 14 | `/en/wellness/` | 200 |
| 15 | `/en/wellness-experiences/` | 200 |
| 16 | `/en/bali-activities/` | 200 |
| 17 | `/en/ubud/` | 200 |
| 18 | `/en/gallery/` | 200 |
| 19 | `/en/blog/` | 200 |
| 20 | `/en/about/` | 200 |
| 21 | `/en/facilities/` | 200 |
| 22 | `/en/contact/` | 200 |

### Redirected Pages (8) - Now Working
| # | URL | Redirect | Final |
|---|-----|----------|-------|
| 23 | `/en/villas/` | 301 | 200 |
| 24 | `/en/offers/` | 301 | 200 |
| 25 | `/en/dining/` | 301 | 200 |
| 26 | `/en/experiences/activities/` | 301 | 200 |
| 27 | `/en/experiences/destination/` | 301 | 200 |
| 28 | `/en/media/galleries/` | 301 | 200 |
| 29 | `/en/media/viceroy-story/` | 301 | 200 |
| 30 | `/en/destination/` | 301 | 200 |

**Total: 30 URLs verified (22 direct + 8 redirects)**

---

## Related Files

- [Pre_migration.md](Pre_migration.md) - Phase 1.5 section documents this fix
- [port.md](phase2_qa/port.md) - Session log references this fix
- [gda-ce01.md](gda-ce01.md) - Server configuration details
