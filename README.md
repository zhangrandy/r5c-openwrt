# OpenWrt 25.12 Custom Build for NanoPi R5C



This repository contains an automated GitHub Actions workflow to compile a custom, vanilla OpenWrt 25.12.x firmware for the FriendlyElec NanoPi R5C. 

This build is specifically engineered to serve as a clean, stable baseline for running **Passwall** (Xray/Sing-box) as a transparent proxy. It utilizes the modern `apk` package manager and permanently bakes in strict kernel modules (`kmod`) to bypass OpenWrt's "vermagic" hash restrictions during manual offline installations.

## 🌟 Key Features of this Build
* **Expanded RootFS**: Fixed at 4GB SquashFS (Ext4 disabled) to prevent upgrade corruption.
* **Pre-Baked Kernel Modules**: Includes all Netfilter (`nftables`), `tproxy`, and `tun` modules required by Passwall so `apk` doesn't throw kernel-mismatch errors during offline installations.
* **Native Chinese UI**: Correctly utilizes the `CONFIG_LUCI_LANG_zh_Hans=y` compiler flag to build the `luci-i18n-base-zh-cn` dictionary.
* **Replaced DNS**: Swaps the default DNS manager for `dnsmasq-full` (required for proxy routing).
* **Storage Utilities**: Includes `fdisk`, `lsblk`, `e2fsprogs`, and `partx-utils` to manually reclaim the rest of the 32GB eMMC chip.
* **Daily Driver Tools**: Ships with `curl`, `unzip`, `nano`, `htop`, and `openssh-sftp-server` for easy headless management.

---

## 🚀 Upgrade & Post-Flash Setup Guide

How you set up your storage and configurations depends entirely on whether you are doing a routine point upgrade or a fresh installation.

### ⚠️ Point Upgrades (via LuCI Web UI)
If you are upgrading to a new point release (e.g., 25.12.x to 25.12.y) via the LuCI web interface, **do not** tick the "Keep Settings" checkbox. Attempting to keep settings across custom `apk` builds causes migration bugs and script errors. 

1. Flash the new `.img.gz` via LuCI with **Keep Settings unchecked**.
2. After the router reboots, manually restore your system configuration backup via LuCI -> System -> Backup / Flash Firmware.
3. *Note: You do not need to run the `fdisk` or `fstab` steps below. Point upgrades via LuCI preserve your partition table and `/mnt/data` drive automatically!*

### 💾 Initial Installation (via FriendlyElec `eflasher`)
Because this is a fresh installation, we will use FriendlyElec's official SD-to-eMMC `eflasher` method. This writes a completely fresh partition table to the eMMC, meaning you must manually reclaim the remaining space on the 32GB drive and mount it as `/mnt/data` afterward.

**Phase 1: Flashing the Firmware to eMMC**
1. Download any FriendlyElec firmware file containing `eflasher` in its name (usually found in their official cloud drive under the "01_System Firmware / 02_SD Card Flashing Firmware" directory). Extract it and flash it to a MicroSD (TF) card.
2. Re-insert the MicroSD card into your computer. A drive named `FriendlyARM` will appear.
3. Copy your newly compiled custom OpenWrt `.img.gz` file directly into this `FriendlyARM` drive. *(Note: If your file ends in just `.img`, rename it to `.raw`).*
4. Open the `eflasher.conf` file on the MicroSD card using a text editor. Change the `autoStart=` line to exactly match your firmware filename. For example:
   `autoStart=openwrt-rockchip-armv8-friendlyelec_nanopi-r5c-ext4-sysupgrade.img.gz`
5. Safely eject the MicroSD card and insert it into your powered-off NanoPi R5C. **Important:** If there is already a system on the eMMC (which defaults to booting first), you must force the router to boot from the SD card. Use a paperclip to press and hold the **MASK** button, plug in the power, count to 3 seconds, and then release the MASK button. The router will now boot from the SD card and automatically copy the system over to the internal eMMC. Watch the onboard LEDs to monitor the progress. Once complete, power off the device, remove the SD card, and boot normally from the internal eMMC.

**Phase 2: Reclaim the 24GB eMMC Storage**
Do not touch the 4GB `p2` system partition. We will only create `p3`.

```bash
fdisk /dev/mmcblk1
```
Execute these exact keystrokes:
1. `n` → `p` → `3` (New Primary Partition 3)
2. First Sector: Type **`8519680`** and press **Enter**. *(Crucial: Do not accept the default here, or it overwrites the bootloader).*
3. Last Sector: Press **Enter**. *(Accept the default to use all remaining space).*
4. `w` (Write and Exit)

Refresh the kernel and format the new partition to `ext4`:
```bash
partx -a /dev/mmcblk1
# Note: If partx says the drive is busy, type `reboot`, wait for restart, and continue below.

mkfs.ext4 /dev/mmcblk1p3
```

**Phase 3: Configure Persistent Mounting**
Create the directory and use OpenWrt's `block` utility to generate the UUID map.
```bash
mkdir -p /mnt/data
block detect > /etc/config/fstab
nano /etc/config/fstab
```
Edit the file to target `/mnt/data` and ensure it is enabled:
```text
config 'mount'
    option  target  '/mnt/data'
    option  device  '/dev/mmcblk1p3'
    option  fstype  'ext4'
    option  enabled '1'
```
Activate the mount and verify:
```bash
/etc/init.d/fstab boot
df -h
```

---

~~📦 Installing Passwall (Granular Offline Method)~~

## Including: Passwall, Xray-core, Sing-box, Hysteria2

## 🛠️ Known Quirks & Fixes

### 1. Forcing the Chinese LuCI UI
If the web interface defaults to English despite the language pack being installed, force the POSIX locale via SSH:
```bash
uci set luci.main.lang='zh_cn'
uci commit luci
/etc/init.d/rpcd restart
```
