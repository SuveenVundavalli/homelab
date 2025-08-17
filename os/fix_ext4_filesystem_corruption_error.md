# Fixing `ext4-fs error: bad block bitmap checksum` on Ubuntu

When booting or using an ext4 filesystem, you may see an error like:

```
EXT4-fs error (device sdb1): ext4\_validate\_block\_bitmap:423:
comm ext4lazyinit: bg 4272: bad block bitmap checksum
```

This means the **ext4 filesystem metadata is corrupted**, specifically the **block bitmap**.  
The kernel’s background `ext4lazyinit` process hit corruption while initializing block groups.

---

## Causes

- Improper shutdown / sudden power loss
- Bad disk sectors (failing drive)
- Disk controller or RAM issues
- Rare ext4/kernel bug

---

## Step-by-Step Fix

### 1. Identify the affected device

Check logs to see which partition has errors:

```bash
dmesg | grep ext4-fs
# or
journalctl -k | grep ext4
```

Look for something like `/dev/sdb1`.

---

### 2. Unmount the filesystem

If it’s mounted, unmount it first:

```bash
sudo umount /dev/sdb1
```

---

### 3. Run `fsck` to repair

Run a full filesystem check and repair:

```bash
sudo e2fsck -f /dev/sdb1
```

To automatically fix all errors without prompting:

```bash
sudo e2fsck -f -y /dev/sdb1
```

---

### 4. Reboot and check again

After repairing, reboot and check logs:

```bash
dmesg | grep ext4
```

---

## If Errors Keep Coming Back

### Check disk health

Use SMART tools:

```bash
sudo smartctl -a /dev/sdb
```

Look for **Reallocated Sector Count** or **Pending Sectors**.
If bad → replace the drive ASAP.

### Check RAM

Run a memory test (e.g. `memtest86+`) to rule out faulty RAM.

---

## Prevent Boot Failures

If the disk is unreliable, you don’t want boot to hang because of a bad mount.
Edit `/etc/fstab` and add the `nofail` option (and optionally a timeout).

Example:

```fstab
UUID=1234-5678   /mnt/data   ext4   defaults,nofail,x-systemd.device-timeout=10s   0   2
```

- `nofail` → continue boot even if mount fails
- `x-systemd.device-timeout=10s` → don’t wait 90s if disk is missing

Find the UUID of your partition:

```bash
blkid
```

---

## TLDR

1. Unmount the partition
2. Run `e2fsck -f -y /dev/sdX#`
3. If errors keep coming back → suspect failing disk → back up immediately
4. Add `nofail` in `/etc/fstab` so boot won’t hang

---

## Example Quick Fix

```bash
sudo umount /dev/sdb1
sudo e2fsck -f -y /dev/sdb1
sudo mount /dev/sdb1 /mnt/data
```

And in `/etc/fstab`:

```fstab
UUID=1234-5678   /mnt/data   ext4   defaults,nofail,x-systemd.device-timeout=10s   0   2
```
