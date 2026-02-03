# Google Cloud Load Balancer Configuration

**Date:** January 31, 2026
**Project:** Viceroy Bali Migration

---

## Overview

Global HTTP(S) Load Balancer with SSL termination, HTTP-to-HTTPS redirect, and Cloud CDN integration.

**Load Balancer IP:** 34.49.188.147

---

## Architecture

```
                         Internet
                             |
                    +--------v--------+
                    |  Global LB IP   |
                    |  34.49.188.147  |
                    +--------+--------+
                             |
              +--------------+--------------+
              |                             |
    +---------v---------+         +---------v---------+
    |   HTTP (Port 80)  |         |  HTTPS (Port 443) |
    |   Redirect Rule   |         |    SSL Termination|
    +---------+---------+         +---------+---------+
              |                             |
              +-------------+---------------+
                            |
                   +--------v--------+
                   |    Cloud CDN    |
                   | (Static Cache)  |
                   +--------+--------+
                            |
                   +--------v--------+
                   |   Backend Svc   |
                   | viceroybali-umig|
                   +--------+--------+
                            |
                   +--------v--------+
                   |  viceroy-bali   |
                   |  34.158.47.112 |
                   |  Nginx + PHP    |
                   +-----------------+
```

---

## Setup Components

### 1. Health Check

Monitors backend instance health via the `/health` endpoint.

```bash
gcloud compute health-checks create http viceroybali-health-check \
  --port=80 \
  --request-path=/health \
  --check-interval=10s \
  --timeout=5s \
  --healthy-threshold=2 \
  --unhealthy-threshold=3
```

**Settings:**
- **Check Interval:** 10 seconds
- **Timeout:** 5 seconds
- **Healthy Threshold:** 2 consecutive successes
- **Unhealthy Threshold:** 3 consecutive failures

### 2. Unmanaged Instance Group (UMIG)

Container for the single VM instance.

```bash
gcloud compute instance-groups unmanaged create viceroybali-umig \
  --zone=asia-southeast1-b

gcloud compute instance-groups unmanaged add-instances viceroybali-umig \
  --zone=asia-southeast1-b \
  --instances=viceroy-bali

gcloud compute instance-groups set-named-ports viceroybali-umig \
  --zone=asia-southeast1-b \
  --named-ports=http:80
```

**Why UMIG?**
- Prevents Google from auto-scaling/deleting the VM
- Allows manual SSH access and control
- Required for Load Balancer integration

### 3. Backend Service

Routes traffic to the instance group with CDN enabled.

```bash
gcloud compute backend-services create viceroybali-backend \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=viceroybali-health-check \
  --global \
  --enable-cdn \
  --cache-mode=CACHE_ALL_STATIC \
  --default-ttl=3600

gcloud compute backend-services add-backend viceroybali-backend \
  --instance-group=viceroybali-umig \
  --instance-group-zone=asia-southeast1-b \
  --balancing-mode=UTILIZATION \
  --max-utilization=0.8 \
  --global
```

**Settings:**
- **Balancing Mode:** UTILIZATION (80% max)
- **CDN:** Enabled
- **Protocol:** HTTP (backend to origin)

### 4. URL Map

Defines routing rules for incoming requests.

```bash
gcloud compute url-maps create viceroybali-url-map \
  --default-service=viceroybali-backend
```

### 5. SSL Certificate (Google-Managed)

Automatic SSL certificate provisioning and renewal.

```bash
gcloud compute ssl-certificates create viceroybali-ssl-cert \
  --domains=www.viceroybali.com,viceroybali.com \
  --global
```

**Status:** Provisioning (will activate once DNS points to Load Balancer IP)

**Check SSL Status:**
```bash
gcloud compute ssl-certificates describe viceroybali-ssl-cert \
  --global \
  --format="get(managed.status)"
```

### 6. HTTPS Target Proxy

Handles HTTPS traffic with SSL termination.

```bash
gcloud compute target-https-proxies create viceroybali-https-proxy \
  --url-map=viceroybali-url-map \
  --ssl-certificates=viceroybali-ssl-cert \
  --global
```

