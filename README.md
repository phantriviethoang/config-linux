# EndeavourOS - Hướng dẫn tái tạo cấu hình tối ưu hiện tại

> **Ghi chú**: Đây là hướng dẫn để tái tạo lại cấu hình CHÍNH XÁC hiện tại của máy bạn sau khi reset/cài lại hệ thống.

---

## 📋 Tổng quan cấu hình hiện tại

**Hệ thống**: EndeavourOS (Arch-based)  
**Kernel**: 6.17.1-arch1-1 (Latest)  
**CPU**: AMD Ryzen (acpi-cpufreq driver)  
**GPU**: AMD Radeon Vega (Renoir) - amdgpu driver  
**Power Management**: TLP + cpupower + disable-boost service

**Đặc điểm**:
- ✅ CPU Governor: **schedutil** (balanced)
- ✅ Turbo Boost: **TẮT** (giảm nhiệt độ)
- ✅ Power Management: **TLP** (không dùng auto-cpufreq)
- ✅ Temperature Monitoring: **lm_sensors**
- ✅ GRUB: Tắt watchdog, loglevel=3

---

## 0️⃣ Sửa GRUB EFI (Nếu mất menu boot hoặc cài mới)

> **Lưu ý**: Chỉ làm phần này nếu GRUB bị lỗi hoặc không nhận Windows. Nếu GRUB hoạt động bình thường, bỏ qua phần này.

### Yêu cầu
- Boot vào EndeavourOS Live USB
- Biết phân vùng root của EndeavourOS (thường là `/dev/nvme0n1p6` hoặc tương tự)

### Các bước thực hiện

**1. Xác định phân vùng root**
```bash
sudo lsblk -f
```
Tìm phân vùng có `FSTYPE` = `ext4` hoặc `btrfs` (thường là `nvme0n1pX`)

**2. Mount phân vùng root**
```bash
sudo mount /dev/nvme0n1pX /mnt
```
*(Thay X bằng số đúng, ví dụ: `p6` nếu đó là root Linux)*

**3. Mount EFI vào đúng chỗ**
```bash
sudo mount /dev/nvme0n1p1 /mnt/boot/efi
```

**4. Chroot vào hệ thống**
```bash
sudo arch-chroot /mnt
```

**5. Cài lại GRUB**
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

**6. Thoát và khởi động lại**
```bash
exit
sudo umount -R /mnt
reboot
```

### ✅ Kết quả mong đợi
Sau khi khởi động lại, menu GRUB sẽ hiển thị:
- EndeavourOS
- Windows Boot Manager

---

## 1️⃣ Cài đặt packages cần thiết

```bash
# Update hệ thống
sudo pacman -Syu

# Cài các packages chính
sudo pacman -S --needed cpupower lm_sensors powertop tlp
```

---

## 2️⃣ Cấu hình CPU Governor (schedutil)

### Enable cpupower service
```bash
sudo systemctl enable cpupower.service
```

### Cấu hình governor là schedutil
```bash
# Backup file gốc (nếu có)
sudo cp /etc/default/cpupower /etc/default/cpupower.bak

# Thêm dòng governor vào cuối file
echo 'governor="schedutil"' | sudo tee -a /etc/default/cpupower
```

### Khởi động service
```bash
sudo systemctl start cpupower.service
```

### Kiểm tra
```bash
# Xem governor hiện tại
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# Kết quả phải là: schedutil

# Kiểm tra service
systemctl status cpupower.service
```

---

## 3️⃣ Tắt CPU Turbo Boost (Giảm nhiệt độ)

### Tạo systemd service tự động tắt turbo boost

```bash
# Tạo file service
sudo nano /etc/systemd/system/disable-boost.service
```

### Paste nội dung sau vào file:

```ini
[Unit]
Description=Disable CPU turbo boost
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo 0 | tee /sys/devices/system/cpu/cpufreq/boost'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

### Enable và start service

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable và start
sudo systemctl enable disable-boost.service
sudo systemctl start disable-boost.service
```

