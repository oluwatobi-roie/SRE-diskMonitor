# MySQL Disk Monitor and Auto RESET MASTER Script

This project provides a secure, automated solution to monitor the disk usage of a specific block device (`/dev/vdb1`) and automatically execute the MySQL `RESET MASTER;` command when disk usage exceeds a set threshold (default: 95%).

> ⚠️ **Warning:** `RESET MASTER;` will delete **all binary logs** from your MySQL server. Only use this if:
> - You're **not using MySQL replication**.
> - You understand the implications of losing binary log history.

---

## 📌 Project Objectives

- 🔍 Monitor `/dev/vdb1` mounted at `/mnt/blockstorage`
- ⚙️ Automatically run `RESET MASTER;` when disk usage ≥ 95%
- 🪵 Log every run with a timestamp and status (reset or skip)
- 🔐 Securely handle MySQL credentials using `.my.cnf`
- 🕑 Run automatically using a cron job (default: every 10 minutes)

---

## 🧱 System Requirements

- Linux server (e.g., Ubuntu)
- MySQL Server installed
- Root or sudo access
- Target disk mounted at `/mnt/blockstorage` (linked to `/dev/vdb1`)

---

## 🛡️ MySQL Credential Security

To avoid hardcoding credentials in the script, use a secure MySQL client config file:

### Create `/root/.my.cnf`

```ini
[client]
user=yousqlusername
password=yousqlpassword[leaveblank if password is empty]
```

> Leave the password empty if your root MySQL user has no password.

Set appropriate permissions:

```bash
sudo chmod 600 /root/.my.cnf
```

This allows you to run MySQL commands without specifying `-u` or `-p` in the script.

---

## ⚙️ Script Setup

### 1. Create the Monitor Script

Create the script file:

```bash
sudo nano /usr/local/bin/check_vdb1_and_reset_master.sh
```

Paste the following:

```bash
#!/bin/bash

# Threshold for disk usage
THRESHOLD=95

# Target mount point to monitor
TARGET_MOUNT="/mnt/blockstorage"

# Log file path
LOG_FILE="/var/log/mysql_reset_monitor.log"

# Get current usage of target mount
USAGE=$(df -h | grep "$TARGET_MOUNT" | awk '{print $5}' | sed 's/%//')

# Get current timestamp
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

# If unable to get usage
if [ -z "$USAGE" ]; then
  echo "$TIMESTAMP - ERROR: Could not determine disk usage for $TARGET_MOUNT" >> "$LOG_FILE"
  exit 1
fi

# Check if usage exceeds threshold
if [ "$USAGE" -ge "$THRESHOLD" ]; then
  echo "$TIMESTAMP - Disk usage at ${USAGE}% on $TARGET_MOUNT. Running RESET MASTER..." >> "$LOG_FILE"
  mysql -e "RESET MASTER;"
  if [ $? -eq 0 ]; then
    echo "$TIMESTAMP - RESET MASTER executed successfully." >> "$LOG_FILE"
  else
    echo "$TIMESTAMP - ERROR: RESET MASTER command failed." >> "$LOG_FILE"
  fi
else
  echo "$TIMESTAMP - Disk usage at ${USAGE}% on $TARGET_MOUNT. No action taken." >> "$LOG_FILE"
fi
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/check_vdb1_and_reset_master.sh
```

---

## 🧭 Cron Job Setup

To automate the script every 10 minutes:

### Edit the root user's crontab:

```bash
sudo crontab -e
```

### Add this line:

```cron
*/10 * * * * /usr/local/bin/check_vdb1_and_reset_master.sh
```

---

## 📄 Log Output

All script activity is logged to:

```bash
/var/log/mysql_reset_monitor.log
```

### Example Entries:

```
2025-06-18 14:10:01 - Disk usage at 96% on /mnt/blockstorage. Running RESET MASTER...
2025-06-18 14:10:02 - RESET MASTER executed successfully.
2025-06-18 14:20:01 - Disk usage at 91% on /mnt/blockstorage. No action taken.
```

---

## 🧠 Optional Enhancements

### 🔁 Use Safer Binary Log Rotation (Recommended for Production)

Instead of deleting **all** logs, rotate logs older than a few days:

```sql
PURGE BINARY LOGS BEFORE NOW() - INTERVAL 3 DAY;
```

Replace this in your script if you prefer safer cleanup.

---

### 🔔 Email Notifications (Optional)

You can modify the script to send an email alert when `RESET MASTER` is triggered:

```bash
echo "Disk usage reached $USAGE%. RESET MASTER executed." | mail -s "MySQL Reset Triggered" your@email.com
```

---

## ✅ Summary

| Feature                  | Description                                            |
|--------------------------|--------------------------------------------------------|
| Disk Monitored           | `/mnt/blockstorage` (`/dev/vdb1`)                     |
| Threshold                | 95% disk usage                                         |
| MySQL Command            | `RESET MASTER;`                                        |
| Credentials              | Stored securely in `/root/.my.cnf`                    |
| Execution Frequency      | Every 10 minutes via cron                              |
| Logs File                | `/var/log/mysql_reset_monitor.log`                    |
| Script Location          | `/usr/local/bin/check_vdb1_and_reset_master.sh`       |

---

## 👨‍🔧 Maintainer Notes

- Monitor the log file regularly.
- Ensure `/mnt/blockstorage` is correctly mounted on reboot.
- Rotate `/var/log/mysql_reset_monitor.log` with `logrotate` if needed.
- Regularly back up important data before using destructive MySQL commands.

---

## 📬 Contact

🔗 [Connect with me on LinkedIn](https://www.linkedin.com/in/akomolafetech/)
