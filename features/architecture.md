# Viceroy Bali - System Architecture

**Domain:** www.viceroybali.com
**Platform:** Google Cloud Platform (GCP)
**CMS:** WordPress 6.7.x

---

## High-Level Architecture

```
                                    ┌─────────────────────────────────────────────────────────────┐
                                    │                         INTERNET                            │
                                    └─────────────────────────────┬───────────────────────────────┘
                                                                  │
                                                                  ▼
                                    ┌─────────────────────────────────────────────────────────────┐
                                    │                      CLOUD DNS                              │
                                    │              www.viceroybali.com                            │
                                    │              viceroybali.com                                │
                                    └─────────────────────────────┬───────────────────────────────┘
                                                                  │
                                                                  ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                         GOOGLE CLOUD PLATFORM                                                │
│                                         Project: gda-viceroy                                                 │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                    GLOBAL LOAD BALANCER                                                │  │
│  │                                    IP: 34.49.188.147                                                   │  │
│  │  ┌─────────────────────────┐    ┌─────────────────────────┐    ┌─────────────────────────────────┐   │  │
│  │  │   HTTP (Port 80)        │    │   HTTPS (Port 443)      │    │   Google-Managed SSL            │   │  │
│  │  │   ┌─────────────────┐   │    │   ┌─────────────────┐   │    │   Certificate                   │   │  │
│  │  │   │ 301 Redirect    │   │    │   │ SSL Termination │   │    │   - www.viceroybali.com         │   │  │
│  │  │   │ to HTTPS        │   │    │   │ TLS 1.2/1.3     │   │    │   - viceroybali.com             │   │  │
│  │  │   └─────────────────┘   │    │   └─────────────────┘   │    └─────────────────────────────────┘   │  │
│  │  └─────────────────────────┘    └───────────┬─────────────┘                                          │  │
│  └─────────────────────────────────────────────┼────────────────────────────────────────────────────────┘  │
│                                                │                                                            │
│                                                ▼                                                            │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                      CLOUD CDN                                                         │  │
│  │                              Cache Mode: CACHE_ALL_STATIC                                              │  │
│  │                              TTL: 3600s (1 hour)                                                       │  │
│  │  ┌────────────────────────────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │  Cached: .jpg .jpeg .png .gif .ico .css .js .pdf .webp .woff .woff2 .ttf .svg .eot .mp4       │   │  │
│  │  │  Edge Locations: Global (Singapore primary)                                                    │   │  │
│  │  └────────────────────────────────────────────────────────────────────────────────────────────────┘   │  │
│  └───────────────────────────────────────────────┬───────────────────────────────────────────────────────┘  │
│                                                  │                                                          │
│                                                  ▼                                                          │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                    BACKEND SERVICE                                                     │  │
│  │                               viceroybali-backend                                                      │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  Health Check: /health (HTTP:80)                                                                │  │  │
│  │  │  Balancing: UTILIZATION (80% max)                                                               │  │  │
│  │  │  Protocol: HTTP                                                                                 │  │  │
│  │  └─────────────────────────────────────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────┬───────────────────────────────────────────────────────┘  │
│                                                  │                                                          │
│                                                  ▼                                                          │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                              UNMANAGED INSTANCE GROUP (UMIG)                                          │  │
│  │                              viceroybali-umig                                                          │  │
│  │                              Zone: asia-southeast1-b                                                   │  │
│  └───────────────────────────────────────────────┬───────────────────────────────────────────────────────┘  │
│                                                  │                                                          │
│  ┌───────────────────────────────────────────────▼───────────────────────────────────────────────────────┐  │
│  │                                     FIREWALL RULES                                                     │  │
│  │  ┌─────────────────────────────┐  ┌─────────────────────────────┐  ┌─────────────────────────────┐   │  │
│  │  │ viceroybali-allow-lb-health │  │ viceroybali-allow-http-https│  │ default-allow-ssh           │   │  │
│  │  │ Source: 130.211.0.0/22     │  │ Source: 0.0.0.0/0           │  │ Source: 0.0.0.0/0           │   │  │
│  │  │         35.191.0.0/16      │  │ Ports: 80, 443              │  │ Port: 22                    │   │  │
│  │  │ Port: 80                   │  │ Tags: http-server           │  │                             │   │  │
│  │  └─────────────────────────────┘  └─────────────────────────────┘  └─────────────────────────────┘   │  │
│  └───────────────────────────────────────────────┬───────────────────────────────────────────────────────┘  │
│                                                  │                                                          │
│                                                  ▼                                                          │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                    COMPUTE ENGINE VM                                                   │  │
│  │                                    gda-ce01                                                            │  │
│  │                                    IP: 34.158.47.112 (Static)                                          │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  Machine: e2-standard-2 (2 vCPU / 8 GB RAM)                                                     │  │  │
│  │  │  OS: Ubuntu 24.04 LTS                                                                           │  │  │
│  │  │  Disk: 50 GB SSD (pd-balanced)                                                                  │  │  │
│  │  │  Zone: asia-southeast1-b (Singapore)                                                            │  │  │
│  │  └─────────────────────────────────────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                                                        │  │
│  │  ┌──────────────────────────────────────── APPLICATION STACK ─────────────────────────────────────┐   │  │
│  │  │                                                                                                 │   │  │
│  │  │  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐        │   │  │
│  │  │  │     NGINX       │   │    PHP-FPM      │   │     MySQL       │   │     Redis       │        │   │  │
│  │  │  │    1.24.0       │   │     8.3.6       │   │   Community     │   │    Server       │        │   │  │
│  │  │  │   Port: 80      │──▶│  Unix Socket    │──▶│   Port: 3306    │   │  Port: 6379     │        │   │  │
│  │  │  │                 │   │                 │   │                 │   │  Object Cache   │        │   │  │
│  │  │  └─────────────────┘   └─────────────────┘   └─────────────────┘   └─────────────────┘        │   │  │
│  │  │          │                     │                     │                     ▲                  │   │  │
│  │  │          │                     │                     │                     │                  │   │  │
│  │  │          ▼                     ▼                     ▼                     │                  │   │  │
│  │  │  ┌─────────────────────────────────────────────────────────────────────────┴──────────────┐   │   │  │
│  │  │  │                              WORDPRESS 6.7.x                                            │   │   │  │
│  │  │  │                         /var/www/viceroybali/public_html                                │   │   │  │
│  │  │  └─────────────────────────────────────────────────────────────────────────────────────────┘   │   │  │
│  │  │                                                                                                 │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                      CLOUD STORAGE                                                      │  │
│  │                                  gs://viceroybali_bucket/                                               │  │
│  │                                  Backup & Asset Storage                                                 │  │
│  └────────────────────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                                              │
└──────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## File System Structure

### Multi-Site Root (Staging)

```
/var/www/
├── viceroybali/
│   └── public_html/              # WordPress (http://34.158.47.112/viceroybali/)
├── 02production/
│   └── index.html                # Placeholder (http://34.158.47.112/02production/)
└── 03production/
    └── index.html                # Placeholder (http://34.158.47.112/03production/)
