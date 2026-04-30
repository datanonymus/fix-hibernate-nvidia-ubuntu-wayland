[🇬🇧 Read the English version here](README.md)
# Fix Hibernate on Ubuntu 26.04 (Wayland) for NVIDIA dGPU-only Laptops

Hướng dẫn khắc phục triệt để lỗi Hibernate (`pci_pm_freeze() returns -5`) trên môi trường **Ubuntu 26.04 LTS (Wayland)** dành cho các dòng Laptop/PC chỉ sử dụng duy nhất GPU rời NVIDIA (NVIDIA-only/dGPU-only).

## 📌 Bối cảnh & Vấn đề:
Trên các dòng Laptop Gaming đời mới (ví dụ: **Lenovo LOQ 2024 - Ryzen 7 7435HS / RTX 4050 Mobile**) không tích hợp iGPU, card NVIDIA đảm nhiệm toàn bộ việc xuất hình. Khi sử dụng Ubuntu 26.04 (mặc định ép dùng Wayland):

1. **Suspend:** Thường bị treo do xung đột S0ix BIOS (vốn được code cứng tối ưu cho Windows). Lỗi này phụ thuộc vào nhà sản xuất BIOS.
2. **Hibernate:** Gặp lỗi `pci_pm_freeze() returns -5`. Nguyên nhân do GNOME Wayland không giải phóng DRM Master của GPU, khiến driver NVIDIA không thể đóng băng và xả VRAM xuống ổ cứng, dẫn đến treo cứng hệ thống (màn đen, quạt quay, đèn phím sáng).

Giải pháp này tập trung vào việc ép Wayland nhả tài nguyên GPU trước khi thực hiện tiến trình Hibernate.

---

## ⚠️ Lưu ý quan trọng:
- **Đối tượng áp dụng:** Máy chỉ có card rời NVIDIA (CPU dòng HX, hoặc máy đã tắt iGPU trong BIOS/dùng chế độ dGPU-only).
- **Không áp dụng:** Các máy chạy song song iGPU + dGPU (NVIDIA Optimus) để tránh xung đột ACPI.

---

## 🚨 CẢNH BÁO KERNEL & SỰ ĐÁNH ĐỔI (CRITICAL):
Hiện tại, bản Kernel `7.0.0` mặc định của Ubuntu 26.04 (cài từ file ISO sạch) đang xung đột nặng với driver NVIDIA Proprietary, gây lỗi đen màn hình khi Hibernate. Để giải pháp này hoạt động, bạn **BẮT BUỘC phải lùi về Kernel 6.8.x**.

⚠️ **Sự đánh đổi:** Kernel 6.8 có thể không nhận diện đủ driver (Wi-Fi, Audio) cho một số Laptop đời quá mới (cuối 2024 - 2025). Anh em hãy cân nhắc xem mình cần "Hibernate" hay cần "Hỗ trợ phần cứng mới nhất" nhé!

**🛠️ Hướng dẫn cài Kernel 6.8 cho máy cài ISO sạch:**
1. Cài đặt công cụ Mainline Kernel:
```bash
sudo add-apt-repository ppa:cappelikan/ppa -y
sudo apt update && sudo apt install mainline -y
```
2. Mở ứng dụng **"Ubuntu Mainline Kernel Installer"** trong menu.
3. Tìm và chọn phiên bản 6.8.x (khuyến nghị `6.8.12` hoặc mới nhất của nhánh 6.8) -> Bấm **"Install"**.
4. Khởi động lại máy, ở màn hình GRUB -> Chọn **"Advanced options for Ubuntu"** -> Boot vào dòng **"Linux 6.8.x-generic"**.

---

## 🛠️ Các bước thực hiện:

