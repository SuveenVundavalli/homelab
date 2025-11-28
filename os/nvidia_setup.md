# NVIDIA Driver and Docker GPU Passthrough Setup

Ubuntu Server 24.04 LTS (Noble)

This guide explains how to install proprietary NVIDIA drivers on Ubuntu Server 24.04 and enable GPU passthrough to Docker containers using the NVIDIA Container Toolkit.
It also includes a full purge procedure for fixing broken driver installs.

---

## 1. Check GPU

```
lspci | grep -i nvidia
```

You should see your GPU model, for example:

```
00:10.0 VGA compatible controller: NVIDIA Corporation TU116 [GeForce GTX 1660 SUPER]
```

---

## 2. Purge any existing NVIDIA drivers

If the system previously had failed driver installs or conflicting versions, clean everything first.

```
sudo apt remove --purge 'nvidia-*'
sudo apt autoremove -y
sudo rm -rf /usr/local/cuda*
```

Optional cleanup:

```
sudo rm -f /etc/modprobe.d/blacklist-nouveau.conf
```

Reboot after the purge:

```
sudo reboot
```

---

## 3. Install the proprietary NVIDIA driver

Ubuntu Server works best with the stable proprietary **nvidia-driver-535** package for compute workloads and Docker GPU passthrough.

Install it:

```
sudo apt update
sudo apt install nvidia-driver-535
```

Reboot:

```
sudo reboot
```

Test the driver:

```
nvidia-smi
```

If everything is installed correctly, you’ll see your GPU info and driver version.

If Secure Boot blocks the driver, the system will prompt you to enroll an MOK on reboot.

---

## 4. Install NVIDIA Container Toolkit for Docker

Update package lists:

```
sudo apt update
```

Install:

```
sudo apt install -y nvidia-container-toolkit
```

Configure Docker to use the NVIDIA runtime:

```
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Verify the runtime is installed:

```
docker info | grep -i nvidia
```

You should see something like:

```
Runtimes: nvidia runc
```

---

## 5. Test GPU passthrough inside Docker

This is the important part.

Run a CUDA container and check if it can access the GPU:

```
docker run --rm --gpus all nvidia/cuda:12.5.0-base-ubuntu22.04 nvidia-smi
```

Expected result: output identical to running `nvidia-smi` on the host.

If this works, your setup is fully complete.

---

## 6. docker-compose example

Example for Compose v2:

```yaml
services:
  myapp:
    image: nvidia/cuda:12.5.0-base-ubuntu22.04
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
```

---

## 7. Troubleshooting

### nvidia-smi fails on host

Driver didn’t load.

```
lsmod | grep nvidia
```

If empty, reinstall the driver or check Secure Boot.

### Docker container can’t access GPU

Check daemon config:

```
cat /etc/docker/daemon.json
```

Make sure `"nvidia"` exists under runtimes.

Restart Docker:

```
sudo systemctl restart docker
```

### Permission issues on /dev/nvidia0

Add your user to the docker group:

```
sudo usermod -aG docker $USER
```

Log out and back in.

---

## 8. Complete reinstall procedure

Use this if you ever need a clean reset:

```
sudo apt remove --purge 'nvidia-*'
sudo apt autoremove -y
sudo rm -rf /usr/local/cuda*
sudo apt install nvidia-driver-535
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
docker run --rm --gpus all nvidia/cuda:12.5.0-base-ubuntu22.04 nvidia-smi
```

---

## 9. Notes

• nvidia-driver-535 is currently the most stable choice for servers
• Avoid the open versions (`nvidia-driver-*-open`) for compute workloads
• No GUI tools are available on Ubuntu Server
• Docker passthrough works as long as `nvidia-smi` works on the host
