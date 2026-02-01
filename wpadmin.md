# WordPress Admin Users Documentation

**Project:** Viceroy Bali
**Site:** http://34.142.200.251 (staging) / https://www.viceroybali.com (production)
**Database:** viceroy_db_name
**Date:** February 1, 2026

---

## Important Security Note

**Passwords are stored as cryptographic hashes** in WordPress for security. Actual plain-text passwords **cannot be retrieved** from the database. This document provides:
- User account information (usernames, emails, roles)
- Commands to retrieve user data from the server
- Password reset procedures
- Security best practices

---

## Retrieving User Information

### On GCP Server (SSH Required)

Connect to the GCP VM and run these commands:

```bash
# SSH into server
gcloud compute ssh viceroy-bali --zone=asia-southeast1-b

# List all WordPress users
wp user list \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root \
  --format=table \
  --fields=ID,user_login,user_email,user_registered,roles

# Get detailed user information
wp user list \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root \
  --format=json
```

### Expected Output Format

```
+----+--------------+---------------------------+---------------------+--------------+
| ID | user_login   | user_email                | user_registered     | roles        |
+----+--------------+---------------------------+---------------------+--------------+
| 1  | admin        | admin@viceroybali.com     | 2020-01-15 08:30:00 | administrator|
| 2  | azlan        | azlan@example.com         | 2020-02-20 10:15:00 | administrator|
| 3  | editor       | editor@viceroybali.com    | 2021-03-10 14:20:00 | editor       |
+----+--------------+---------------------------+---------------------+--------------+
```

### Database Query Method

```bash
# Direct database query for user information
wp db query "
  SELECT
    u.ID,
    u.user_login,
    u.user_email,
    u.user_registered,
    u.display_name
  FROM vb21_users u
  ORDER BY u.ID;
" --path=/var/www/viceroybali/public_html/ --allow-root

# Get user roles
wp db query "
  SELECT
    um.user_id,
    u.user_login,
    um.meta_value as capabilities
  FROM vb21_usermeta um
  JOIN vb21_users u ON um.user_id = u.ID
  WHERE um.meta_key = 'vb21_capabilities'
  ORDER BY um.user_id;
" --path=/var/www/viceroybali/public_html/ --allow-root
```

---

## WordPress User Roles

### Role Hierarchy

| Role | Capabilities | Typical Use |
|------|-------------|-------------|
| **Administrator** | Full site control, all capabilities | Site owner, developers |
| **Editor** | Publish/manage all posts/pages | Content managers |
| **Author** | Publish/manage own posts | Content creators |
| **Contributor** | Write posts (pending review) | Guest writers |
| **Subscriber** | Read content, manage profile | Registered visitors |

### Current User Accounts

**Note:** Run the commands above on the GCP server to populate this section with actual user data.

#### Administrator Accounts

```
User 1:
- Username: [Run wp user list to retrieve]
- Email: [Run wp user list to retrieve]
- Role: Administrator
- Registered: [Run wp user list to retrieve]
- Password: **HASHED** (cannot retrieve, see password reset section)

User 2:
- Username: [Run wp user list to retrieve]
- Email: [Run wp user list to retrieve]
- Role: Administrator
- Registered: [Run wp user list to retrieve]
- Password: **HASHED** (cannot retrieve, see password reset section)
```

#### Other Accounts

```
[Populate after running wp user list command]
```

---

## Password Management

### Why Passwords Cannot Be Retrieved

WordPress uses **bcrypt hashing** with salt:
- One-way cryptographic function
- Cannot be reversed or decrypted
- Industry-standard security practice
- Each password has unique salt

Example of hashed password in database:
```
$P$B1234567890abcdefghijklmnopqrstuv
```

**This is intentional and secure.** Never store plain-text passwords.

### Password Reset Procedures

#### Method 1: WordPress Admin Dashboard (Preferred)

1. Login to WordPress admin: http://34.142.200.251/wp-admin/
2. Navigate to: **Users → All Users**
3. Click on the user to edit
4. Scroll to "Account Management" section
5. Click **"Generate Password"** button
6. Copy the generated password or set a custom one
7. Send password securely to user (do NOT email plain-text)
8. Click **"Update User"**

#### Method 2: WP-CLI (Server Access Required)

```bash
# Generate new password for a user
wp user update USERNAME --user_pass="NEW_SECURE_PASSWORD" \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root

# Example:
wp user update admin --user_pass="StrongP@ssw0rd123!" \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root

# Generate random password
wp user update USERNAME --user_pass="$(openssl rand -base64 16)" \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root
```

#### Method 3: WordPress "Lost Password" Flow

1. Go to: http://34.142.200.251/wp-login.php
2. Click **"Lost your password?"**
3. Enter username or email
4. Check email for reset link
5. Click link and set new password

#### Method 4: Direct Database Update (Emergency Only)