### Kiểm tra

```bash
# Xem trạng thái turbo boost (phải là 0)
cat /sys/devices/system/cpu/cpufreq/boost
# Kết quả: 0 (đã tắt)

# Kiểm tra service
systemctl status disable-boost.service
```

---

## 4️⃣ Cấu hình TLP (Power Management)

### Enable TLP service

```bash
sudo systemctl enable tlp.service
sudo systemctl start tlp.service
```

### File cấu hình TLP

**Lưu ý**: File `/etc/tlp.conf` của bạn hiện tại đang dùng **TẤT CẢ CÁC SETTINGS MẶC ĐỊNH** (tất cả đều comment `#`).

Điều này có nghĩa là:
- ✅ **KHÔNG CẦN** chỉnh sửa file `/etc/tlp.conf`
- ✅ TLP sẽ tự động áp dụng các thiết lập mặc định tối ưu
- ✅ File `/etc/tlp.d/` chỉ có template, không có custom config

**→ Bạn chỉ cần enable service là đủ!**

### Kiểm tra TLP

```bash
# Xem trạng thái TLP
sudo tlp-stat -s

# Xem power settings
sudo tlp-stat -p

# Xem battery info
sudo tlp-stat -b
```

---

## 5️⃣ Cấu hình Temperature Monitoring (lm_sensors)

### Detect và cấu hình sensors

```bash
# Chạy sensors-detect (trả lời YES cho hầu hết các câu hỏi)
sudo sensors-detect
```

**Lưu ý**: Khi được hỏi "Do you want to add these lines automatically?", chọn **YES**.

### Kiểm tra nhiệt độ

```bash
# Xem nhiệt độ hiện tại
sensors
```

**Kết quả mong đợi** (tương tự máy bạn):
```
k10temp-pci-00c3
Adapter: PCI adapter
Tctl:         +47.6°C  

amdgpu-pci-0400
Adapter: PCI adapter
edge:         +47.0°C  

thinkpad-isa-0000
Adapter: ISA adapter
fan1:        1800 RPM
fan2:        1800 RPM
CPU:          +47.0°C  
```

---

## 6️⃣ Cấu hình GRUB

**Cấu hình GRUB hiện tại của bạn**:
- Tắt watchdog (`nowatchdog`)
- Enable nvme load (`nvme_load=YES`)
- Giảm log level (`loglevel=3`)

### Chỉnh sửa GRUB

```bash
sudo nano /etc/default/grub
```

### Tìm dòng `GRUB_CMDLINE_LINUX_DEFAULT` và sửa thành:

```bash
GRUB_CMDLINE_LINUX_DEFAULT='nowatchdog nvme_load=YES loglevel=3'
```

### Update GRUB

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

**Lưu ý**: 
- `nowatchdog` = Tắt NMI watchdog (tiết kiệm pin)
- `loglevel=3` = Giảm log khi boot (boot nhanh hơn)
- `nvme_load=YES` = Load driver NVMe sớm

---

## 7️⃣ Powertop (Tùy chọn - Phân tích power)

### Chạy powertop để phân tích

```bash
# Chạy powertop interactive
sudo powertop
```

**Lưu ý**: 
- Bạn **KHÔNG** dùng auto-tune của powertop
- Chỉ dùng để **phân tích** mức tiêu thụ điện

---

## 8️⃣ Xác nhận toàn bộ cấu hình

### Script kiểm tra nhanh

```bash
#!/bin/bash
echo "=== KIỂM TRA CẤU HÌNH ==="
echo ""

echo "1. CPU Governor:"
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

echo ""
echo "2. Turbo Boost (0=tắt, 1=bật):"
cat /sys/devices/system/cpu/cpufreq/boost

echo ""
echo "3. cpupower service:"
systemctl is-enabled cpupower.service
systemctl is-active cpupower.service

echo ""
echo "4. disable-boost service:"
systemctl is-enabled disable-boost.service
systemctl is-active disable-boost.service

echo ""
echo "5. TLP service:"
systemctl is-enabled tlp.service
systemctl is-active tlp.service

echo ""
echo "6. Nhiệt độ hiện tại:"
sensors | grep -E "Tctl|edge|CPU:"

echo ""
echo "7. CPU Frequency:"
cpupower frequency-info | grep "current CPU frequency"
```