```

### Nginx Path-Based Routing

| Path | Document Root | Content |
|------|---------------|---------|
| `/` | Redirect | → `/viceroybali/` |
| `/viceroybali/` | `/var/www/viceroybali/public_html/` | WordPress |
| `/02production/` | `/var/www/02production/` | Placeholder |
| `/03production/` | `/var/www/03production/` | Placeholder |
| `/health` | - | Load balancer health check |

### Server Root (viceroybali)

```
/var/www/viceroybali/
├── .htpasswds/                    # Password protection configs (from cPanel)
│   ├── public_html/
│   └── cache/
├── .cpanel/                       # Legacy cPanel data (from Hostinger)
│   ├── logs/
│   ├── datastore/
│   └── caches/
└── public_html/                   # ◀── WEB ROOT
```

### WordPress Installation

```
/var/www/viceroybali/public_html/
│
├── index.php                      # WordPress entry point
├── wp-config.php                  # Configuration (DB, salts, cache)
├── wp-activate.php                # User activation
├── wp-blog-header.php             # Template loader
├── wp-comments-post.php           # Comment handler
├── wp-cron.php                    # Scheduled tasks
├── wp-links-opml.php              # OPML export
├── wp-load.php                    # Bootstrap loader
├── wp-login.php                   # Login/logout handler
├── wp-mail.php                    # Email posting
├── wp-settings.php                # Core settings
├── wp-signup.php                  # Multisite signup
├── wp-trackback.php               # Trackback handler
├── xmlrpc.php                     # XML-RPC API (blocked)
├── license.txt                    # GPL license
├── readme.html                    # WordPress readme
│
├── wp-admin/                      # Admin dashboard
│   ├── admin.php
│   ├── admin-ajax.php
│   ├── edit.php
│   ├── post.php
│   ├── media.php
│   ├── themes.php
│   ├── plugins.php
│   ├── options.php
│   └── ...
│
├── wp-includes/                   # Core WordPress files
│   ├── version.php
│   ├── functions.php
│   ├── class-wp.php
│   ├── class-wp-query.php
│   ├── rest-api/                  # REST API endpoints
│   ├── blocks/                    # Gutenberg blocks
│   ├── js/                        # Core JavaScript
│   │   ├── jquery/
│   │   ├── tinymce/
│   │   └── dist/
│   ├── css/                       # Core styles
│   └── ...
│
├── wp-content/                    # ◀── CUSTOMIZATIONS
│   │
│   ├── themes/                    # Theme files
│   │   ├── viceroybali/           # ◀── ACTIVE THEME
│   │   ├── viceroybali-git/       # Git-managed version
│   │   ├── viceroy-git/
│   │   ├── viceroy-git-2/
│   │   ├── viceroy-git-3/
│   │   ├── viceroyblock/
│   │   └── twentytwentyfive/      # Default theme
│   │
│   ├── plugins/                   # Plugins (managed via DB)
│   │
│   ├── uploads/                   # Media library
│   │   ├── 2015/
│   │   ├── 2018/
│   │   ├── 2019/
│   │   │   └── 01/ - 12/          # Monthly folders
│   │   ├── 2020/
│   │   └── 2021/
│   │
│   ├── languages/                 # Translations
│   │
│   ├── cache/                     # Cache files
│   │   ├── wprocket/
│   │   └── litespeed/
│   │
│   ├── litespeed/                 # LiteSpeed cache
│   │
│   ├── smush-webp/                # WebP conversions
│   │
│   ├── upgrade/                   # Update temp files
│   │
│   └── upgrade-temp-backup/       # Backup during updates
│
├── .well-known/                   # SSL/Domain verification
│
├── img/                           # Static images
│   ├── logo/
│   ├── blog/
│   ├── download/                  # Downloadable assets
│   └── black/
│
├── video/                         # Video files
│
├── documents/                     # PDF/Documents
│
├── 360/                           # 360° virtual tours
│
└── cgi-bin/                       # CGI scripts (legacy)
```

---

## Component Details

### 1. Load Balancer

| Property | Value |
|----------|-------|
| Type | Global External HTTP(S) Load Balancer |
| IP Address | 34.49.188.147 |
| SSL Certificate | Google-managed (auto-renewal) |
| HTTP Handling | 301 redirect to HTTPS |
| Health Check | `/health` endpoint |

### 2. Cloud CDN

| Property | Value |
|----------|-------|
| Cache Mode | CACHE_ALL_STATIC |
| Default TTL | 3600 seconds (1 hour) |
| Max TTL | 86400 seconds (24 hours) |
| Negative Caching | Enabled |

**Cached Content Types:**
- Images: `.jpg`, `.jpeg`, `.png`, `.gif`, `.webp`, `.svg`, `.ico`
- Fonts: `.woff`, `.woff2`, `.ttf`, `.eot`
- Scripts: `.js`, `.css`
- Documents: `.pdf`
- Media: `.mp4`

### 3. Compute Engine VM

| Property | Value |
|----------|-------|
| Name | gda-ce01 |
| Machine Type | e2-standard-2 |
| vCPUs | 2 |
| Memory | 8 GB |
| Disk | 50 GB SSD (pd-balanced) - 20 GB free |
| Zone | asia-southeast1-b |
| External IP | 34.158.47.112 (Static: gda-ce01-static) |
| Internal IP | 10.148.0.4 |
| Network Tags | http-server, https-server |
| Instance Group | gaiada-ce |

### 4. Application Stack

| Component | Version | Configuration |
|-----------|---------|---------------|
| Nginx | 1.24.0 | Port 80, gzip enabled |
| PHP-FPM | 8.3.30 | Unix socket, 256M memory, 50 workers |
| MariaDB | 10.11.15 | Port 3306, localhost only |
| Redis | 7.x | Port 6379, 256MB max, **ACTIVE** object cache |

### 5. Database

| Property | Value |
|----------|-------|
| Database Name | viceroy_db_name |
| User | viceroy_user |
| Table Prefix | vb21_ |
| Tables | 107 |
| Character Set | utf8mb4_unicode_520_ci |

### 6. Firewall Rules

| Rule Name | Direction | Source | Ports | Target |
|-----------|-----------|--------|-------|--------|
| viceroybali-allow-lb-health | INGRESS | 130.211.0.0/22, 35.191.0.0/16 | TCP:80 | http-server |
| viceroybali-allow-http-https | INGRESS | 0.0.0.0/0 | TCP:80,443 | http-server, https-server |
| default-allow-ssh | INGRESS | 0.0.0.0/0 | TCP:22 | All |

---

## Active WordPress Plugins

| Plugin | Purpose |
|--------|---------|
| **polylang** | Multilingual support |
| **advanced-custom-fields-pro** | Custom fields |
| **contact-form-7** | Contact forms |
| **litespeed-cache** | Caching & optimization |
| **akismet** | Spam protection |
| **updraftplus** | Backups |
| **redirection** | URL redirects |
| **instagram-feed** | Instagram integration |
| **microsoft-clarity** | Analytics |
| **simple-history** | Activity logging |
| **duplicate-page** | Content duplication |
| **duplicator** | Site migration |
| **query-monitor** | Debug/performance |
| **custom-post-type-ui** | Custom post types |
| **3d-flipbook-dflip-lite** | Digital flipbooks |
| **floating-toc** | Table of contents |
| **gutenberg-ramp** | Block editor control |
| **owm-weather** | Weather widget |
| **stripe-terminal-backend** | Payment processing |
| **wicked-folders** | Media organization |

---

## Request Flow

```
1. User Request
   │
   ▼
