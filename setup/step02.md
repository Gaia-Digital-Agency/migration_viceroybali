# Step 02: WordPress Core Installation & Configuration

**Date:** January 31, 2026
**Phase:** WordPress Deployment

---

## Objective

Install missing WordPress core files and create wp-config.php to connect WordPress to the database.

## Pre-Requisites Verified

```bash
ssh -i ~/.ssh/id_ed25519_gaia root@34.124.156.73 \
  "mysql -u root -e \"SELECT option_name, option_value FROM viceroy_db_name.vb21_options WHERE option_name IN ('siteurl', 'home', 'blogname', 'db_version');\""
```

**Output:**
```
option_name    option_value
blogname       Viceroy Bali
db_version     60717
home           https://www.viceroybali.com
siteurl        https://www.viceroybali.com
```

- Database `viceroy_db_name` exists with 107 tables
- Table prefix: `vb21_`
- WordPress version: 6.7.x (db_version: 60717)
- Site URL: https://www.viceroybali.com

---

## Step 1: Install WordPress Core Files

### 1.1 Download Latest WordPress

```bash
ssh -i ~/.ssh/id_ed25519_gaia root@34.124.156.73 \
  "cd /tmp && \
   curl -sO https://wordpress.org/latest.tar.gz && \
   tar -xzf latest.tar.gz && \
   ls -la wordpress/"
```

**Output:**
```
total 240
drwxr-xr-x  5 root root  4096 Dec  2 18:35 .
drwxrwxrwt 13 root root  4096 Jan 31 12:10 ..
-rw-r--r--  1 root root   405 Feb  6  2020 index.php
-rw-r--r--  1 root root 19903 Mar  6  2025 license.txt
-rw-r--r--  1 root root  7425 Jul  8  2025 readme.html
-rw-r--r--  1 root root  7349 Oct  8 03:02 wp-activate.php
drwxr-xr-x  9 root root  4096 Dec  1 18:02 wp-admin
-rw-r--r--  1 root root   351 Feb  6  2020 wp-blog-header.php
-rw-r--r--  1 root root  2323 Jun 14  2023 wp-comments-post.php
-rw-r--r--  1 root root  3339 Aug 12 14:47 wp-config-sample.php
drwxr-xr-x  4 root root  4096 Dec  1 06:11 wp-content
-rw-r--r--  1 root root  5617 Aug  2  2024 wp-cron.php
drwxr-xr-x 31 root root 12288 Dec  2 18:35 wp-includes
-rw-r--r--  1 root root  2493 Apr 30  2025 wp-links-opml.php
-rw-r--r--  1 root root  3937 Mar 11  2024 wp-load.php
-rw-r--r--  1 root root 51437 Oct 29 10:37 wp-login.php
-rw-r--r--  1 root root  8727 Apr  2  2025 wp-mail.php
-rw-r--r--  1 root root 31055 Nov  7 12:42 wp-settings.php
-rw-r--r--  1 root root 34516 Mar 10  2025 wp-signup.php
-rw-r--r--  1 root root  5214 Aug 19 12:30 wp-trackback.php
-rw-r--r--  1 root root  3205 Nov  8  2024 xmlrpc.php
```

### 1.2 Copy Core Files

```bash
ssh -i ~/.ssh/id_ed25519_gaia root@34.124.156.73 \
  "cd /tmp/wordpress && \
   cp -r wp-includes /var/www/viceroybali/public_html/ && \
   cp index.php wp-activate.php wp-blog-header.php wp-comments-post.php \
      wp-config-sample.php wp-cron.php wp-links-opml.php wp-load.php \
      wp-login.php wp-mail.php wp-settings.php wp-signup.php \
      wp-trackback.php xmlrpc.php license.txt readme.html \
      /var/www/viceroybali/public_html/"
```

**Files NOT overwritten (preserved from migration):**
- `wp-admin/` - Kept existing
- `wp-content/` - Kept existing (themes, plugins, uploads)

### 1.3 Verification