### 7. HTTP-to-HTTPS Redirect

Automatically redirects all HTTP traffic to HTTPS.

```bash
gcloud compute url-maps import viceroybali-http-redirect --source=/dev/stdin --global << 'EOF'
name: viceroybali-http-redirect
defaultUrlRedirect:
  httpsRedirect: true
  redirectResponseCode: MOVED_PERMANENTLY_DEFAULT
EOF

gcloud compute target-http-proxies create viceroybali-http-proxy \
  --url-map=viceroybali-http-redirect \
  --global
```

### 8. Global Static IP Address

Reserved public IP for the Load Balancer.

```bash
gcloud compute addresses create viceroybali-ip \
  --global \
  --ip-version=IPV4

gcloud compute addresses describe viceroybali-ip \
  --global \
  --format="value(address)"
```

**Reserved IP:** 34.49.188.147

### 9. Forwarding Rules

Routes traffic from the public IP to the proxies.

```bash
# HTTPS Forwarding Rule
gcloud compute forwarding-rules create viceroybali-https-rule \
  --load-balancing-scheme=EXTERNAL \
  --network-tier=PREMIUM \
  --address=viceroybali-ip \
  --global \
  --target-https-proxy=viceroybali-https-proxy \
  --ports=443

# HTTP Forwarding Rule
gcloud compute forwarding-rules create viceroybali-http-rule \
  --load-balancing-scheme=EXTERNAL \
  --network-tier=PREMIUM \
  --address=viceroybali-ip \
  --global \
  --target-http-proxy=viceroybali-http-proxy \
  --ports=80
```

---

## Load Balancer Components Summary

| Resource | Name | Status |
|----------|------|--------|
| Health Check | viceroybali-health-check | Active |
| Instance Group | viceroybali-umig | Active |
| Backend Service | viceroybali-backend | Active (CDN Enabled) |
| URL Map | viceroybali-url-map | Active |
| HTTP Redirect | viceroybali-http-redirect | Active |
| SSL Certificate | viceroybali-ssl-cert | Provisioning |
| HTTPS Proxy | viceroybali-https-proxy | Active |
| HTTP Proxy | viceroybali-http-proxy | Active |
| Global IP | viceroybali-ip | 34.49.188.147 |
| HTTPS Forwarding | viceroybali-https-rule | Active (Port 443) |
| HTTP Forwarding | viceroybali-http-rule | Active (Port 80) |

---

## Verification

### Check Backend Health

```bash
gcloud compute backend-services get-health viceroybali-backend --global
```

**Expected Output:**
```
backend: .../instanceGroups/viceroybali-umig
status:
  healthStatus:
  - healthState: HEALTHY
    instance: .../instances/viceroy-bali
    ipAddress: 10.148.0.3
    port: 80
```

### Test Load Balancer

```bash
# Test HTTP (should redirect to HTTPS)
curl -I http://34.49.188.147

# Test HTTPS (once SSL is provisioned)
curl -I https://34.49.188.147
```

### View Load Balancer Details

```bash
gcloud compute forwarding-rules list --global
gcloud compute backend-services list --global
gcloud compute health-checks list
```

---

## Traffic Flow

1. **User Request:** Browser requests www.viceroybali.com
2. **DNS Resolution:** Resolves to 34.49.188.147
3. **Load Balancer:** Receives request at global IP
4. **SSL Termination:** HTTPS Proxy decrypts SSL traffic
5. **CDN Check:** Checks if content is cached
6. **Backend Routing:** Routes to backend service
7. **Health Check:** Verifies backend is healthy
8. **Origin Request:** Forwards to viceroy-bali VM
9. **Response:** Nginx processes request and returns response
10. **CDN Cache:** Static content cached at edge locations

---

## Notes

- **SSL Certificate:** Will auto-activate once DNS points to 34.49.188.147
- **Zero Downtime:** Can update backends without affecting traffic
- **Auto-Healing:** Unhealthy instances automatically removed from rotation
- **Global Anycast:** Single IP serves traffic from nearest location

---

**Status:** Active
**Next Step:** DNS cutover when ready for production