```bash
# Only use if locked out of WordPress completely
# Generate password hash
wp eval 'echo wp_hash_password("YourNewPassword123!");' \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root

# Update database with hashed password
wp db query "
  UPDATE vb21_users
  SET user_pass = '[PASTE_HASH_FROM_ABOVE]'
  WHERE user_login = 'admin';
" --path=/var/www/viceroybali/public_html/ --allow-root
```

---

## User Management Commands

### Create New User

```bash
# Create administrator
wp user create newadmin admin@viceroybali.com \
  --role=administrator \
  --user_pass="SecurePassword123!" \
  --display_name="Admin Name" \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root

# Create editor
wp user create neweditor editor@viceroybali.com \
  --role=editor \
  --user_pass="SecurePassword123!" \
  --display_name="Editor Name" \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root
```

### Update User Role

```bash
# Promote user to administrator
wp user set-role USERNAME administrator \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root

# Demote user to editor
wp user set-role USERNAME editor \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root
```

### Delete User

```bash
# Delete user and reassign posts to another user
wp user delete USERNAME --reassign=OTHERUSERNAME \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root

# Example: Delete old admin, reassign posts to new admin
wp user delete oldadmin --reassign=1 \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root
```

### List User Sessions

```bash
# Check active user sessions
wp user session list USERNAME \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root

# Destroy all sessions for a user (force logout)
wp user session destroy USERNAME --all \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root
```

---

## Security Best Practices

### Password Requirements

**Minimum standards for WordPress admin accounts:**

- **Length:** At least 12 characters
- **Complexity:** Mix of uppercase, lowercase, numbers, symbols
- **Uniqueness:** Different from other accounts
- **No dictionary words:** Avoid common passwords
- **Regular rotation:** Change every 90 days for admin accounts

**Good password examples:**
```
V1c3r0y!B@l1#2026$Adm1n
GCP-W0rdPr3ss!Secure#789
St@ging$S1te-Admin!2026
```

**Bad password examples:**
```
password123        # Too simple, dictionary word
admin2026          # No symbols, predictable
viceroybali        # Related to site name
12345678           # Only numbers
```

### Two-Factor Authentication (2FA)

**Recommended plugins:**
- **Wordfence** (already installed) - Includes 2FA
- **Google Authenticator**
- **Duo Two-Factor Authentication**

**Enable 2FA for all administrator accounts:**

1. Install 2FA plugin via WP Admin or WP-CLI:
```bash
wp plugin install wordfence --activate \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root
```

2. Configure in WordPress admin:
   - Navigate to: **Wordfence → Login Security**
   - Enable 2FA for administrators
   - Scan QR code with authenticator app

### Login Security

#### Limit Login Attempts

Already configured in wp-config.php security settings.

#### Change Default Admin Username

```bash
# Never use "admin" as username
# If admin user exists, create new admin and delete old one

# Create new admin user
wp user create secureadmin admin@viceroybali.com \
  --role=administrator \
  --user_pass="[SECURE_PASSWORD]" \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root

# Reassign all admin posts to new user
wp user delete admin --reassign=secureadmin \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root
```

#### Disable File Editing

**Already configured** in wp-config.php:
```php
define('DISALLOW_FILE_EDIT', true);
```

This prevents editing PHP files from WordPress admin.

### Access Control

#### Restrict wp-admin by IP (Optional)

Edit nginx config to allow only specific IPs:

```nginx
# /etc/nginx/sites-available/viceroybali
location /wp-admin/ {
    # Allow office IP
    allow 203.0.113.0/24;
    # Allow VPN IP
    allow 198.51.100.50;
    # Deny all others
    deny all;

    # Continue normal processing
    try_files $uri $uri/ /index.php?$args;
}
```

#### Monitor Login Attempts

```bash
# Check WordPress login attempts in access log
grep "wp-login.php" /var/log/nginx/viceroybali_access.log | tail -20

# Check for failed login attempts
grep "wp-login.php" /var/log/nginx/viceroybali_access.log | grep " 302 "

# Monitor in real-time
tail -f /var/log/nginx/viceroybali_access.log | grep "wp-login"
```

---

## Emergency Access Recovery

### Scenario: Locked Out of WordPress Admin

#### Option 1: Create New Admin via WP-CLI

```bash
# SSH into GCP server
gcloud compute ssh viceroy-bali --zone=asia-southeast1-b

# Create emergency admin account
wp user create emergency emergency@viceroybali.com \
  --role=administrator \
  --user_pass="Temp!Pass#$(date +%s)" \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root

# The command will output the password
# Login with emergency/[password]
# Then create proper admin account and delete emergency user
```

#### Option 2: Promote Existing User

```bash
# List all users
wp user list --path=/var/www/viceroybali/public_html/ --allow-root

# Promote existing user to admin
wp user set-role USERNAME administrator \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root
```

#### Option 3: Direct Database Password Reset