### Kết quả mong đợi:

```
=== KIỂM TRA CẤU HÌNH ===

1. CPU Governor:
schedutil

2. Turbo Boost (0=tắt, 1=bật):
0

3. cpupower service:
enabled
active

4. disable-boost service:
enabled
active

5. TLP service:
enabled
active

6. Nhiệt độ hiện tại:
Tctl:         +47.6°C  
edge:         +47.0°C  
CPU:          +47.0°C  

7. CPU Frequency:
current CPU frequency: 1.99 GHz (asserted by call to kernel)
```

---

## 9️⃣ Các lệnh hữu ích

### Xem thông tin CPU chi tiết
```bash
cpupower frequency-info
```

### Xem tất cả services liên quan power
```bash
systemctl list-unit-files | grep -E "tlp|cpu|power|boost" | grep enabled
```

### Monitor nhiệt độ real-time
```bash
watch -n 1 sensors
```

### Xem GPU driver
```bash
lspci -k | grep -A 3 -E "VGA|3D"
```

### Xem battery status (nếu laptop)
```bash
sudo tlp-stat -b
```

---

## 🔟 Lưu ý quan trọng

### ❌ KHÔNG CÀI auto-cpufreq
- Máy bạn **KHÔNG** dùng auto-cpufreq
- Đã cài nhưng **KHÔNG** enable service
- Chỉ dùng **TLP + cpupower**

### ❌ KHÔNG enable fancontrol
- Máy bạn **KHÔNG** dùng fancontrol service
- Fan điều khiển bởi BIOS/firmware (tự động)

### ✅ Config đơn giản
- TLP dùng **tất cả settings mặc định** (file `/etc/tlp.conf` không custom gì)
- cpupower chỉ set **governor="schedutil"**
- disable-boost chỉ tắt turbo boost

---

## 📊 Hiệu quả đạt được

Với cấu hình này, máy bạn:
- ✅ Nhiệt độ: ~47°C (rất mát)
- ✅ Fan: ~1800 RPM (êm)
- ✅ Governor: schedutil (balanced)
- ✅ Turbo boost: Tắt (ổn định)
- ✅ Battery life: Tốt (nhờ TLP)

---

## 🔧 Troubleshooting

### Nếu governor không đổi sang schedutil
```bash
# Kiểm tra config
cat /etc/default/cpupower

# Restart service
sudo systemctl restart cpupower.service

# Xem log
journalctl -u cpupower.service
```

### Nếu turbo boost không tắt
```bash
# Kiểm tra service
systemctl status disable-boost.service

# Restart service
sudo systemctl restart disable-boost.service

# Kiểm tra manual
echo 0 | sudo tee /sys/devices/system/cpu/cpufreq/boost
```

### Nếu TLP không hoạt động
```bash
# Xem log
sudo tlp-stat -s

# Restart
sudo systemctl restart tlp.service
```

---

## 📝 Checklist sau khi cài xong

- [ ] GRUB menu hiển thị EndeavourOS + Windows (nếu dual boot)
- [ ] cpupower service: enabled & active
- [ ] disable-boost service: enabled & active
- [ ] tlp service: enabled & active
- [ ] CPU Governor = schedutil
- [ ] Turbo Boost = 0 (tắt)
- [ ] sensors hoạt động
- [ ] Nhiệt độ < 50°C khi idle

---

**✅ Hoàn tất!** Sau khi làm xong các bước trên, máy bạn sẽ có lại cấu hình tối ưu như hiện tại.
