# 🐧 Ubuntu Chainload Boot via USB on HP MicroServer Gen8

This guide sets up a USB stick with GRUB that chainloads into Ubuntu installed on an internal disk (like an HDD/SSD), using UUIDs to avoid issues when adding/removing drives. Assumes:

- Ubuntu is installed on an internal disk (e.g., `/dev/sdb2` with ext4)
- A USB stick (e.g., `/dev/sda`) is used as the GRUB boot device
- USB is formatted with FAT32
- We're using UUIDs to make this future-proof

---

## ✅ Step 1: Identify Devices

Run this to list all drives and partitions:

```bash
lsblk -o NAME,UUID,MOUNTPOINT,FSTYPE,SIZE
```

Identify:

- The USB drive (usually the smaller one, like 8GB or 16GB)
- The internal disk (where Ubuntu is installed, usually much larger)

Get UUIDs:

```bash
sudo blkid
```

Note down:

- USB partition UUID (e.g., `C627-9591`)
- Ubuntu root partition UUID (e.g., `44d9d8cd-323f-4cc0-855c-bdb3d7149329`)

---

## 🧹 Step 2: Prepare the USB Stick

**⚠️ WARNING: This will erase all data on the USB stick**

1. Launch GParted or use terminal:

```bash
sudo parted /dev/sdX
```

(Replace `/dev/sdX` with your USB disk. Double-check!)

2. Inside `parted`:

```
mklabel msdos
mkpart primary fat32 1MiB 100%
set 1 boot on
quit
```

3. Format the new partition:

```bash
sudo mkfs.vfat -F 32 /dev/sdX1
```

4. Mount it:

```bash
sudo mkdir -p /mnt/usb
sudo mount /dev/sdX1 /mnt/usb
```

---

## 🛠️ Step 3: Install GRUB on the USB

```bash
sudo grub-install --target=i386-pc --boot-directory=/mnt/usb/boot /dev/sdX
```

- Replace `/dev/sdX` with your USB drive (not partition)

---

## 📁 Step 4: Mount Ubuntu System Disk

Assuming Ubuntu is installed on `/dev/sdY2`:

```bash
sudo mkdir -p /mnt/root
sudo mount /dev/sdY2 /mnt/root
```

Copy kernel and initrd from internal Ubuntu:

```bash
sudo cp /mnt/root/boot/vmlinuz* /mnt/usb/boot/vmlinuz
sudo cp /mnt/root/boot/initrd.img* /mnt/usb/boot/initrd.img
```

---

## ✏️ Step 5: Create GRUB Config on USB

```bash
sudo nano /mnt/usb/boot/grub/grub.cfg
```

Paste this (update UUIDs accordingly):

```cfg
set timeout=3
set default=0

menuentry "Boot Ubuntu from internal disk" {
    insmod part_gpt
    insmod ext2
    search --no-floppy --fs-uuid --set=root 44d9d8cd-323f-4cc0-855c-bdb3d7149329
    linux /boot/vmlinuz root=UUID=44d9d8cd-323f-4cc0-855c-bdb3d7149329 ro quiet
    initrd /boot/initrd.img
}
```

---

## 💾 Step 6: Final Steps

1. Sync and unmount:

```bash
sync
sudo umount /mnt/usb
```

2. Reboot and boot from USB via BIOS boot menu

---

## 🧪 Testing

- On boot, you should see a GRUB menu
- If successful, Ubuntu should load from the internal disk

---

## 🛠️ Troubleshooting

- If it fails to boot, double-check UUIDs with `blkid`
- Try adding `insmod ext2`, `insmod vfat`, or other necessary modules in GRUB
- You can drop to a GRUB shell and run `ls` or `search --fs-uuid UUID` to debug