```bash
# Generate hash for temporary password
TEMP_PASS="TempPass123!"
HASH=$(wp eval "echo wp_hash_password('$TEMP_PASS');" \
  --path=/var/www/viceroybali/public_html/ --allow-root)

# Update user password in database
wp db query "UPDATE vb21_users SET user_pass='$HASH' WHERE user_login='admin';" \
  --path=/var/www/viceroybali/public_html/ --allow-root

# Login with admin/TempPass123!
# Immediately change password after login
```

---

## Audit and Monitoring

### Check Last Login Times

WordPress doesn't track last login by default. Install plugin:

```bash
wp plugin install wp-last-login --activate \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root
```

### Review User Activity

```bash
# Check recent WordPress admin access
grep "wp-admin" /var/log/nginx/viceroybali_access.log | tail -50

# Check login page access
grep "wp-login.php" /var/log/nginx/viceroybali_access.log | tail -50

# Filter by specific user or IP
grep "wp-admin" /var/log/nginx/viceroybali_access.log | grep "203.0.113.50"
```

### Export User List for Audit

```bash
# Export to CSV
wp user list \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root \
  --format=csv \
  --fields=ID,user_login,user_email,user_registered,roles \
  > /var/www/backups/users_audit_$(date +%Y%m%d).csv

# Export to JSON
wp user list \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root \
  --format=json \
  > /var/www/backups/users_audit_$(date +%Y%m%d).json
```

---

## Backup User Data

### Database Backup (Includes Users)

```bash
# Full database backup (includes vb21_users and vb21_usermeta tables)
wp db export /var/www/backups/database_with_users_$(date +%Y%m%d).sql \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root

# Backup only user tables
wp db export /var/www/backups/users_only_$(date +%Y%m%d).sql \
  --tables=vb21_users,vb21_usermeta \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root
```

### Restore User Data

```bash
# Restore from full database backup
wp db import /var/www/backups/database_with_users_YYYYMMDD.sql \
  --path=/var/www/viceroybali/public_html/ \
  --allow-root
```

---

## Recommendations

### Immediate Actions

1. **Audit Current Users**
   ```bash
   # Run this command on GCP server
   wp user list --path=/var/www/viceroybali/public_html/ --allow-root
   ```

2. **Remove Unnecessary Accounts**
   - Delete test accounts
   - Remove ex-employee accounts
   - Keep only active administrators

3. **Reset All Admin Passwords**
   - Use strong passwords (12+ characters)
   - Store in password manager (1Password, LastPass, Bitwarden)
   - Enable 2FA for all admin accounts

4. **Document Authorized Users**
   - Update this file with current user list
   - Note each user's purpose/responsibility
   - Set password expiration policy

### Ongoing Maintenance

1. **Monthly Reviews**
   - Audit user list
   - Check login activity
   - Remove inactive accounts

2. **Quarterly Password Rotation**
   - Reset admin passwords every 90 days
   - Document rotation in change log

3. **Security Monitoring**
   - Monitor failed login attempts
   - Set up alerts for suspicious activity
   - Review Wordfence security scans

---

## Change Log

### User Account Changes

Document all user account changes here:

```
Date: 2026-02-01
Action: Initial documentation created
By: Claude Code
Notes: Created wpadmin.md with user management procedures

Date: [YYYY-MM-DD]
Action: [Created/Deleted/Updated user]
User: [username]
By: [admin name]
Notes: [reason for change]
```

---

## Quick Reference Commands

```bash
# List all users
wp user list --path=/var/www/viceroybali/public_html/ --allow-root

# Create admin user
wp user create USERNAME email@example.com --role=administrator \
  --user_pass="PASSWORD" --path=/var/www/viceroybali/public_html/ --allow-root

# Reset password
wp user update USERNAME --user_pass="NEWPASSWORD" \
  --path=/var/www/viceroybali/public_html/ --allow-root

# Delete user
wp user delete USERNAME --reassign=1 \
  --path=/var/www/viceroybali/public_html/ --allow-root

# Change role
wp user set-role USERNAME administrator \
  --path=/var/www/viceroybali/public_html/ --allow-root

# Force logout all sessions
wp user session destroy USERNAME --all \
  --path=/var/www/viceroybali/public_html/ --allow-root
```

---

## Related Documentation

- **[setup/step02.md](setup/step02.md)** - WordPress deployment and configuration
- **[features/architecture.md](features/architecture.md)** - System architecture
- **wp-config.php** - Security settings (DISALLOW_FILE_EDIT)

---

## Important Security Reminders

1. ✅ **NEVER** commit passwords to git
2. ✅ **NEVER** email passwords in plain text
3. ✅ **ALWAYS** use strong, unique passwords
4. ✅ **ALWAYS** enable 2FA for administrator accounts
5. ✅ **ALWAYS** remove inactive or unnecessary user accounts
6. ✅ **REGULARLY** audit user access and permissions

---

**Status:** Template created - populate with actual user data from GCP server
**Last Updated:** February 1, 2026
**Next Audit Due:** March 1, 2026 (30 days)
