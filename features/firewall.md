# GCP Firewall Configuration

**Date:** January 31, 2026
**Project:** Viceroy Bali Migration

---

## Overview

Firewall rules to allow HTTP/HTTPS traffic and Load Balancer health checks to the viceroy-bali VM instance.

---

## Network Tags

Network tags are applied to the VM instance to target firewall rules.

```bash
gcloud compute instances add-tags viceroy-bali \
  --zone=asia-southeast1-b \
  --tags=http-server,https-server
```

**Tags Applied:**
- `http-server` - Allows HTTP traffic
- `https-server` - Allows HTTPS traffic

### Verify Tags

```bash
gcloud compute instances describe viceroy-bali \
  --zone=asia-southeast1-b \
  --format="get(tags.items)"
```

---

## Firewall Rules

### 1. Load Balancer Health Check Rule

Allows health checks from Google Cloud Load Balancer IP ranges.

```bash
gcloud compute firewall-rules create viceroybali-allow-lb-health \
  --network=default \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:80 \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=http-server
```

**Configuration:**
| Setting | Value |
|---------|-------|
| **Name** | viceroybali-allow-lb-health |
| **Network** | default |
| **Direction** | INGRESS (incoming) |
| **Priority** | 1000 |
| **Action** | ALLOW |
| **Protocol/Port** | TCP:80 |
| **Source Ranges** | 130.211.0.0/22, 35.191.0.0/16 |
| **Target Tags** | http-server |

**Why These IP Ranges?**
- `130.211.0.0/22` - Google Cloud Load Balancer health checks
- `35.191.0.0/16` - Google Cloud Load Balancer health checks

These are official Google Cloud IP ranges for health checking.

### 2. HTTP/HTTPS Web Traffic Rule

Allows public HTTP and HTTPS traffic from the internet.

```bash
gcloud compute firewall-rules create viceroybali-allow-http-https \
  --network=default \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=http-server,https-server
```

**Configuration:**
| Setting | Value |
|---------|-------|
| **Name** | viceroybali-allow-http-https |
| **Network** | default |
| **Direction** | INGRESS (incoming) |
| **Priority** | 1000 |
| **Action** | ALLOW |
| **Protocol/Port** | TCP:80, TCP:443 |
| **Source Ranges** | 0.0.0.0/0 (anywhere) |
| **Target Tags** | http-server, https-server |

---

## Firewall Rules Summary

| Rule Name | Port(s) | Source | Target Tags | Purpose |
|-----------|---------|--------|-------------|---------|
| viceroybali-allow-lb-health | 80 | 130.211.0.0/22<br>35.191.0.0/16 | http-server | Load Balancer health checks |
| viceroybali-allow-http-https | 80, 443 | 0.0.0.0/0 (internet) | http-server<br>https-server | Public web traffic |

---

## Verification

### List All Firewall Rules

```bash
gcloud compute firewall-rules list \
  --filter="name~viceroybali" \
  --format="table(name,direction,priority,sourceRanges.list():label=SRC_RANGES,allowed[].map().firewall_rule().list():label=ALLOW,targetTags.list():label=TARGET_TAGS)"
```

### Test Firewall Rules

```bash
# Test from outside - should succeed
curl -I http://34.158.47.112

# Check if port 80 is accessible
nc -zv 34.158.47.112 80

# Check if port 443 is accessible
nc -zv 34.158.47.112 443
```

### View Specific Rule Details

```bash
gcloud compute firewall-rules describe viceroybali-allow-lb-health
gcloud compute firewall-rules describe viceroybali-allow-http-https
```

---

## Security Considerations

### What's Allowed

✅ **HTTP (Port 80)** from Load Balancer health checks
✅ **HTTP (Port 80)** from internet (redirects to HTTPS)
✅ **HTTPS (Port 443)** from internet
✅ **SSH (Port 22)** - Default GCP firewall rule allows SSH

### What's Blocked

❌ **MySQL (Port 3306)** - Database not exposed to internet
❌ **Redis (Port 6379)** - Cache not exposed to internet
❌ **All other ports** - Deny by default

### Best Practices

1. **Minimal Exposure:** Only necessary ports are open
2. **Tagged Targets:** Rules apply only to specific instances
3. **Health Check Isolation:** Separate rule for Load Balancer probes
4. **Default Deny:** All other traffic implicitly denied
5. **SSH Access:** Should be restricted to specific IP ranges (not shown - use VPN or Cloud IAP)

---

## Additional Security Recommendations

### 1. Restrict SSH Access (Optional)

Currently SSH is open from anywhere (default GCP rule). Consider restricting:

```bash
gcloud compute firewall-rules create viceroybali-allow-ssh-restricted \
  --network=default \
  --direction=INGRESS \
  --priority=900 \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=YOUR_IP_ADDRESS/32 \
  --target-tags=http-server
```

Replace `YOUR_IP_ADDRESS` with your actual IP.

### 2. Use Cloud Armor (Optional)

For DDoS protection and WAF:

```bash
gcloud compute security-policies create viceroybali-security-policy \
  --description="Security policy for Viceroy Bali"

gcloud compute backend-services update viceroybali-backend \
  --security-policy=viceroybali-security-policy \
  --global
```

### 3. Enable VPC Flow Logs (Optional)

For network traffic analysis:

```bash
gcloud compute networks subnets update default \
  --region=asia-southeast1 \
  --enable-flow-logs
```

---

## Firewall Rule Management

### Update a Rule

```bash
gcloud compute firewall-rules update viceroybali-allow-http-https \
  --source-ranges=NEW_IP_RANGE
```

### Delete a Rule

```bash
gcloud compute firewall-rules delete RULE_NAME
```

### Disable a Rule (Temporarily)

```bash
gcloud compute firewall-rules update RULE_NAME --disabled
```

### Re-enable a Rule

```bash
gcloud compute firewall-rules update RULE_NAME --no-disabled
```

---

## Notes

- All firewall rules are stateful (return traffic automatically allowed)
- Priority 1000 is standard (lower number = higher priority)
- Rules are applied at the VPC network level, not instance level
- Changes take effect immediately (no restart required)

---

**Status:** Active
**Network:** default VPC
**VM Instance:** viceroy-bali (asia-southeast1-b)
