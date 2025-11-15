# Fedora T2 Linux Tuning Guide for MacBook Pro 2019 16

This guide provides instructions and configuration snippets for optimizing **Fedora Workstation T2 Linux** on a **MacBook Pro 2019 (16-inch model)**. It focuses on improving battery life, performance, and addressing common hardware and software quirks for daily usability.

| Detail | Specification |
| :--- | :--- |
| **Target Fedora Version** | Fedora 43 (Workstation Edition) |
| **Author** | [Your GitHub Username] |
| **License** | MIT License |

---

## 💻 Hardware Specifications

| Component | Detail |
| :--- | :--- |
| **Model** | MacBook Pro 2019 16 (`MacBookPro16,1`) |
| **Processor** | Intel Core i9-9880H |
| **RAM** | 16GB 2667MHz DDR4 |
| **Storage** | 1TB NVMe SSD |
| **iGPU** | Intel® UHD Graphics 630 (CFL GT2) |
| **dGPU** | AMD Radeon Pro 5500M 4GB |
| **Display** | 16-inch Retina Display (3072 x 1920) |

---

## ⚙️ Software & Configuration

| Component | Detail |
| :--- | :--- |
| **Firmware** | 2094.40.1.0.0 (iBridge: 23.16.11072.0.0,0) |
| **Primary OS** | Fedora Linux 43 (Workstation Edition) |
| **Dual Boot** | MacOS Tahoe 26.1 / Windows 11 24H2 |
| **Desktop** | GNOME 49 |
| **Compositor** | Wayland |
| **Kernel** | Linux 6.16.9-210.t2.fc43.x86_64 |

---

## ✅ Tuning Steps Overview

The following sections detail the required steps to optimize the system:

* **Installation quirks:** Sanitizing EFI, GRUB configuration.
* **Power Management:** Optimizing performance vs. power consumption / heat.
* **Graphics:** Integrated Intel iGPU and dedicated AMD gGPU configuration.
* **Audio:** Configuring DSP sound card and DSP microphone.
* **Network:** Securing DNS with Cloudflare WARP.
* **Display:** Configuring DPI and Night Light.
* **Password Management:** KeePass, Google Drive Sync and Web browser integration.
* **Crypto Wallets:** Ledger, Software wallets.
* **IOS Synchronization:** Mail, Calendar, Contacts, Notes, Passwords.
* **Messengers:** WhatsAPP, Telegram, Signal, Discord.
* **Video acceleration:** Codecs, Players.
* **Web browsers:** Firefox flavors, Chromium flavors.

---

## 1. 🗑️ Installation Quirks: Sanitizing EFI and GRUB (Via Fedora Live ISO)

If there was previously installed different flavour of Linux (Mint for example), the residual EFI boot entries (e.g., leftover `ubuntu` directories) may conflict with Fedora installation. To ensure a clean boot, we'll use the Fedora Live environment to manage the EFI System Partition (ESP) and the UEFI firmware entries.

### 1.1. Boot into Fedora Live Environment

1.  **Create a Fedora Live USB.**
2.  **Boot the MacBook Pro:** Hold down the **Option** **{⌥}** key and select the **EFI Boot** entry for the USB drive (the last entry to the right).
3.  **Open Terminal** once the Live desktop loads.

### 1.2. Clean up Residual EFI Directory

1.  **Identify the ESP:** Find your main disk (e.g., `/dev/nvme0n1`) and the small FAT32 ESP partition (e.g., `/dev/nvme0n1p1`).

    ```bash
    lsblk -f
    ```

2.  **Mount the ESP:** (Replace `/dev/sdXY` with your actual partition ID)

    ```bash
    sudo mkdir /mnt/efi
    sudo mount /dev/nvme0n1p1 /mnt/efi
    ```

3.  **Remove the Residual Directory:** Navigate to the EFI directory and delete the unwanted folder (e.g., `ubuntu`).

    ```bash
    cd /mnt/efi/EFI
    sudo rm -rf ubuntu
    ```

### 1.3. Delete the UEFI Boot Entry with `efibootmgr`

This step cleans the boot entries stored in the system's **NVRAM**.

1.  **List Current Boot Entries:** Find the hexadecimal boot number (e.g., `0008`) corresponding to the unwanted entry (e.g., "ubuntu").

    ```bash
    sudo efibootmgr
    ```

2.  **Delete the Entry:** Warning! Replace `0008` in this example with the correct number of your unwanted entry!

    ```bash
    sudo efibootmgr -b 0008 -B
    ```

3.  **Unmount and Reboot:**

    ```bash
    sudo umount /mnt/efi
    # Reboot the system and continue with Fedora installation
    ```

### 1.4. Post-Installation GRUB and Initrd Configuration

After successfully installing and booting into your new Fedora system, run these commands to ensure the boot environment is stable and correct for the running kernel:

1.  **Update `initrd` (Initial Ramdisk):**

    ```bash
    sudo dracut -f /boot/initramfs-$(uname -r).img $(uname -r)
    ```

2.  **Rebuild the GRUB Configuration:**

    ```bash
    sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg
    ```

***

## 2. ⚡ Power Management: Optimizing performance vs power consumption / heat

We will use **TLP** and a dedicated fan control daemon.

### 2.1. Installing TLP and Removing Conflicts

**TLP** applies system-level power-saving tweaks automatically. Before installation, you must **remove or mask conflicting power management services** to ensure TLP functions correctly.

You must remove or mask the conflicting power management tool, which is **`tuned`** and **`tuned-ppd`** on **Fedora 41 and newer**. You also need to **mask** the **`systemd-rfkill.service`** and **`systemd-rfkill.socket`** services to prevent interference and allow TLP's radio device switching to work properly.

#### 2.1.1. Removal of Conflicting Tools and Masking Services

##### For Fedora 41 and newer

1. **Remove `tuned` and `tuned-ppd`:**

    ```bash
    # Stop, disable, and remove tuned and its power-profiles-daemon plugin
    sudo dnf remove tuned tuned-ppd
    ```

2. **Mask `systemd-rfkill` services:**

    ```bash
    # Mask systemd-rfkill services to prevent interference with TLP
    sudo systemctl mask systemd-rfkill.service
    sudo systemctl mask systemd-rfkill.socket
    ```

#### 2.1.2. TLP Installation

```bash
# Install TLP
sudo dnf install tlp tlp-rdw

# Enable and start the TLP service
sudo systemctl enable tlp
sudo systemctl start tlp
```

#### 2.1.3. TLP Configuration `/etc/tlp.conf`

The following settings balance performance vs power consumption and heat:

1. **Edit `/etc/tlp.conf`:**

```bash
CPU_BOOST_ON_AC=0
CPU_HWP_DYN_BOOST_ON_AC=1
PLATFORM_PROFILE_ON_AC=balanced
RADEON_DPM_PERF_LEVEL_ON_AC=low
RADEON_DPM_STATE_ON_AC=balanced
RUNTIME_PM_DRIVER_DENYLIST="mei_me nouveau xhci_hcd"
```

2. **After edit of the configuration run:**

```bash
# Apply TLP settings
sudo tlp start
```

The changes are applied immediately by kernel 'intel_pstate' driver.

#### 2.2.1. Processor Information `tlp-stat -p`

1. **Verify the settings:**

```bash
# Verify TLP settings
sudo tlp-stat -p 
```

2. **Correct boost settings:**

```bash
/sys/devices/system/cpu/intel_pstate/no_turbo          =   1
/sys/devices/system/cpu/intel_pstate/hwp_dynamic_boost =   1
```

#### 2.2.2. Thermal Information `tlp-stat -t`

Displays CPU temperature and fan speeds:

```bash
# Thermal info
sudo tlp-stat -t 
```

### 2.3. Controlling `Intel Turbo Boost`

The above configuration of `tlp.conf` disables Intel Turbo Boost to reduce idle and light-load temperatures and power consumption.
For CPU intensive tasks it can be enabled back.

* **Temporarily disable:**

```bash 
# Disable Turbo Boost
echo "1" | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo
```

* **Temporarily enable:**

```bash 
# Enable Turbo Boost
echo "0" | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo
```

### 2.4. Fan Control Options

#### 2.4.1 `t2fanrd`

A dedicated fan daemon `t2fanrd` is installed and enabled on Fedora by default. 

1. **Edit `/etc/t2fand.conf`:**

```bash
#T2 fan control settings
[Fan1]
low_temp=55
high_temp=75
speed_curve=exponential
always_full_speed=false

[Fan2]
low_temp=55
high_temp=75
speed_curve=exponential
always_full_speed=false
```

2. **After edit run:**

```bash
# Apply t2fanrd settings
sudo systemctl restart t2fanrd
```

#### 2.4.2 `mbpfan`

For situations where the workstation is constantly under heavier load it is possible to use `mbpfan` daemon, which trades the fan silence for more heat under heavier load.

1. **Stop and disable `t2fanrd`:**

```bash
# Stop and disable t2fanrd service
sudo systemctl stop t2fanrd
sudo systemctl disable t2fanrd
```

2. **Install `mbpfan`:**
   
```bash
# Install mbpfan service
sudo dnf install mbpfan
```

3. **Edit `/etc/mbpfan.conf`:**

```bash
# Fan control settings
min_fan1_speed = 1500
max_fan1_speed = 6000
low_temp = 60 
high_temp = 65
max_temp = 86
```

4. **Enable and start `mbpfan`:**

```bash
# Enable and start mbpfan service
sudo systemctl enable mbpfan
sudo systemctl start mbpfan
```
