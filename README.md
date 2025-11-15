# Fedora T2 Linux Tuning Guide for MacBook Pro 2019 16

This guide provides instructions and configuration snippets for optimizing **T2 Linux Fedora Workstation T2 Linux** on a **MacBook Pro 2019 (16-inch model)**. It focuses on improving battery life, performance, and addressing common hardware and software quirks for daily usability.

| Detail | Specification |
| :--- | :--- |
| **Target Fedora Version** | Fedora 43 (Workstation Edition) |
| **Author** | [g-r-3-y] |
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

As a first step we disable automatic suspend in GNOME. 

### 2.1. **GNOME** Power Settings

Using the GNOME desktop `Settings` app:
1. Open `Settings` app.
2. Select `Power` from the settings menu.
3. Toggle the Automatic Suspend switch to `Off`. 

Next we will use **TLP** and a dedicated fan control daemon.

### 2.2. Installing TLP and Removing Conflicts

**TLP** applies system-level power-saving tweaks automatically. Before installation, you must **remove or mask conflicting power management services** to ensure TLP functions correctly.

You must remove or mask the conflicting power management tool, which is **`tuned`** and **`tuned-ppd`** on **Fedora 41 and newer**. You also need to **mask** the **`systemd-rfkill.service`** and **`systemd-rfkill.socket`** services to prevent interference and allow TLP's radio device switching to work properly.

#### 2.2.1. Removal of Conflicting Tools and Masking Services

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

#### 2.2.2. TLP Installation

```bash
# Install TLP
sudo dnf install tlp tlp-rdw     

# Enable and start the TLP service
sudo systemctl enable tlp
sudo systemctl start tlp
```

#### 2.2.3. TLP Configuration `/etc/tlp.conf`

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

#### 2.2.4. Processor Information `tlp-stat -p`

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

#### 2.2.5. Thermal Information `tlp-stat -t`

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

2. **Apply the settings:**

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

### 2.5 Suspend and Closing of the Lid

#### 2.5.1 Suspend

Suspension currently does not work. After booting `Fedora 42.2 Live iso` the suspend does work and the workstation resumes correctly. However after installation the suspend does not work. The workstation will be configured not to suspend, but to lock the session after closing of the lid.

#### 2.5.2 Session Lock

After closing of the lid the session will be locked.

1. **Edit `/etc/systemd/logind.conf.d/logind.conf`:**

    ```bash
    #Session Lock settings
    HandleLidSwitch=lock
    HandleLidSwitchExternalPower=lock
    HandleLidSwitchDocked=lock
    ```

The power button next to the touchbar can be configured to shutdown the system.

2. **Edit `/etc/systemd/logind.conf.d/logind.conf`:**

    ```bash
    #Shutdown settings
    HandlePowerKey=poweroff
    ```

3. **Restart `systemd-logind`:**

    ```bash
    # Restart systemd-logind service
    sudo systemctl restart systemd-logind
    ```

After opening of the lid, the password can be typed directly without pressing the `ENTER` key to unlock the session.

## 3. 🖥️ Graphics: Configuring Intel iGPU and AMD dGPU

The MacBook Pro 2019 16 requires dynamic management of its two GPUs: the low-power integrated **Intel iGPU** and the high-performance dedicated **AMD Radeon dGPU**. The **Hybrid Mode** approach provides the best flexibility, using the Intel iGPU by default while keeping the AMD dGPU available but in a low-power state. That reduces power consumption and heat.