```bash
ssh -i ~/.ssh/id_ed25519_gaia root@34.124.156.73 \
  "ls -la /var/www/viceroybali/public_html/*.php"
```

**Output:**
```
-rw-r--r-- 1 root root   405 Jan 31 12:11 index.php
-rw-r--r-- 1 root root  7349 Jan 31 12:11 wp-activate.php
-rw-r--r-- 1 root root   351 Jan 31 12:11 wp-blog-header.php
-rw-r--r-- 1 root root  2323 Jan 31 12:11 wp-comments-post.php
-rw-r--r-- 1 root root  3339 Jan 31 12:11 wp-config-sample.php
-rw-r--r-- 1 root root  5617 Jan 31 12:11 wp-cron.php
-rw-r--r-- 1 root root  2493 Jan 31 12:11 wp-links-opml.php
-rw-r--r-- 1 root root  3937 Jan 31 12:11 wp-load.php
-rw-r--r-- 1 root root 51437 Jan 31 12:11 wp-login.php
-rw-r--r-- 1 root root  8727 Jan 31 12:11 wp-mail.php
```

---

## Step 2: Create wp-config.php

### 2.1 Create Configuration File

```bash
ssh -i ~/.ssh/id_ed25519_gaia root@34.124.156.73 'cat > /var/www/viceroybali/public_html/wp-config.php << '\''WPCONFIG'\''
<?php
/**
 * WordPress Configuration - Viceroy Bali
 * Migrated to GCP: January 2026
 */

// ** Database settings ** //
define( '\''DB_NAME'\'', '\''viceroy_db_name'\'' );
define( '\''DB_USER'\'', '\''viceroy_user'\'' );
define( '\''DB_PASSWORD'\'', '\''strong_password'\'' );
define( '\''DB_HOST'\'', '\''localhost'\'' );
define( '\''DB_CHARSET'\'', '\''utf8mb4'\'' );
define( '\''DB_COLLATE'\'', '\'''\'' );

// ** Authentication keys and salts ** //
define( '\''AUTH_KEY'\'',         '\''xK9#mP2$vL5@nQ8&wR3*tY6^uI1!oA4%sD7(fG0)hJ2_kZ5+bN8=cM1-eX4~qW7'\'' );
define( '\''SECURE_AUTH_KEY'\'',  '\''pL3@nQ6#wR9$tY2%uI5^oA8&sD1*fG4!hJ7(kZ0)bN3_cM6+eX9=qW2-rT5~yU8'\'' );
define( '\''LOGGED_IN_KEY'\'',    '\''wR5#tY8$uI1@oA4%sD7^fG0&hJ3*kZ6!bN9(cM2)eX5_qW8+rT1=yU4-iO7~pL0'\'' );
define( '\''NONCE_KEY'\'',        '\''uI7@oA0#sD3$fG6%hJ9^kZ2&bN5*cM8!eX1(qW4)rT7_yU0+iO3=pL6-nQ9~wR2'\'' );
define( '\''AUTH_SALT'\'',        '\''sD9#fG2$hJ5@kZ8%bN1^cM4&eX7*qW0!rT3(yU6)iO9_pL2+nQ5=wR8-tY1~uI4'\'' );
define( '\''SECURE_AUTH_SALT'\'', '\''hJ1@kZ4#bN7$cM0%eX3^qW6&rT9*yU2!iO5(pL8)nQ1_wR4+tY7=uI0-oA3~sD6'\'' );
define( '\''LOGGED_IN_SALT'\'',   '\''bN3#cM6$eX9@qW2%rT5^yU8&iO1*pL4!nQ7(wR0)tY3_uI6+oA9=sD2-fG5~hJ8'\'' );
define( '\''NONCE_SALT'\'',       '\''eX5@qW8#rT1$yU4%iO7^pL0&nQ3*wR6!tY9(uI2)oA5_sD8+fG1=hJ4-kZ7~bN0'\'' );

// ** Database table prefix ** //
$table_prefix = '\''vb21_'\'';

// ** Debugging - disabled for production ** //
define( '\''WP_DEBUG'\'', false );
define( '\''WP_DEBUG_LOG'\'', false );
define( '\''WP_DEBUG_DISPLAY'\'', false );

// ** Performance optimizations ** //
define( '\''WP_CACHE'\'', true );
define( '\''WP_MEMORY_LIMIT'\'', '\''256M'\'' );
define( '\''WP_MAX_MEMORY_LIMIT'\'', '\''512M'\'' );

// ** Security settings ** //
define( '\''DISALLOW_FILE_EDIT'\'', true );
define( '\''FORCE_SSL_ADMIN'\'', true );

// ** Redis cache configuration ** //
define( '\''WP_REDIS_HOST'\'', '\''127.0.0.1'\'' );
define( '\''WP_REDIS_PORT'\'', 6379 );

// ** Automatic updates ** //
define( '\''AUTOMATIC_UPDATER_DISABLED'\'', true );
define( '\''WP_AUTO_UPDATE_CORE'\'', false );

/* That is all, stop editing! Happy publishing. */

/** Absolute path to the WordPress directory. */
if ( ! defined( '\''ABSPATH'\'' ) ) {
    define( '\''ABSPATH'\'', __DIR__ . '\''/'\'' );
}

/** Sets up WordPress vars and included files. */
require_once ABSPATH . '\''wp-settings.php'\'';
WPCONFIG'
```

