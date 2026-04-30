[🇻🇳 Đọc hướng dẫn bằng Tiếng Việt tại đây](README-vi.md)
# [GUIDE] The Ultimate Fix for Hibernate on Ubuntu 26.04 (Wayland) for NVIDIA dGPU-only Laptops.

A definitive guide to fixing the `pci_pm_freeze() returns -5` error when hibernating on **Ubuntu 26.04 LTS (Wayland)** for systems running exclusively on NVIDIA discrete GPUs (dGPU-only).

## 📌 Background & The Issue:
On modern gaming laptops (e.g., **Lenovo LOQ 2024 - Ryzen 7 7435HS / RTX 4050 Mobile**) that lack an iGPU, the NVIDIA card handles all display rendering. When using Ubuntu 26.04 (which defaults to Wayland):

1. **Suspend:** Often fails/freezes due to BIOS S0ix states being hardcoded for Windows. This is a vendor-level BIOS issue.
2. **Hibernate:** Results in a `pci_pm_freeze() returns -5` error. The root cause is the GNOME Wayland compositor refusing to release the DRM Master of the GPU. Consequently, the NVIDIA driver cannot freeze and dump the VRAM to the disk, leading to a hard system lock (black screen, fans spinning, keyboard lit).

This workaround forces Wayland to drop the GPU resources right before the hibernation sequence begins.

---

## ⚠️ Important Disclaimer:
- **Target Audience:** Systems with ONLY an NVIDIA dGPU (e.g., Intel HX processors, or systems with the iGPU disabled in the BIOS/MUX switch set to dGPU).
- **Do NOT Apply:** If you are using NVIDIA Optimus (iGPU + dGPU hybrid mode), as this may cause severe ACPI conflicts.

---

## 🚨 KERNEL WARNING & THE TRADE-OFF (CRITICAL):
Currently, the default Kernel `7.0.0` on Ubuntu 26.04 (from a clean ISO install) severely conflicts with the NVIDIA Proprietary driver. When attempting to Hibernate, the screen will flash black for 2 seconds and then **permanently freeze at the lock screen**, forcing you to hard reset by holding the power button. For this fix to work, you **MUST downgrade to Kernel 6.8.x**.

⚠️ **The Trade-off:** Kernel 6.8 might lack hardware support (Wi-Fi, Audio) for some very new laptops (late 2024 - 2025). You have to weigh what you need more: "Hibernation" or "The latest hardware compatibility"!

**🛠️ How to install Kernel 6.8 for clean ISO installs:**
1. Install the Mainline Kernel tool:
```bash
sudo add-apt-repository ppa:cappelikan/ppa -y
sudo apt update && sudo apt install mainline -y
```
2. Open the **"Ubuntu Mainline Kernel Installer"** app from your menu.
3. Find and select version 6.8.x (e.g., `6.8.12` or the latest in the 6.8 branch) -> Click **"Install"**.
4. Reboot your machine. At the GRUB menu -> Select **"Advanced options for Ubuntu"** -> Boot into **"Linux 6.8.x-generic"**.

---

## 🛠️ Step-by-Step Fix:

### Step 1: Install Proprietary NVIDIA Drivers:
The default open-source kernel modules (`nvidia-open`) have power management issues. Switch to the proprietary drivers.
```bash
sudo apt-get remove --purge '^nvidia-.*-open' -y
sudo apt-get install nvidia-driver-595 -y
```
*(Note: Replace `595` with the latest stable proprietary driver for your hardware. Check via `ubuntu-drivers devices`).*

### Step 2: Create a Large Swapfile:
Your swap size must exceed your RAM (e.g., 24GB RAM -> 32GB Swap).
```bash
sudo swapoff -a
sudo fallocate -l 32G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### Step 3: Configure GRUB & Initramfs Resume Parameters:
Retrieve your root partition's UUID and the swapfile's `resume_offset`:
```bash
findmnt -no UUID -T /swapfile
sudo filefrag -v /swapfile | awk '{ if($1=="0:"){print substr($4, 1, length($4)-2)} }'
```
Add `resume=UUID=<YOUR_UUID> resume_offset=<YOUR_OFFSET>` to the following files:
1. `sudo nano /etc/default/grub` (Append to `GRUB_CMDLINE_LINUX_DEFAULT`).
2. `sudo nano /etc/initramfs-tools/conf.d/resume` (Create/Add to the file).

### Step 4: Configure NVIDIA Power Management:
Force VRAM preservation and disable GSP Firmware to prevent wakeup crashes.
```bash
sudo nano /etc/modprobe.d/nvidia-power-management.conf
```
Add or modify this line:
```text
options nvidia NVreg_PreserveVideoMemoryAllocations=1 NVreg_TemporaryFilePath=/var/tmp NVreg_EnableGpuFirmware=0
```

### Step 5: Systemd Workaround (The Core Fix):
Force the system to switch to a TTY console (TTY6) before hibernation, forcing Wayland to release the GPU, and switch back to the GUI (TTY2) upon resume.

**Create a hibernate service override:**
```bash
sudo mkdir -p /etc/systemd/system/nvidia-hibernate.service.d
sudo nano /etc/systemd/system/nvidia-hibernate.service.d/override.conf
```
Paste:
```ini
[Service]
ExecStartPre=+/usr/bin/chvt 6
ExecStartPre=+/bin/sleep 2
```

**Create a resume service override:**
```bash
sudo mkdir -p /etc/systemd/system/nvidia-resume.service.d
sudo nano /etc/systemd/system/nvidia-resume.service.d/override.conf
```
Paste:
```ini
[Service]
ExecStartPost=+/usr/bin/chvt 2
```

### Step 6: Enable Services & Update System:
```bash
sudo systemctl enable nvidia-suspend.service nvidia-resume.service nvidia-hibernate.service
sudo systemctl daemon-reload
sudo update-grub
sudo update-initramfs -u -k $(uname -r)
```

### Step 7: Disable ACPI Wakeups (Optional):
Prevent USB/Bluetooth devices from immediately waking up the laptop after hibernation.
```bash
sudo sh -c "for dev in \$(awk '/enabled/ {print \$1}' /proc/acpi/wakeup); do echo \$dev > /proc/acpi/wakeup; done"
```

---

## 🚀 The Result:
Hibernate your system safely using:
```bash
sudo systemctl hibernate
```
*(The screen will flash to a black TTY console for ~2 seconds before fully powering off).*

## 👨‍💻 Author:
- **[Nguyễn Trọng Đạt](https://github.com/datanonymus)**.
- *Solution researched and developed with analytical assistance from AI (Gemini).*