We will use a dedicated **GPU Switcher Script** (from https://github.com/basecamp/omarchy/issues/1844) to automate the complex modifications required for `GRUB`, `modprobe`, and `udev` rules.

---

### 3.1. Installing `dracut`
    
```bash
# Install dracut tool for generating an initramfs/initrd image
sudo dnf install dracut
```

### 3.2. Installing the GPU Switcher Script

Save the provided script content (named `gpu-switcher.sh`) into an executable file, typically in `/usr/local/bin/` or `~/bin`.

#### 3.2.1. Create and Install the Script

1.  **Create the script file:**

    ```bash
    # Paste the entire script content here, then save and exit.
    sudo nano gpu-switcher.sh
    ```

2. **Script content:**

    ```bash
    #!/bin/bash

    # T2 Linux GPU Switcher Script
    # Manages Intel iGPU and AMD dGPU configuration
    # Based on:
    # @ngodn Ein Sof
    # [https://github.com/basecamp/omarchy/issues/1844](https://github.com/basecamp/omarchy/issues/1844)
    # v0.2

    set -e

    APPLE_GMUX_CONF="/etc/modprobe.d/apple-gmux.conf"
    BLACKLIST_CONF="/etc/modprobe.d/gpu-blacklist.conf"
    GRUB_CONFIG="/etc/default/grub"
    AMDGPU_DPM_RULES="/etc/udev/rules.d/30-amdgpu-pm.rules"
    BACKUP_DIR="$HOME/.gpu-switcher-backup"

    # Colors for output
    RED='\033[0;31m'
    GREEN='\033[0;32m'
    YELLOW='\033[1;33m'
    BLUE='\033[0;34m'
    NC='\033[0m' # No Color

    print_status() {
        echo -e "${BLUE}[INFO]${NC} $1"
    }

    print_success() {
        echo -e "${GREEN}[SUCCESS]${NC} $1"
    }

    print_warning() {
        echo -e "${YELLOW}[WARNING]${NC} $1"
    }

    print_error() {
        echo -e "${RED}[ERROR]${NC} $1"
    }

    check_root() {
        if [[ $EUID -ne 0 ]]; then
            print_error "This script must be run as root (use sudo)"
            exit 1
        fi
    }

    create_backup() {
        print_status "Creating backup of current configuration..."
        mkdir -p "$BACKUP_DIR"

        # Backup current configs
        [[ -f "$APPLE_GMUX_CONF" ]] && cp "$APPLE_GMUX_CONF" "$BACKUP_DIR/apple-gmux.conf.bak"
        [[ -f "$BLACKLIST_CONF" ]] && cp "$BLACKLIST_CONF" "$BACKUP_DIR/gpu-blacklist.conf.bak"
        [[ -f "$GRUB_CONFIG" ]] && cp "$GRUB_CONFIG" "$BACKUP_DIR/grub.bak"
        [[ -f "$AMDGPU_DPM_RULES" ]] && cp "$AMDGPU_DPM_RULES" "$BACKUP_DIR/30-amdgpu-pm.rules.bak"

        # Save current kernel cmdline
        cat /proc/cmdline > "$BACKUP_DIR/cmdline.bak"

        print_success "Backup created in $BACKUP_DIR"
    }

    restore_backup() {
        print_status "Restoring previous configuration..."

        if [[ ! -d "$BACKUP_DIR" ]]; then
            print_error "No backup found in $BACKUP_DIR"
            exit 1
        fi

        # Restore configs
        [[ -f "$BACKUP_DIR/apple-gmux.conf.bak" ]] && cp "$BACKUP_DIR/apple-gmux.conf.bak" "$APPLE_GMUX_CONF"
        [[ -f "$BACKUP_DIR/grub.bak" ]] && cp "$BACKUP_DIR/grub.bak" "$GRUB_CONFIG"
        [[ -f "$BACKUP_DIR/30-amdgpu-pm.rules.bak" ]] && cp "$BACKUP_DIR/30-amdgpu-pm.rules.bak" "$AMDGPU_DPM_RULES"

        # Remove files if they were created by this script
        [[ -f "$BLACKLIST_CONF" ]] && rm -f "$BLACKLIST_CONF"
        [[ -f "$AMDGPU_DPM_RULES" ]] && [[ ! -f "$BACKUP_DIR/30-amdgpu-pm.rules.bak" ]] && rm -f "$AMDGPU_DPM_RULES"

        update_system
        print_success "Configuration restored from backup"
    }

    configure_intel_only() {
        print_status "Configuring Intel GPU only..."

        # Enable iGPU in apple-gmux
        echo "# Enable the iGPU by default if present" > "$APPLE_GMUX_CONF"
        echo "options apple-gmux force_igd=y" >> "$APPLE_GMUX_CONF"

        # Blacklist AMD GPU
        echo "# Blacklist AMD GPU for Intel-only mode" > "$BLACKLIST_CONF"
        echo "blacklist amdgpu" >> "$BLACKLIST_CONF"
        echo "blacklist radeon" >> "$BLACKLIST_CONF"

        # Remove AMD DPM rules since AMD GPU is blacklisted
        [[ -f "$AMDGPU_DPM_RULES" ]] && rm -f "$AMDGPU_DPM_RULES"

        # Ensure Intel GUC is enabled
        if ! grep -q "i915.enable_guc=2" "$GRUB_CONFIG"; then
            sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="/&i915.enable_guc=2 /' "$GRUB_CONFIG"
        fi

        print_success "Intel GPU only configuration applied"
    }

    configure_amd_only() {
        print_status "Configuring AMD GPU only..."

        # Disable iGPU in apple-gmux
        echo "# Disable the iGPU to use dGPU" > "$APPLE_GMUX_CONF"
        echo "options apple-gmux force_igd=n" >> "$APPLE_GMUX_CONF"

        # Blacklist Intel GPU
        echo "# Blacklist Intel GPU for AMD-only mode" > "$BLACKLIST_CONF"
        echo "blacklist i915" >> "$BLACKLIST_CONF"

        # Configure AMD GPU DPM for low power (default for better thermal management)
        echo "# Set AMD GPU to low power mode by default" > "$AMDGPU_DPM_RULES"
        echo "SUBSYSTEM==\"drm\", DRIVERS==\"amdgpu\", ATTR{device/power_dpm_force_performance_level}=\"low\"" >> "$AMDGPU_DPM_RULES"

        # Remove Intel GUC parameter
        sed -i 's/ i915.enable_guc=[0-9]//g' "$GRUB_CONFIG"

        print_success "AMD GPU only configuration applied"
    }

    configure_hybrid() {
        print_status "Configuring hybrid graphics (both GPUs available)..."

        # Allow both GPUs, but prefer iGPU
        echo "# Allow both GPUs - hybrid mode" > "$APPLE_GMUX_CONF"
        echo "options apple-gmux force_igd=y" >> "$APPLE_GMUX_CONF"

        # Remove any GPU blacklists
        [[ -f "$BLACKLIST_CONF" ]] && rm -f "$BLACKLIST_CONF"

        # Configure AMD GPU DPM for low power (default for better thermal management)
        echo "# Set AMD GPU to low power mode by default in hybrid mode" > "$AMDGPU_DPM_RULES"
        echo "SUBSYSTEM==\"drm\", DRIVERS==\"amdgpu\", ATTR{device/power_dpm_force_performance_level}=\"low\"" >> "$AMDGPU_DPM_RULES"

        # Keep Intel GUC for better performance
        if ! grep -q "i915.enable_guc=2" "$GRUB_CONFIG"; then
            sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="/&i915.enable_guc=2 /' "$GRUB_CONFIG"
        fi

        print_success "Hybrid graphics configuration applied"
    }

    update_system() {
        print_status "Updating system configuration..."

        # Update GRUB
        grub2-mkconfig -o /boot/grub2/grub.cfg

        # Regenerate initramfs
        dracut --force 

        # Reload udev rules
        udevadm control --reload-rules
        udevadm trigger

        print_success "System configuration updated"
    }

    show_current_status() {
        echo -e "\n${BLUE}=== Current GPU Status ===${NC}"

        echo -e "\n${YELLOW}Loaded GPU drivers:${NC}"
        lsmod | grep -E "(i915|amdgpu)" || echo "No GPU drivers loaded"

        echo -e "\n${YELLOW}Active GPU (OpenGL):${NC}"
        if command -v glxinfo >/dev/null 2>&1; then
            glxinfo | grep -E "(OpenGL vendor|OpenGL renderer)" || echo "Unable to detect"
        else
            echo "glxinfo not available (install mesa-utils)"
        fi

        echo -e "\n${YELLOW}Apple GMux config:${NC}"
        [[ -f "$APPLE_GMUX_CONF" ]] && cat "$APPLE_GMUX_CONF" || echo "Not configured"

        echo -e "\n${YELLOW}GPU blacklist:${NC}"
        [[ -f "$BLACKLIST_CONF" ]] && cat "$BLACKLIST_CONF" || echo "No blacklists"

        echo -e "\n${YELLOW}AMD DPM configuration:${NC}"
        [[ -f "$AMDGPU_DPM_RULES" ]] && cat "$AMDGPU_DPM_RULES" || echo "No DPM rules configured"

        echo -e "\n${YELLOW}Current AMD DPM level:${NC}"
        if [[ -f /sys/bus/pci/drivers/amdgpu/*/power_dpm_force_performance_level ]]; then
            cat /sys/bus/pci/drivers/amdgpu/*/power_dpm_force_performance_level 2>/dev/null || echo "AMD GPU not active or accessible"
        else
            echo "AMD GPU not found or not loaded"
        fi

        echo -e "\n${YELLOW}Kernel parameters:${NC}"
        cat /proc/cmdline | grep -o 'i915[^ ]*' || echo "No i915 parameters"
    }

    set_amd_dpm() {
        local mode="$1"
        if [[ "$mode" != "low" && "$mode" != "high" ]]; then
            print_error "Invalid DPM mode. Use 'low' or 'high'"
            return 1
        fi

        print_status "Setting AMD DPM to $mode performance level..."

        # Update udev rules
        if [[ -f "$AMDGPU_DPM_RULES" ]]; then
            sed -i "s/=\"low\"/=\"$mode\"/g; s/=\"high\"/=\"$mode\"/g" "$AMDGPU_DPM_RULES"
            print_success "AMD DPM udev rule updated to $mode"
        else
            print_warning "No AMD DPM rules found. Run amd-only or hybrid mode first."
            return 1
        fi

        # Apply immediately if AMD GPU is loaded
        if [[ -f /sys/bus/pci/drivers/amdgpu/*/power_dpm_force_performance_level ]]; then
            echo "$mode" | tee /sys/bus/pci/drivers/amdgpu/*/power_dpm_force_performance_level >/dev/null 2>&1
            print_success "AMD DPM immediately set to $mode"
        else
            print_warning "AMD GPU not currently loaded. Changes will take effect after reboot."
        fi

        # Reload udev rules
        udevadm control --reload-rules
        udevadm trigger
    }

    show_help() {
        echo "T2 Linux GPU Switcher"
        echo "Usage: sudo \$0 [OPTION]"
        echo ""
        echo "Options:"
        echo "  intel-only    Configure Intel GPU only (power saving)"
        echo "  amd-only      Configure AMD GPU only (performance)"
        echo "  hybrid        Configure both GPUs (hybrid mode)"
        echo "  restore       Restore previous configuration"
        echo "  status        Show current GPU status"
        echo "  amd-dpm-low   Set AMD DPM to low power (thermal management)"
        echo "  amd-dpm-high  Set AMD DPM to high performance (gaming/rendering)"
        echo "  help          Show this help message"
        echo ""
        echo "Note: This script requires root privileges and will:"
        echo "- Modify /etc/modprobe.d/ configurations"
        echo "- Update GRUB configuration"
        echo "- Configure AMD DPM via udev rules"
        echo "- Regenerate initramfs"
        echo "- Require a reboot to take effect (except DPM changes)"
    }

    main() {
        case "\${1:-}" in
            "intel-only")
                check_root
                create_backup
                configure_intel_only
                update_system
                print_warning "Reboot required for changes to take effect"
                ;;
            "amd-only")
                check_root
                create_backup
                configure_amd_only
                update_system
                print_warning "Reboot required for changes to take effect"
                ;;
            "hybrid")
                check_root
                create_backup
                configure_hybrid
                update_system
                print_warning "Reboot required for changes to take effect"
                ;;
            "restore")
                check_root
                restore_backup
                print_warning "Reboot required for changes to take effect"
                ;;
            "status")
                show_current_status
                ;;
            "amd-dpm-low")
                check_root
                set_amd_dpm "low"
                ;;
            "amd-dpm-high")
                check_root
                set_amd_dpm "high"
                ;;
            "help"|"--help"|"-h")
                show_help
                ;;
            *)
                show_help
                exit 1
                ;;
        esac
    }

    main "\$@"

    ```

3. **Make the script executable:**

   ```bash
   sudo chmod +x gpu-switcher.sh
   ```

#### 3.2.2. Run in Hybrid Mode (Default Recommendation)

This command applies the necessary configuration to make both GPUs available but forces the AMD card to its lowest performance level for better battery life and thermal management.

```bash
sudo gpu-switcher.sh hybrid amd-dpm-low
```

Note: This script automatically creates a backup of your system files, applies GPU configuration changes, updates GRUB, and regenerates the initramfs. A reboot is required for kernel-level changes to take full effect.

### 3.3. Switching GPU Modes

The script provides commands to quickly switch between the three primary graphics configurations.

* Mode	        Command	Primary Use	Notes
* Hybrid	    sudo gpu-switcher.sh hybrid	Best Balance, Default.	Both GPUs loaded; AMD is throttled low.
* Intel Only    sudo gpu-switcher.sh intel-only	Maximum Battery Savings.	Blacklists the AMD driver entirely.
* AMD Only	    sudo gpu-switcher.sh amd-only	Maximum Performance/External Display.	Blacklists the Intel driver; high power usage.


### 3.4. Managing AMD Dynamic Power Management (DPM)

In Hybrid or AMD Only mode, you can change the dGPU's DPM level instantly without rebooting. This setting controls the power profile of the AMD card (e.g., for gaming or rendering vs. idle).

#### 3.4.1. Set DPM to Low Power

Use this for general tasks to minimize heat and noise.

```bash
sudo gpu-switcher.sh amd-dpm-low
```

#### 3.4.2. Set DPM to High Performance

Use this for demanding applications to maximize rendering speed.

```bash
sudo gpu-switcher.sh amd-dpm-high
 ```
   
### 3.5. Verification and Status

Verify the current active configuration using the script's status command.

#### 3.5.1. Show Current Status

```bash
sudo gpu-switcher.sh status
```

#### 3.5.2. Restore Configuration

You can revert the system files to the state they were in before the script was first run. This typically requires a reboot afterward.

```bash
sudo gpu-switcher.sh restore
```
