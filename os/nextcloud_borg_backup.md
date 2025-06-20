# 🛠️ Making Borg Backup Readable for Rsync – Post-Backup Permission Fix

This document describes how to ensure that Borg backups created by Nextcloud (running as `root`) are readable by the regular user (`suveen`) so that they can be backed up to a remote machine using `rsync`.

## 🧾 Context

- Nextcloud is set up to run daily Borg backups at **4:00 AM**.
- Backups are stored in:
  `/mnt/storage/backup/nextcloud/borg`
- Borg creates some files that are owned by `root` and not readable by `suveen`.
- We want to `rsync` these backups **as user `suveen`**, but permission issues prevent that.

---

## ✅ Solution: Fix Permissions After Backup

We'll create a script that resets ownership and permissions on the backup folder, and run it **at 4:15 AM** via a root cron job.

---

## 🧪 Step-by-Step

### 1. Create Post-Backup Fix Script

Create a script that will reset the permissions:

```bash
sudo nano /usr/local/bin/fix-borg-perms.sh
```

Paste the following:

```bash
#!/bin/bash
chown -R suveen:suveen /mnt/storage/backup/nextcloud/borg
chmod -R u+rwX /mnt/storage/backup/nextcloud/borg
```

Save and exit.

### 2. Make the Script Executable

```bash
sudo chmod +x /usr/local/bin/fix-borg-perms.sh
```

---

### 3. Schedule It to Run After Backup

Edit root's crontab:

```bash
sudo crontab -e
```

Add the following line to run the fix 15 minutes after the backup:

```cron
15 4 * * * /usr/local/bin/fix-borg-perms.sh
```

---

## 🧪 Verification

After the cron job runs, check that the directory and files are owned by `suveen`:

```bash
ls -l /mnt/storage/backup/nextcloud/borg
```

You should see `suveen suveen` as owner\:group, and be able to `rsync` from another machine as `suveen`.

---

## 🧼 Optional Cleanup

If you only want to copy the actual repository without temporary files, you can filter them during `rsync`:

```bash
rsync -az --exclude 'lock.exclusive' suveen@yourserver:/mnt/storage/backup/nextcloud/borg /mnt/backups/nextcloud/
```

---

## ✅ Done

Now your Borg backups will be readable by your user, and `rsync` will work from any remote server as expected.

---
