# Viceroybali Infrastructure Comparison

## Hostinger vs GCP

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **Hosting Type** | Shared Hosting | Dedicated Cloud VM |
| **Server Location** | Unknown | Singapore (asia-southeast1) |

## Server Resources

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **CPU** | Shared | 2 vCPU (dedicated) |
| **RAM** | Shared | 8 GB (dedicated) |
| **Storage** | Shared | 50 GB SSD |
| **Resource Isolation** | No | Yes |
| **Machine Type** | N/A | e2-standard-2 |

## Web Server

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **Web Server** | LiteSpeed | Nginx 1.24.0 |
| **Control Panel** | cPanel | CLI / GCP Console |
| **HTTP/2** | Yes | Yes |
| **HTTP/3** | Yes | No |
| **Gzip Compression** | Yes | Yes |

## PHP

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **PHP Version** | 8.x | 8.3.30 |
| **Handler** | LiteSpeed SAPI | PHP-FPM (Unix Socket) |
| **Memory Limit** | Limited | 256M / 512M (admin) |
| **Max Execution Time** | Limited | 300 seconds |
| **OPcache** | Yes | Yes |

## Database

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **Database** | MySQL/MariaDB | MariaDB 10.11.15 |
| **Database Name** | viceroybali_vb2021 | viceroy_db_name |
| **Size** | ~183 MB | ~183 MB |
| **External Access** | No | No (localhost only) |

## CDN (Content Delivery Network)

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **CDN** | No | Yes |
| **CDN Provider** | N/A | Google Cloud CDN |
| **Edge Locations** | N/A | 200+ worldwide |
| **Cache Mode** | N/A | CACHE_ALL_STATIC |
| **Default TTL** | N/A | 1 hour |
| **Max TTL** | N/A | 24 hours |

## Load Balancer

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **Load Balancer** | No | Yes |
| **Type** | N/A | Global HTTP(S) LB |
| **SSL Termination** | Server-level | Load Balancer |
| **Health Checks** | No | Yes (10s interval) |
| **Auto-failover** | No | Yes |
| **DDoS Protection** | Basic | Built-in (Google scale) |

## SSL/TLS Certificate

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **SSL Certificate** | Yes | Yes |
| **Type** | Standard / Let's Encrypt | Google-managed |
| **Auto-renewal** | Manual / Auto | Automatic |
| **TLS Version** | 1.2/1.3 | 1.2/1.3 |

## Caching Layer

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **Redis** | No | Yes |
| **Redis Memory** | N/A | 256 MB |
| **Redis Eviction** | N/A | allkeys-lru |
| **Object Cache** | LiteSpeed Cache | Redis object-cache.php |
| **Page Cache Plugin** | LiteSpeed Cache | WP Rocket 3.20.1.2 |
| **Page Cache Status** | Active | **Active** |
| **CSS/JS Minification** | LiteSpeed Cache | WP Rocket |
| **Lazy Loading** | LiteSpeed Cache | WP Rocket |
| **Expected TTFB** | 800-1200ms | 200-400ms |

## Firewall

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **Firewall** | cPanel-based | GCP VPC Firewall |
| **Stateful Rules** | Unknown | Yes |
| **Default Policy** | Allow | Deny (whitelist) |
| **Custom Rules** | Limited | Full control |
| **Port 80 (HTTP)** | Open | Open |
| **Port 443 (HTTPS)** | Open | Open |
| **Port 22 (SSH)** | Open | Open |
| **Port 3306 (MySQL)** | Blocked | Blocked |
| **Port 6379 (Redis)** | N/A | Blocked |

## Nginx / Apache / LiteSpeed

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **Server Software** | LiteSpeed | Nginx |
| **Apache** | No | No |
| **Configuration Control** | Limited (.htaccess) | Full (/etc/nginx/) |
| **Security Headers** | Basic | Custom (X-Frame-Options, XSS, etc.) |
| **XML-RPC** | Enabled | Disabled |
| **Sensitive File Blocking** | Basic | Yes (.git, wp-config, .htaccess) |

## Backup & Recovery

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **Automated Backups** | Weekly (Hostinger) | Weekly VM Snapshots |
| **Database Backups** | Manual / UpdraftPlus | Daily (UpdraftPlus) |
| **Backup Storage** | Hostinger servers | Google Cloud Storage |
| **Snapshot Retention** | Unknown | 4 weeks |
| **Disaster Recovery** | Limited | Full VM restore |

## Scalability

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **Vertical Scaling** | No | Yes (resize VM) |
| **Horizontal Scaling** | No | Yes (add instances) |
| **Auto-scaling** | No | Possible (with MIG) |
| **Traffic Handling** | Limited | High capacity |

## Monitoring & Logging

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **Server Monitoring** | Basic (cPanel) | GCP Cloud Monitoring |
| **Log Access** | Limited | Full access |
| **Uptime Monitoring** | Basic | Health checks + alerts |
| **Performance Metrics** | Limited | Comprehensive |

## Security Features

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **DDoS Protection** | Basic | Google-scale |
| **Shielded VM** | No | Yes (vTPM, Integrity) |
| **File Edit Disabled** | No | Yes (DISALLOW_FILE_EDIT) |
| **Force SSL Admin** | No | Yes |
| **Malware Scanning** | Yes (.quarantine) | Manual |

## Cost Model

| Feature | Hostinger | GCP |
|---------|-----------|-----|
| **Pricing** | Flat monthly fee | Pay-as-you-go |
| **Predictability** | Fixed cost | Variable |
| **Value** | Budget-friendly | Performance-focused |

## Summary

| Category | Winner |
|----------|--------|
| **Cost** | Hostinger |
| **Performance** | GCP |
| **Scalability** | GCP |
| **Security** | GCP |
| **CDN** | GCP |
| **Caching** | GCP (Redis) |
| **Control** | GCP |
| **Ease of Use** | Hostinger |
| **DDoS Protection** | GCP |
| **Global Reach** | GCP |

## Quick Reference

### Hostinger Has:
- LiteSpeed web server
- cPanel control panel
- Shared hosting environment
- LiteSpeed Cache
- Basic firewall
- Budget-friendly pricing

### Hostinger Does NOT Have:
- Dedicated resources
- CDN
- Load balancer
- Redis cache
- Advanced firewall control
- VM snapshots
- Auto-scaling

### GCP Has:
- Dedicated VM (e2-standard-2)
- Nginx web server
- Google Cloud CDN (200+ edge locations)
- Global HTTP(S) Load Balancer
- Redis Object Cache (256MB)
- WP Rocket (page cache, minification, lazy loading)
- VPC Firewall with custom rules
- Automated weekly snapshots
- Google-managed SSL certificate
- Health checks and auto-failover
- DDoS protection at Google scale
- Shielded VM security

### GCP Does NOT Have:
- cPanel (uses CLI)
- LiteSpeed (uses Nginx)
- Fixed monthly pricing
- HTTP/3 support (currently)
