# Google Cloud CDN Configuration

**Date:** January 31, 2026
**Project:** Viceroy Bali Migration

---

## Overview

Google Cloud CDN is enabled on the backend service to cache static assets globally, reducing latency for international visitors.

## CDN Configuration

### Backend Service with CDN Enabled

```bash
gcloud compute backend-services create viceroybali-backend \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=viceroybali-health-check \
  --global \
  --enable-cdn \
  --cache-mode=CACHE_ALL_STATIC \
  --default-ttl=3600
```

### CDN Settings

| Setting | Value | Description |
|---------|-------|-------------|
| **Cache Mode** | CACHE_ALL_STATIC | Caches all static content automatically |
| **Default TTL** | 3600 seconds (1 hour) | How long content stays in cache |
| **Protocol** | HTTP | Backend protocol |
| **Scope** | Global | CDN edge locations worldwide |

### Cached Content Types

The following file types are cached by Cloud CDN:

- **Images:** .jpg, .jpeg, .png, .gif, .ico, .webp, .svg
- **Stylesheets:** .css
- **Scripts:** .js
- **Documents:** .pdf
- **Fonts:** .woff, .woff2, .ttf, .eot

### Nginx Static File Configuration

Static files are configured with long cache headers in Nginx:

```nginx
location ~* \.(jpg|jpeg|png|gif|ico|css|js|pdf|webp|woff|woff2|ttf|svg|eot)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
    access_log off;
}
```

### WP Uploads Caching

```nginx
location ~ ^/wp-content/uploads/ {
    expires 30d;
    add_header Cache-Control "public";
}
```

## Cache Behavior

1. **First Request:** Content is fetched from origin server and stored in CDN
2. **Subsequent Requests:** Content served from nearest CDN edge location
3. **Cache Expiry:** After TTL expires, CDN fetches fresh content from origin

## Benefits

- **Reduced Latency:** Static assets served from edge locations near users
- **Lower Origin Load:** Origin server handles fewer requests for static content
- **Better Performance:** Faster page loads for international visitors
- **Cost Savings:** Reduced bandwidth usage from origin server

## Verification

Check backend service CDN status:

```bash
gcloud compute backend-services describe viceroybali-backend \
  --global \
  --format="get(enableCDN,cdnPolicy)"
```

## Cache Invalidation

If you need to purge cached content:

```bash
gcloud compute url-maps invalidate-cdn-cache viceroybali-url-map \
  --path="/*" \
  --async
```

## Notes

- CDN caching is automatic for static content
- Dynamic PHP content (WordPress pages) is not cached by CDN
- Use WordPress caching plugins (WP Rocket, LiteSpeed) for dynamic content caching
- SSL is terminated at the Load Balancer level, CDN serves content over HTTPS

---

**Status:** Active
**CDN Edge Locations:** Global (200+ locations worldwide)
