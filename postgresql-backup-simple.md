# Stop Using Cron for Database Backups: A Simple systemd Approach

## The Problem

Your PostgreSQL backup cron job has been "working" for months. Until you discover it's been creating empty files for 3 weeks because PostgreSQL was restarting during backup time.

**This happened to me. Here's the better way.**

## What Everyone Does (Cron)

```bash
# /etc/crontab
0 2 * * * postgres pg_dump mydb > /backups/backup_$(date +%Y%m%d).sql
```

**Problems:**
- Runs even if PostgreSQL is down (creates empty files)
- No error handling or retries
- Logs go nowhere (silent failures)
- No resource limits

## The systemd Solution

### 1. Simple Backup Script
```bash
#!/bin/bash
# /usr/local/bin/pg-backup.sh

set -e  # Exit on any error

DB_NAME="mydb"
BACKUP_DIR="/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/pg_${DB_NAME}_${TIMESTAMP}.sql"

echo "Starting backup: $BACKUP_FILE"
pg_dump "$DB_NAME" > "$BACKUP_FILE"
gzip "$BACKUP_FILE"

echo "Backup completed: $BACKUP_FILE.gz"

# Optional: Upload to S3
# aws s3 cp "$BACKUP_FILE.gz" s3://my-backups/

# Cleanup old backups (keep 7 days)
find "$BACKUP_DIR" -name "pg_*.sql.gz" -mtime +7 -delete
```

### 2. systemd Service
```ini
# /etc/systemd/system/pg-backup.service
[Unit]
Description=PostgreSQL Backup
After=postgresql.service
Requires=postgresql.service

[Service]
Type=oneshot
User=postgres
ExecStart=/usr/local/bin/pg-backup.sh
```

### 3. systemd Timer
```ini
# /etc/systemd/system/pg-backup.timer
[Unit]
Description=Run PostgreSQL backup daily

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

## Deploy It

```bash
# Install files
sudo cp pg-backup.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/pg-backup.sh
sudo cp pg-backup.service /etc/systemd/system/
sudo cp pg-backup.timer /etc/systemd/system/

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable pg-backup.timer
sudo systemctl start pg-backup.timer

# Check it works
sudo systemctl status pg-backup.timer
sudo systemctl list-timers pg-backup.timer
```

## Why This is Better

| Issue | Cron | systemd |
|-------|------|---------|
| **Runs when DB is down** | ✅ Creates empty files | ❌ Waits for PostgreSQL |
| **Silent failures** | ✅ No idea it failed | ❌ Shows in `systemctl status` |
| **No retries** | ✅ Wait until tomorrow | ❌ Can retry automatically |
| **Scattered logs** | ✅ Where did output go? | ❌ `journalctl -u pg-backup` |

## Monitor Your Backups

```bash
# Check backup status
sudo systemctl status pg-backup.service

# View backup logs
sudo journalctl -u pg-backup.service -f

# Test backup manually
sudo systemctl start pg-backup.service

# List recent backups
ls -la /backups/pg_*.sql.gz
```

## Add Error Handling (Optional)

If you want retries on failure:
```ini
# Add to [Service] section
Restart=on-failure
RestartSec=300
StartLimitBurst=3
```

## Real Impact

**Before (cron):** 3 weeks of empty backup files, no alerts
**After (systemd):** Immediate notification when backups fail, automatic dependency management

## The Bottom Line

- **Cron runs commands** blindly at scheduled times
- **systemd manages services** with dependencies and error handling
- **Database backups are services**, not just commands

**Use systemd. Your future self (and your data) will thank you.**

---

*This approach has prevented multiple backup disasters in production. Keep it simple, keep it reliable.*