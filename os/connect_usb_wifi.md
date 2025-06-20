# 🛠️ Ubuntu Server WiFi Setup for Realtek USB (TP-Link T3U)

This guide walks you through setting up WiFi on a fresh Ubuntu Server install using a Realtek-based USB adapter like the TP-Link T3U, which needs custom drivers.

---

## ✅ Prerequisites

- Fresh Ubuntu Server 24.04 installed
- TP-Link T3U WiFi USB adapter (or similar Realtek 88x2bu device)
- Temporary internet connection (e.g. LAN or old driver working)

---

## 1. Install Required Packages

```bash
sudo apt update
sudo apt install -y dkms git build-essential network-manager
```

---

## 2. Clone & Install the morrownr Driver

```bash
git clone https://github.com/morrownr/88x2bu-20210702.git
cd 88x2bu-20210702
sudo ./install-driver.sh
```

This installs the correct Realtek 88x2bu driver using DKMS.

---

## 3. Plug USB Adapter into USB 3.0 Port

Use the blue USB 3.0 port on the back of the HP MicroServer Gen8.

---

## 4. Configure Netplan for NetworkManager

Edit your netplan config (example: `/etc/netplan/01-netcfg.yaml`):

```yaml
network:
  version: 2
  renderer: NetworkManager
```

Apply the config:

```bash
sudo netplan generate
sudo netplan apply
```

---

## 5. Create WiFi Connection with `nmcli`

Find your WiFi interface (likely starts with `wl`):

```bash
nmcli device
```

Then connect:

```bash
nmcli device wifi list
nmcli device wifi connect "YourSSID" password "yourpassword"
```

You should get an IP and be online:

```bash
ip a
ping 1.1.1.1
```

---

## 6. (Optional) Disable Old In-Kernel Driver

This step is not needed if the usb dongle is not installed during OS installation.

Blacklist it so it doesn’t load:

```bash
sudo tee /etc/modprobe.d/blacklist-rtw88.conf <<EOF
blacklist rtw88_8822bu
blacklist rtw88_usb
blacklist rtw88_8822b
blacklist rtw88_core
EOF

sudo update-initramfs -u
```

---

## 7. Reboot and Verify

```bash
sudo reboot
```

After reboot:

```bash
lsmod | grep 88x2bu      # should show new driver
nmcli device status      # should show WiFi connected
```

You're done! 🎉

---

## Troubleshooting

- `iw dev <interface> link` → Shows if WiFi is connected
- `journalctl -u NetworkManager -b` → Shows logs if something fails
- Use `usb 3.0` port for full 5GHz speed (USB 2.0 will bottleneck at \~100Mbps)

---

## Notes

- This setup uses `NetworkManager` instead of `networkd` because it handles WPA/WiFi better on servers
- Works well with desktop or headless setups
