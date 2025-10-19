# 🧠 Raspberry Pi 5 – PoE + NVMe Home Lab Node

This project documents the setup of a **single-node Raspberry Pi 5** as a prototype for a **future multi-node PoE-powered mini cluster**, used for home lab learning, testing, and development.  
The setup includes NVMe SSD boot, RTC battery configuration, and remote console access via JetKVM.

---

## 🧾 Bill of Materials (BoM)

| Item | Description | Qty | Unit Price | Total |
|------|-------------|-----|-------------|--------|
| **Raspberry Pi 5 (8 GB)** | Main compute board | 1 | £76.80 | £76.80 |
| **RTC Battery for Raspberry Pi 5** | Real-Time Clock backup battery | 1 | £4.80 | £4.80 |
| **Raspberry Pi SSD (512 GB)** | Official Raspberry Pi M.2 SSD | 1 | £43.20 | £43.20 |
| **RP2350 Zero Mini Development Board** | Waveshare RP2350-based microcontroller board | 2 | £4.60 | £9.20 |
| **Raspberry Pi 5 PCIe → M.2 with PoE HAT+ (B)** | High-speed NVMe adapter + PoE support (IceCrab / AliExpress) | 1 | £18.21 | £18.21 |

**Shipping:** £7.40 [ HAT+ free shipping ] 
**VAT (20%):** £23.56  
**Grand Total:** **£159.61 GBP**  


---

