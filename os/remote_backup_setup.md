# 🛡️ Automated Remote Backup: Immich & Nextcloud (Pull-based over SSH)

This document describes how to set up **automated daily backups** of data from a local server (`susra.suveen.me`) to a remote server using `rsync` over SSH. The backups are initiated from the **remote server** (pull model), and are scheduled using `cron`.

---

## 📦 Backup Targets

| Service   | Source (on susra)                             | Destination (on remote)        | Schedule |
| --------- | --------------------------------------------- | ------------------------------ | -------- |
| Immich    | `/mnt/storage/docker/volumes/immich/library/` | `/mnt/backup/SuSra/immich/`    | 03:45 AM |
| Nextcloud | `/mnt/storage/backup/nextcloud/`              | `/mnt/backup/SuSra/nextcloud/` | 05:00 AM |

---

## ✅ Assumptions

- SSH access to `susra.suveen.me` is already configured and passwordless
- SSH is available on port `100`
- The permissions on the remote server allow `suveen` to read/write to the backup directories. (see [nextcloud_borg_backup.md](nextcloud_borg_backup.md) section below)
- Backups are being **pulled** from the remote server
- User has sufficient permissions to access backup destinations and log paths

---

## 📁 Directory Structure

```
/usr/local/bin/
├── backup_immich.sh
└── backup_nextcloud.sh

/var/log/suveen-backups/
├── susra-immich.log
└── susra-nextcloud.log

/mnt/backup/SuSra/
├── immich/
└── nextcloud/
```

---

## 🧰 Step 1: Create Backup Scripts

### 🔹 `/usr/local/bin/backup_immich.sh`

```bash
#!/bin/bash

REMOTE_USER="suveen"
REMOTE_HOST="susra.suveen.me"
REMOTE_PORT=100

REMOTE_PATH="/mnt/storage/docker/volumes/immich/library/"
LOCAL_PATH="/mnt/backup/SuSra/immich"
LOG_FILE="/var/log/suveen-backups/susra-immich.log"

mkdir -p "$LOCAL_PATH"

rsync -avz --delete -e "ssh -p $REMOTE_PORT" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_PATH" "$LOCAL_PATH" >> "$LOG_FILE" 2>&1
```

### 🔹 `/usr/local/bin/backup_nextcloud.sh`

```bash
#!/bin/bash

REMOTE_USER="suveen"
REMOTE_HOST="susra.suveen.me"
REMOTE_PORT=100

REMOTE_PATH="/mnt/storage/backup/nextcloud/"
LOCAL_PATH="/mnt/backup/SuSra/nextcloud"
LOG_FILE="/var/log/suveen-backups/susra-nextcloud.log"

mkdir -p "$LOCAL_PATH"

rsync -avz --delete -e "ssh -p $REMOTE_PORT" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_PATH" "$LOCAL_PATH" >> "$LOG_FILE" 2>&1
```

### 🛠 Make them executable

```bash
sudo chmod +x /usr/local/bin/backup_immich.sh
sudo chmod +x /usr/local/bin/backup_nextcloud.sh
```

---

## 📂 Step 2: Prepare Logging Directory

```bash
sudo mkdir -p /var/log/suveen-backups
sudo touch /var/log/suveen-backups/susra-immich.log
sudo touch /var/log/suveen-backups/susra-nextcloud.log
sudo chown -R $(whoami):$(whoami) /var/log/suveen-backups
```

---

## 🕒 Step 3: Setup Cron Jobs

Edit crontab:

```bash
crontab -e
```

Add the following lines:

```cron
# Daily Immich backup at 03:45
45 3 * * * /usr/local/bin/backup_immich.sh

# Daily Nextcloud backup at 05:00
0 5 * * * /usr/local/bin/backup_nextcloud.sh
```

---

## 🧪 Step 4: Manual Test (Optional)

You can test manually like this:

```bash
/usr/local/bin/backup_immich.sh
/usr/local/bin/backup_nextcloud.sh
```

Then verify the log output:

```bash
tail -n 50 /var/log/suveen-backups/susra-immich.log
tail -n 50 /var/log/suveen-backups/susra-nextcloud.log
```

---

## 🔄 Step 5: (Optional) Logrotate Setup

To avoid infinitely growing log files, configure log rotation:

```bash
sudo nano /etc/logrotate.d/susra-backups
```

Paste the following:

```
/var/log/suveen-backups/susra-*.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
    create 644 suveen suveen
}
```

---

## ✅ Done!

You now have a clean and robust pull-based backup system set up with:

- Separate scripts per service
- Daily cron-based scheduling
- Centralized logging with optional rotation
- Simple maintainability and testability
