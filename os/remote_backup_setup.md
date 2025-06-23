# 🛡️ Automated Remote Backup: Immich & Nextcloud (Pull-based over SSH)

This document describes how to set up **automated daily backups** of data from a local server (`susra.suveen.me`) to a remote server using `rsync` over SSH. The backups are initiated from the **remote server** (pull model), and are scheduled using `cron`. In addition, Immich backups are wrapped with a Borg backup system to retain 7 days of snapshot history.

---

## 📦 Backup Targets

| Service   | Source (on susra)                             | Destination (on remote)            | Schedule |
| --------- | --------------------------------------------- | ---------------------------------- | -------- |
| Immich    | `/mnt/storage/docker/volumes/immich/library/` | `/mnt/backup/SuSra/immich/` + Borg | 03:45 AM |
| Nextcloud | `/mnt/storage/backup/nextcloud/`              | `/mnt/backup/SuSra/nextcloud/`     | 05:00 AM |

---

## ✅ Assumptions

- SSH access to `susra.suveen.me` is already configured and passwordless
- SSH is available on port `100`
- The permissions on the remote server allow `suveen` to read/write to the backup directories
- Backups are being **pulled** from the remote server
- Borg is installed on the remote server (`sudo apt install borgbackup`)
- Borg repo initialized at `/mnt/backup/SuSra/immich-borg` using:

```bash
borg init --encryption=none /mnt/backup/SuSra/immich-borg
```

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
├── immich/             <- Rsynced files
├── immich-borg/        <- Borg repo with snapshots
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
BORG_REPO="/mnt/backup/SuSra/immich-borg"
LOG_FILE="/var/log/suveen-backups/susra-immich.log"

mkdir -p "$LOCAL_PATH"

# Rsync data from Susra
echo "=== $(date): Starting rsync from Susra ===" >> "$LOG_FILE"
rsync -avz --delete -e "ssh -p $REMOTE_PORT" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_PATH" "$LOCAL_PATH" >> "$LOG_FILE" 2>&1

# Create Borg snapshot
echo "=== $(date): Creating borg snapshot ===" >> "$LOG_FILE"
borg create --stats --compression lz4 \
  "$BORG_REPO::$(date +%F)" \
  "$LOCAL_PATH" >> "$LOG_FILE" 2>&1

# Prune to keep 7 daily backups
echo "=== $(date): Pruning old snapshots ===" >> "$LOG_FILE"
borg prune -v --list "$BORG_REPO" --keep-daily=7 >> "$LOG_FILE" 2>&1

echo "=== $(date): Backup complete ===" >> "$LOG_FILE"
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

```bash
/usr/local/bin/backup_immich.sh
/usr/local/bin/backup_nextcloud.sh
```

Then verify logs:

```bash
tail -n 50 /var/log/suveen-backups/susra-immich.log
tail -n 50 /var/log/suveen-backups/susra-nextcloud.log
```

---

## 🔄 Step 5: (Optional) Logrotate Setup

```bash
sudo nano /etc/logrotate.d/susra-backups
```

Paste:

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

## 🔁 Restore from Borg Backup (Immich)

To list snapshots:

```bash
borg list /mnt/backup/SuSra/immich-borg
```

To mount a snapshot:

```bash
borg mount /mnt/backup/SuSra/immich-borg::2025-06-22 /mnt/tmp-restore
```

To extract:

```bash
borg extract /mnt/backup/SuSra/immich-borg::2025-06-22
```

---

## ✅ Done!

You now have a clean and robust pull-based backup system set up with:

- Separate scripts per service
- Rsync + Borg for Immich with snapshot support
- Daily cron-based scheduling
- Centralized logging with optional rotation
- Full 7-day rollback support for Immich