2. Cloud DNS Resolution
   │  www.viceroybali.com → 34.49.188.147
   │
   ▼
3. Global Load Balancer
   │  ├── HTTP (80) → 301 Redirect to HTTPS
   │  └── HTTPS (443) → SSL Termination
   │
   ▼
4. Cloud CDN Check
   │  ├── Cache HIT → Return cached content
   │  └── Cache MISS → Forward to origin
   │
   ▼
5. Backend Service
   │  Health check: /health
   │
   ▼
6. Instance Group (UMIG)
   │
   ▼
7. Firewall Rules
   │  Allow: 80, 443 from 0.0.0.0/0
   │
   ▼
8. VM (gda-ce01)
   │
   ▼
9. Nginx (Path-Based Routing)
   │  ├── /viceroybali/ → WordPress
   │  ├── /02production/ → Placeholder
   │  ├── /03production/ → Placeholder
   │  ├── Static files → Serve directly
   │  └── PHP files → FastCGI to PHP-FPM
   │
   ▼
10. PHP-FPM (8.3)
    │
    ▼
11. WordPress
    │  ├── Redis → Object cache lookup
    │  └── MySQL → Database queries
    │
    ▼
12. Response → Back through stack → User
```

---

## Third-Party Integrations

| Service | Purpose | Integration Point |
|---------|---------|-------------------|
| Stripe | Payment processing | stripe-terminal-backend plugin |
| Instagram | Social feed | instagram-feed plugin |
| Microsoft Clarity | Analytics | clarity plugin |
| External Booking System | Reservations | 3rd party API |

---

## Backup Strategy

| Type | Frequency | Retention | Storage |
|------|-----------|-----------|---------|
| VM Snapshot | Weekly (Sunday 3AM) | 4 weeks | GCP Snapshots |
| Database Backup | Daily (via UpdraftPlus) | 30 days | Cloud Storage |
| Files Backup | Daily (via UpdraftPlus) | 30 days | Cloud Storage |
| Gold Master | Post-migration | Permanent | Cloud Storage |

---

## Performance Optimizations

### Nginx
- Gzip compression (level 6)
- Static file caching (30 days)
- FastCGI buffer optimization
- Worker connections: 2048

### PHP
- OPcache enabled (128MB, can increase to 256MB)
- Memory limit: 256M (512M max)
- Max execution: 300s
- PHP-FPM pool: 50 max workers (dynamic)
- **Redis object caching: ACTIVE** ✅

### WordPress
- WP_CACHE enabled
- **Redis Object Cache: ACTIVE** (600+ keys, 54%+ hit rate)
- WP Rocket page caching
- WebP image optimization (Smush)
- Lazy loading enabled
- Object cache drop-in installed

### CDN
- Global edge caching
- Static asset caching (1 hour)
- Immutable cache headers

---

## Security Measures

| Layer | Protection |
|-------|------------|
| Load Balancer | DDoS protection, SSL/TLS |
| Firewall | IP-based access control |
| Nginx | Security headers, xmlrpc blocked |
| WordPress | DISALLOW_FILE_EDIT, FORCE_SSL_ADMIN |
| Database | localhost only, strong password |
| Files | wp-config.php restricted (640) |

---

## Monitoring Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/health` | Load balancer health check |
| `/wp-admin/` | Admin dashboard |
| `/wp-json/` | REST API |
