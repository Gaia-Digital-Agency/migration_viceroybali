# Database Configuration

**Project:** Viceroy Bali Migration
**Database:** MariaDB 10.11.15

---

## Database Creation

```sql
CREATE DATABASE viceroy_db_name;
CREATE USER 'viceroy_user'@'localhost' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON viceroy_db_name.* TO 'viceroy_user'@'localhost';
FLUSH PRIVILEGES;
```

## Database Access

```bash
# Login to database
mysql -u viceroy_user -p viceroy_db_name

# Password
strong_password
```

## Database Snapshot

| Field | Value |
|-------|-------|
| **Snapshot File** | /var/www/backups/viceroy_db_snapshot.sql |
| **File Size** | 183 MB |
| **Original DB Name** | viceroybali_vb2021 |
| **New DB Name** | viceroy_db_name |
| **Table Prefix** | vb21_ |
| **Total Tables** | 107 tables |

## Import Database

```bash
# Import the snapshot
mysql -u viceroy_user -p viceroy_db_name < /var/www/backups/viceroy_db_snapshot.sql

# Verify import
mysql -u viceroy_user -p viceroy_db_name -e "SHOW TABLES;"
```

## Status

- [x] Database created
- [x] User created with privileges
- [x] Snapshot transferred to /var/www/backups/
- [ ] Snapshot imported
- [ ] wp-config.php updated with new database credentials

---

**Next Step:** Import database snapshot and configure WordPress