### 📦 Suppliers
- **The Pi Hut (UK)** — [thepihut.com](https://thepihut.com)  
- **IceCrab Store (China)** — via [AliExpress](https://www.aliexpress.com)  

---

## 🔧 Hardware Setup Overview  

I’m starting with a **single-node** setup for initial testing before expanding to a **mini cluster**.  

- **Power via PoE:** Using an existing **Ubiquiti USW Pro Max 16 PoE** switch to power the Pi 5 through the PoE HAT.  
- **Storage:** NVMe SSD connected via the **Waveshare PoE M.2 HAT (B)** adapter.  
- **RTC:** Backup battery maintains system time and enables wake-up from halt.  
- **Cluster goal:** Validate NVMe boot, firmware stability, and PoE operation before scaling to multiple Pi 5 nodes.

**Documentation Links:**
- [Waveshare PoE M.2 HAT (B) NVMe SSD boot guide](https://www.waveshare.com/wiki/PoE_M.2_HAT+_(B)#NVMe_SSD_boot)  
- [Raspberry Pi docs – Enable battery charging](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#enable-battery-charging)  

---

## 🌐 Network & Hardware Diagram
```text
          ┌────────────────────────────┐
          │     JetKVM Console         │
          │  (Keyboard / HDMI / Web)   │
          └────────────┬───────────────┘
                       │ USB2 + HDMI
                       │
               ┌───────▼────────┐
               │ Raspberry Pi 5 │
               │  + PoE M.2 HAT │
               │  + NVMe SSD    │
               │  + RTC Battery │
               └───────┬────────┘
                       │ PoE (Ethernet)
                       │
         ┌─────────────▼──────────────┐
         │ Ubiquiti USW Pro Max 16 PoE│
         │  (Power + Data Backbone)   │
         └─────────────┬──────────────┘
                       │
                       ▼
             🌐 Home Network / Router
```
---

## ⚡ Power Consumption Summary
Measurements taken from the **Ubiquiti USW Pro Max 16 PoE** switch PoE monitoring interface:

| State | Average Power Draw | Notes |
|--------|--------------------|--------|
| **Idle / Light Load** | ~6 W | Normal operation at idle, headless or SSH access. |
| **High Load / Boot / Update** | ~9.5 W | Peak observed during firmware update and NVMe access. |
| **Shutdown / Standby (PoE Board only)** | ~1.0 W | Power consumption when Raspberry Pi 5 is halted but PoE HAT remains powered. |

> Total daily PoE energy usage for single-node setup: **≈137 Wh**, based on continuous 24-hour operation.

---

## 🧰 Setup & Installation Notes

### 🖥️ Console Access via JetKVM
For setup, I used a **JetKVM** device for full remote access:
- **USB 2.0** → Keyboard + Mouse emulation  
- **Micro-HDMI → HDMI adapter** → Display  
- **Web interface:** [https://app.jetkvm.com](https://app.jetkvm.com)

> ⚠️ Tip:  
> Full keyboard + mouse emulation caused a pre-boot lockup.  
> Switching JetKVM to **keyboard-only mode** fixed the issue.

---

### 💾 OS Imaging & Firmware Preparation
1. Used **Raspberry Pi Imager** to write  
   → **Debian GNU/Linux 13 (Trixie) ARM64 Lite** to an **SD card**.  
2. Booted from the SD card and ran updates:
   ```bash
   sudo apt update && sudo apt full-upgrade -y
   sudo rpi-update
   ```
3. Updated the **Pi 5 firmware** to the latest stable release.

---

### ⚙️ PCIe & NVMe Configuration
When testing the **Waveshare PoE M.2 HAT (B)**:
- **PCIe Gen 3** caused instability and incomplete NVMe detection.  
- Switching to **PCIe Gen 2** resolved it and the NVMe was detected successfully.

Add this line to `/boot/firmware/config.txt`:
```bash
dtparam=pciex1
```

---

### 🚀 Migrating OS from SD Card to NVMe
Once NVMe was stable, I cloned the OS to the NVMe drive.

**Test Image and Software Tools used:**
- **Raspberry Pi Imager CLI (v2.0.0-rc3)**  
  → [Raspberry_Pi_Imager-2.0.0-cli-aarch64.AppImage](https://github.com/raspberrypi/rpi-imager/releases/download/v2.0.0-rc3/Raspberry_Pi_Imager-2.0.0-cli-aarch64.AppImage)
- **Raspberry Pi OS Lite (ARM64)** (2025-10-02):  
  [raspios_lite_arm64-2025-10-02](https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2025-10-02/)

**Example command:**
```bash
wget https://github.com/raspberrypi/rpi-imager/releases/download/v2.0.0-rc3/Raspberry_Pi_Imager-2.0.0-cli-aarch64.AppImage
wget https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2025-10-02/2025-10-01-raspios-trixie-arm64-lite.img.xz
wget https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2025-10-02/2025-10-01-raspios-trixie-arm64-lite.img.xz.sha256
sha256sum -c 2025-10-01-raspios-trixie-arm64-lite.img.xz.sha256 
2025-10-01-raspios-trixie-arm64-lite.img.xz: OK

chmod +x Raspberry_Pi_Imager-2.0.0-cli-aarch64.AppImage
sudo ./Raspberry_Pi_Imager-2.0.0-cli-aarch64.AppImage ./2025-10-01-raspios-trixie-arm64-lite.img.xz /dev/nvme0n1
sudo ./Raspberry_Pi_Imager-2.0.0-cli-aarch64.AppImage ./2025-10-01-raspios-trixie-arm64-lite.img.xz /dev/nvme0n1 --debug
Loading cache settings:
  Caching enabled: true
  Last file name: ""
  Uncompressed hash (extract_sha256): "(empty)"
  Compressed hash (image_download_sha256): "(empty)"
CacheManager initialized with background thread
Starting background cache operations
Starting background drive list polling
Background: Checking disk space for cache operations
Background: Cache directory: "/root/.cache/Raspberry Pi/Raspberry Pi Imager"
Background: Available space: 10 GiB
Device selection changed to: "/dev/nvme0n1"
Disk space check complete: 10 GiB available in "/root/.cache/Raspberry Pi/Raspberry Pi Imager"
Parsed .xz file. Uncompressed size: 2927624192
Detected total system memory: 8059 MB on "Linux"
Adaptive sync configuration: "Medium memory (8059MB)" - Sync interval: 100 MB - Time interval: 5000 ms - Platform: "Linux"
Optimal write buffer size: 4096 KB for 8059 MB system
Using buffer size: 4194304 bytes with page size: 16384 bytes
Optimal write buffer size: 4096 KB for 8059 MB system
Unmounting: "/dev/nvme0n1"
Try to perform TRIM/DISCARD on device     
BLKDISCARD successful. Discarding took 9 seconds
Zeroing out first and last MB of drive
Done zero'ing out start and end of drive. Took 0 seconds
File can be handled by libarchive as archive format
Started progress updates after successful drive opening
_writeFile: captured first block ( 4194304 ) and advanced file offset via seek
Performing periodic sync at 4194304 bytes written ( 4194304 bytes since last sync, 9741 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 109051904 bytes written ( 104857600 bytes since last sync, 1812 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 213909504 bytes written ( 104857600 bytes since last sync, 459 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 318767104 bytes written ( 104857600 bytes since last sync, 352 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 423624704 bytes written ( 104857600 bytes since last sync, 354 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 528482304 bytes written ( 104857600 bytes since last sync, 351 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 633339904 bytes written ( 104857600 bytes since last sync, 1133 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 738197504 bytes written ( 104857600 bytes since last sync, 1526 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 843055104 bytes written ( 104857600 bytes since last sync, 1071 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 947912704 bytes written ( 104857600 bytes since last sync, 702 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 1052770304 bytes written ( 104857600 bytes since last sync, 360 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 1157627904 bytes written ( 104857600 bytes since last sync, 820 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 1262485504 bytes written ( 104857600 bytes since last sync, 1324 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 1367343104 bytes written ( 104857600 bytes since last sync, 1319 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 1472200704 bytes written ( 104857600 bytes since last sync, 1773 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 1577058304 bytes written ( 104857600 bytes since last sync, 1720 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 1681915904 bytes written ( 104857600 bytes since last sync, 732 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 1786773504 bytes written ( 104857600 bytes since last sync, 1613 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 1891631104 bytes written ( 104857600 bytes since last sync, 1460 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 1996488704 bytes written ( 104857600 bytes since last sync, 1075 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 2101346304 bytes written ( 104857600 bytes since last sync, 1840 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 2206203904 bytes written ( 104857600 bytes since last sync, 2470 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 2311061504 bytes written ( 104857600 bytes since last sync, 1518 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 2415919104 bytes written ( 104857600 bytes since last sync, 1161 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 2520776704 bytes written ( 104857600 bytes since last sync, 1794 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 2625634304 bytes written ( 104857600 bytes since last sync, 1994 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 2730491904 bytes written ( 104857600 bytes since last sync, 1632 ms elapsed) on "Linux"
Periodic sync completed successfully
Performing periodic sync at 2835349504 bytes written ( 104857600 bytes since last sync, 1276 ms elapsed) on "Linux"
Periodic sync completed successfully
Hash of uncompressed image: "687005a17400459b1550a0612d7275b4b3bfb3bdaf2473e0958f1fd53120897a"
Write done in 34 seconds
DownloadExtractThread::_verify() called (child class implementation with progress updates)
Post-write verification using 4096 KB buffer for 2792 MB image
Verify hash: "687005a17400459b1550a0612d7275b4b3bfb3bdaf2473e0958f1fd53120897a"
Verify done in 8.667 seconds
Writing first block (which we skipped at first)
Write successful.                         
Stopping background drive list polling
Cleaning up CacheManager
CacheManager destructor: cleaning up background thread
CacheManager destructor: cleanup complete
Cancelling running thread in ImageWriter destructor
Ejecting drive:  "BIWIN-CE430T5D100-512G-2534093201986"
# check disk structure: 
sudo lsblk -f
NAME        FSTYPE FSVER LABEL  UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0       swap   1                                                                
mmcblk0                                                                             
├─mmcblk0p1 vfat   FAT32 bootfs 1C94-4EC3                                           
└─mmcblk0p2 ext4   1.0   rootfs f0abac56-08be-42e2-8726-9baa083e8685                
zram0       swap   1     zram0  65273a6c-86db-4465-9d1d-a435435b39a8                [SWAP]
nvme0n1                                                                             
├─nvme0n1p1 vfat   FAT32 bootfs 1C94-4EC3                             436.9M    14% /boot/firmware
└─nvme0n1p2 ext4   1.0   rootfs f0abac56-08be-42e2-8726-9baa083e8685  446.1G     1% /

```

Then:
1. Set boot order to **NVMe first** in firmware.  
2. Reboot → system boots cleanly from NVMe.
3. Configure user and password for the new OS.  
4. Shutdown and remove the SD card ✅  

---

### 🔋 RTC & Power Configuration
After successful NVMe boot:
- Enabled **RTC battery charging**  
  → [Official guide](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#enable-battery-charging)  
- Enabled **battery alarm wake-up** from halt.  
- Verified RTC retains time across full power loss.

---

## 🔗 References
- [Waveshare PoE M.2 HAT (B) – NVMe SSD Boot](https://www.waveshare.com/wiki/PoE_M.2_HAT+_(B)#NVMe_SSD_boot)  
- [Raspberry Pi Documentation – Enable Battery Charging](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#enable-battery-charging)  
- [JetKVM Web Console](https://app.jetkvm.com)  
- [Raspberry Pi Imager CLI v2.0.0-rc3](https://github.com/raspberrypi/rpi-imager/releases/download/v2.0.0-rc3/Raspberry_Pi_Imager-2.0.0-cli-aarch64.AppImage)  
- [Raspberry Pi OS Lite 2025-10-02 (ARM64)](https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2025-10-02/)

---

## 🧩 Future Plans – Cluster Expansion
- Add **2–4 additional Pi 5 nodes**, each powered via PoE.  
- Centralised management and provisioning (PXE / NVMe boot).  
- Explore **Kubernetes**, **Docker Swarm**, or **Nomad** for orchestration.  
- Implement shared storage (NFS / GlusterFS) and monitoring stack (Prometheus + Grafana).  

---

© 2025 Martin Lambev · Raspberry Pi 5 PoE Cluster Project

