# GDA-CE01 Staging Instance

**Date Created:** February 2, 2026
**Last Updated:** February 3, 2026
**Cloned From:** viceroy-bali
**Purpose:** Multi-site staging/testing environment

---

## Instance Details

| Property | Value |
|----------|-------|
| Instance Name | gda-ce01 |
| Zone | asia-southeast1-b |
| External IP | 34.158.47.112 (Static) |
| Internal IP | 10.148.0.4 |
| Status | RUNNING |
| Instance Group | gaiada-ce |

### Static IP Configuration
```bash
# IP reserved as static
gcloud compute addresses create gda-ce01-static \
  --region=asia-southeast1 \
  --addresses=34.158.47.112

# Verify
gcloud compute addresses list --filter="region:asia-southeast1"
```

---

## Instance Group (UMIG)

| Property | Value |
|----------|-------|
| Name | gaiada-ce |
| Zone | asia-southeast1-b |
| Named Ports | http:80 |
| Current Instances | 1 (gda-ce01) |

### Add more instances later
```bash
gcloud compute instance-groups unmanaged add-instances gaiada-ce \
  --zone=asia-southeast1-b \
  --instances=INSTANCE_NAME
```

### List instances in group
```bash
gcloud compute instance-groups list-instances gaiada-ce --zone=asia-southeast1-b
```

---

## Access

### SSH via gcloud
```bash
gcloud compute ssh gda-ce01 --zone=asia-southeast1-b
```

### SSH via key
```bash
ssh -i ~/.ssh/id_ed25519_gaia root@34.158.47.112
```

---

## Multi-Site Staging URLs

| Site | URL | Location |
|------|-----|----------|
| Viceroy Bali (WordPress) | http://34.158.47.112/viceroybali/ | `/var/www/viceroybali/public_html/` |
| 02 Production (Placeholder) | http://34.158.47.112/02production/ | `/var/www/02production/` |
| 03 Production (Placeholder) | http://34.158.47.112/03production/ | `/var/www/03production/` |

### WordPress Pages
- Homepage: http://34.158.47.112/viceroybali/en/
- Reservation: http://34.158.47.112/viceroybali/en/reservation/

---

## Folder Structure

```
/var/www/
├── viceroybali/
│   └── public_html/    → WordPress staging
├── 02production/
│   └── index.html      → Placeholder (future WordPress)
└── 03production/
    └── index.html      → Placeholder (future WordPress)
```

---

## Services

All services running:
- Nginx (path-based routing enabled)
- PHP 8.3-FPM
- MySQL/MariaDB
- Redis

---

## WordPress Configuration

- Site URL: `http://34.158.47.112/viceroybali`
- Home URL: `http://34.158.47.112/viceroybali`
- Web Root: `/var/www/viceroybali/public_html/`

---

## Nginx Configuration

Path-based routing configured in `/etc/nginx/sites-available/viceroybali`:
- Root `/` → redirects to `/viceroybali/`
- `/viceroybali/` → WordPress site
- `/02production/` → Placeholder site
- `/03production/` → Placeholder site
- `/health` → Load balancer health check

See [multisite_staging.md](setup/multisite_staging.md) for full configuration.

---

## Notes

- This instance was cloned from `viceroy-bali` on Feb 2, 2026
- External IP reserved as static on Feb 3, 2026
- WordPress URLs updated to `34.158.47.112/viceroybali`
- Cloudbeds booking widget will show "Oops, No image" on staging (domain validation issue - works after DNS switch)
- 02production and 03production are placeholders for future WordPress installs
