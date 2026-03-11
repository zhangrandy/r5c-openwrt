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

## 🚀 Post-Flash Setup Guide

Because flashing a new `.img.gz` file rewrites the partition table, you must manually reclaim the remaining space on the 32GB eMMC drive and mount it as `/mnt/data` after every sysupgrade.

### Step 1: Reclaim the 24GB eMMC Storage
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

### Step 2: Configure Persistent Mounting
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
    # Or keep option uuid '...' instead of device
    option  fstype  'ext4'
    option  enabled '1'
```
Activate the mount and verify:
```bash
/etc/init.d/fstab boot
df -h
```

---

## 📦 Installing Passwall (Offline Method)

Because third-party repositories often lag behind OpenWrt's `apk` transition, compiling Passwall directly or using their online repos can cause `Error 8` (404 Not Found) for `APKINDEX.tar.gz`. 

Use this manual, offline method directly on the router using our new `/mnt/data` drive:

```bash
# 1. Enter your workspace
cd /mnt/data

# 2. Download the latest aarch64_generic release
# (Replace the URL with the actual latest GitHub release link)
curl -L -o passwall.zip "https://github.com/moetayuko/openwrt-passwall-build/releases/download/vXYZ/passwall_packages_aarch64_generic.zip"

# 3. Extract and Install
unzip passwall.zip -d passwall_installer
cd passwall_installer
apk update
apk add --allow-untrusted $(find . -name "*.apk")

# 4. Clean up
cd /mnt/data
rm -rf passwall_installer passwall.zip
```

---

## 🛠️ Known Quirks & Fixes

### 1. Forcing the Chinese LuCI UI
If the web interface defaults to English despite the language pack being installed, force the POSIX locale via SSH:
```bash
uci set luci.main.lang='zh_cn'
uci commit luci
/etc/init.d/rpcd restart
```

### 2. Passwall "echolog" Error
When enabling Xray Load Balancing in Passwall, the background `nohup` script may crash with `nohup: failed to run command 'echolog': No such file or directory`. Fix this by creating a global wrapper script:
```bash
cat << 'EOF' > /usr/bin/echolog
#!/bin/sh
echo "$(date "+%Y-%m-%d %H:%M:%S"): $*" >> /tmp/log/passwall.log
EOF
chmod +x /usr/bin/echolog
```

### 3. Missing CPU Temperature in LuCI
Vanilla OpenWrt does not include customized thermal UI injections like Lean's LEDE. To get CPU load and SoC temperature on your LuCI Overview page, manually install `gSpotx2f`'s widgets.

*(The `lm-sensors` backend is already baked into this firmware).*
```bash
# Install CPU Status Widget
wget --no-check-certificate -O /tmp/luci-app-cpu.apk https://github.com/gSpotx2f/packages-openwrt/raw/master/25.12/luci-app-cpu-status-0.6.2-r1.apk
apk --allow-untrusted add /tmp/luci-app-cpu.apk

# Install Temp Status Widget
wget --no-check-certificate -O /tmp/luci-app-temp.apk https://github.com/gSpotx2f/packages-openwrt/raw/master/25.12/luci-app-temp-status-0.8.0-r1.apk
apk --allow-untrusted add /tmp/luci-app-temp.apk

rm /tmp/*.apk
service rpcd reload
```
