Project Name: GDA-COMPUTE
Project ID: gda-viceroy
Project Number: 292070531785


SSH: ssh -i /Users/rogerwoolie/Container/dotfolders/.ssh/id_ed25519_gaia root@34.158.47.112

VM Instance: gda-ce01 (formerly viceroy-bali)
GCP Storage Bucket: gs://viceroybali_bucket/

## Static IP Configuration
External IP: 34.158.47.112 (Static - gda-ce01-static)

## Project Folder Structure
```
/var/www/
├── viceroybali/
│   └── public_html/    → WordPress (http://34.158.47.112/viceroybali/)
├── 02production/
│   └── index.html      → Placeholder (http://34.158.47.112/02production/)
└── 03production/
    └── index.html      → Placeholder (http://34.158.47.112/03production/)
```

## URLs
| Site | Staging URL | Production URL |
|------|-------------|----------------|
| Viceroy Bali | http://34.158.47.112/viceroybali/ | https://www.viceroybali.com/ |
| 02 Production | http://34.158.47.112/02production/ | TBD |
| 03 Production | http://34.158.47.112/03production/ | TBD |

Load Balancer IP: 34.49.188.147
CDN: Enabled (Google Cloud CDN)
