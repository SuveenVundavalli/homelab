# Formatting and Mounting a Drive on Ubuntu Server (CLI)

This guide shows how to format and mount a new drive **without partitions** using the full disk device (e.g., `/dev/sdb`).

---

## 1. Identify the target drive

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
```

Find your drive, e.g. `/dev/sdb`.
⚠️ Double-check to avoid wiping the wrong disk.

---

## 2. Unmount if mounted

```bash
sudo umount /dev/sdb
```

---

## 3. Wipe old filesystem signatures

```bash
sudo wipefs -a /dev/sdb
```

(Optional deeper wipe for first 100MB):

```bash
sudo dd if=/dev/zero of=/dev/sdb bs=1M count=100
```

---

## 4. Format the full drive

Example with **ext4**:

```bash
sudo mkfs.ext4 -L mydata /dev/sdb
```

Other options:

- **XFS**: `sudo mkfs.xfs -L mydata /dev/sdb`
- **Btrfs**: `sudo mkfs.btrfs -L mydata /dev/sdb`

---

## 5. Create mount point

```bash
sudo mkdir -p /mnt/mydata
```

---

## 6. Mount the drive

```bash
sudo mount /dev/sdb /mnt/mydata
```

Verify:

```bash
df -hT /mnt/mydata
```

---

## 7. Get UUID

```bash
blkid /dev/sdb
```

Example output:

```
/dev/sdb: UUID="abcd-1234" TYPE="ext4" LABEL="mydata"
```

---

## 8. Make mount permanent

Edit `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Add a line:

```
UUID=abcd-1234   /mnt/mydata   ext4   defaults   0   2
```

---

## 9. Reload mounts

```bash
sudo mount -a
```

Check again:

```bash
df -hT /mnt/mydata
```

---

✅ That’s it. The drive will auto-mount on every boot.