### Bước 1: Cài đặt NVIDIA Proprietary Driver:
Bắt buộc sử dụng driver bản đóng của NVIDIA thay vì bản Open-source (`nvidia-open`) để quản lý điện năng tốt nhất.
```bash
sudo apt-get remove --purge '^nvidia-.*-open' -y
sudo apt-get install nvidia-driver-595 -y
```
*(Lưu ý: Thay `595` bằng phiên bản driver mới nhất tương thích với máy bạn. Kiểm tra bằng lệnh `ubuntu-drivers devices`).*

### Bước 2: Khởi tạo phân vùng Swap:
Dung lượng Swap khuyến nghị phải lớn hơn RAM (Ví dụ: RAM 24GB -> Swap 32GB).
```bash
sudo swapoff -a
sudo fallocate -l 32G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### Bước 3: Cấu hình Resume tham số GRUB & Initramfs:
Lấy UUID ổ cài Ubuntu và resume_offset của file Swap:
```bash
findmnt -no UUID -T /swapfile
sudo filefrag -v /swapfile | awk '{ if($1=="0:"){print substr($4, 1, length($4)-2)} }'
```
Thêm thông số `resume=UUID=<MÃ_UUID> resume_offset=<SỐ_OFFSET>` vào 2 file sau:
1. Mở `sudo nano /etc/default/grub` (thêm vào dòng `GRUB_CMDLINE_LINUX_DEFAULT`).
2. Mở `sudo nano /etc/initramfs-tools/conf.d/resume` (thêm vào file trống).

### Bước 4: Thiết lập NVIDIA Power Management:
Ép driver bảo toàn VRAM và tắt GSP Firmware để tránh lỗi wakeup.
```bash
sudo nano /etc/modprobe.d/nvidia-power-management.conf
```
Sửa/thêm dòng sau:
```text
options nvidia NVreg_PreserveVideoMemoryAllocations=1 NVreg_TemporaryFilePath=/var/tmp NVreg_EnableGpuFirmware=0
```

### Bước 5: Systemd Workaround (Core Fix):
Ép hệ thống chuyển sang TTY6 (Console) trước khi Hibernate để Wayland giải phóng GPU, và quay lại TTY2 (GUI) khi Wakeup.

**Tạo override cho hibernate service:**
```bash
sudo mkdir -p /etc/systemd/system/nvidia-hibernate.service.d
sudo nano /etc/systemd/system/nvidia-hibernate.service.d/override.conf
```
Dán nội dung:
```ini
[Service]
ExecStartPre=+/usr/bin/chvt 6
ExecStartPre=+/bin/sleep 2
```

**Tạo override cho resume service:**
```bash
sudo mkdir -p /etc/systemd/system/nvidia-resume.service.d
sudo nano /etc/systemd/system/nvidia-resume.service.d/override.conf
```
Dán nội dung:
```ini
[Service]
ExecStartPost=+/usr/bin/chvt 2
```

### Bước 6: Kích hoạt dịch vụ và Cập nhật hệ thống:
```bash
sudo systemctl enable nvidia-suspend.service nvidia-resume.service nvidia-hibernate.service
sudo systemctl daemon-reload
sudo update-grub
sudo update-initramfs -u -k $(uname -r)
```

### Bước 7: Vô hiệu hóa ACPI Wakeup (Tùy chọn):
Chặn các thiết bị USB/Bluetooth tự động đánh thức máy khi đang ngủ đông.
```bash
sudo sh -c "for dev in \$(awk '/enabled/ {print \$1}' /proc/acpi/wakeup); do echo \$dev > /proc/acpi/wakeup; done"
```

---

## 🚀 Kết quả:
Sử dụng lệnh sau để thực hiện Hibernate an toàn (Máy sẽ chớp sang TTY đen khoảng 2s trước khi tắt hẳn):
```bash
sudo systemctl hibernate
```

## 👨‍💻 Tác giả:
- **[Nguyễn Trọng Đạt](https://github.com/datanonymus)**.
- *Giải pháp được debug và phát triển với sự hỗ trợ từ công cụ AI (Gemini).*