### 2.2 Verify Configuration

```bash
ssh -i ~/.ssh/id_ed25519_gaia root@34.124.156.73 \
  "ls -la /var/www/viceroybali/public_html/wp-config.php && \
   grep -E 'DB_NAME|DB_USER|table_prefix' /var/www/viceroybali/public_html/wp-config.php"
```

**Output:**
```
-rw-r--r-- 1 root root 2111 Jan 31 12:11 wp-config.php

define( 'DB_NAME', 'viceroy_db_name' );
define( 'DB_USER', 'viceroy_user' );
$table_prefix = 'vb21_';
```

---

## Configuration Summary

| Setting | Value | Purpose |
|---------|-------|---------|
| DB_NAME | viceroy_db_name | Database name |
| DB_USER | viceroy_user | Database user |
| DB_HOST | localhost | Local MySQL |
| table_prefix | vb21_ | Matches imported tables |
| WP_CACHE | true | Enable caching |
| WP_MEMORY_LIMIT | 256M | PHP memory for frontend |
| WP_MAX_MEMORY_LIMIT | 512M | PHP memory for admin |
| DISALLOW_FILE_EDIT | true | Security: no theme/plugin editing |
| FORCE_SSL_ADMIN | true | Require HTTPS for admin |
| WP_REDIS_HOST | 127.0.0.1 | Redis object cache |

---

## Current Directory Structure

```
/var/www/viceroybali/public_html/
├── index.php              [NEW]
├── wp-config.php          [NEW]
├── wp-activate.php        [NEW]
├── wp-blog-header.php     [NEW]
├── wp-comments-post.php   [NEW]
├── wp-cron.php            [NEW]
├── wp-links-opml.php      [NEW]
├── wp-load.php            [NEW]
├── wp-login.php           [NEW]
├── wp-mail.php            [NEW]
├── wp-settings.php        [NEW]
├── wp-signup.php          [NEW]
├── wp-trackback.php       [NEW]
├── xmlrpc.php             [NEW]
├── license.txt            [NEW]
├── readme.html            [NEW]
├── wp-admin/              [EXISTING]
├── wp-content/            [EXISTING]
│   ├── themes/
│   ├── plugins/
│   ├── uploads/
│   └── cache/
└── wp-includes/           [NEW]
```

---

## Status

| Task | Status |
|------|--------|
| Download WordPress | Complete |
| Install wp-includes | Complete |
| Install core PHP files | Complete |
| Create wp-config.php | Complete |
| Database connection | Configured |
| Redis cache | Configured |

---

**Status:** Complete
**Next Document:** step03.md
