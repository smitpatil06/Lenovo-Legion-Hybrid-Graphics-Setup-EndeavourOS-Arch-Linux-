# Lenovo Legion Hybrid Graphics Setup (EndeavourOS / Arch Linux)

> **Status:** Stable
> **OS:** EndeavourOS (Arch Linux)
> **Kernel:** `6.18.9-arch1-2` (Latest Custom/Bleeding Edge)
> **WM:** dwm
> **Compositor:** Picom (Custom Config)

## 📖 Overview

This repository documents the precise configuration required to run a fully stable **Hybrid Graphics** setup on a Lenovo Legion 5 (15IR) laptop.

This configuration solves specific, critical crashes related to:

1. **"Invalid Head" Freeze:** Caused by the NVIDIA GPU sleeping too aggressively and losing video memory during fullscreen transitions.
2. **Interrupt Storm:** Caused by High-Polling Rate (4KHz/8KHz) mice crashing the CPU during GPU context switches.
3. **IBT Conflict:** A security feature in 13th Gen Intel CPUs that prevents the proprietary NVIDIA driver from loading.

## 💻 System Specifications

The configuration was tested and verified on the following hardware:

| Component | Specification | Detail |
| --- | --- | --- |
| **Host** | Lenovo Legion 5 15IR | Model 83LY |
| **CPU** | 13th Gen Intel Core i7 | Raptor Lake-S |
| **iGPU** | Intel UHD Graphics | Raptor Lake-S (rev 04) |
| **dGPU** | NVIDIA GeForce RTX 5060 | Mobile / Max-Q (rev a1) |
| **Kernel** | Linux 6.18.9-arch1-2 | `SMP PREEMPT_DYNAMIC` |
| **Storage** | NVMe SSD | 512GB (Boot/Root/Home split) |
| **Resolution** | 1920x1200 (Internal) | 1920x1080 (External HDMI) |

---

## 🛠 Step 1: Pre-Installation Cleanup

Before installing the drivers, ensure the system is clean of conflicting drivers. Specifically, the legacy Intel driver and the open-source Nouveau Vulkan driver can cause tearing and freezes.

```bash
# Remove Legacy Intel & Nouveau drivers to prevent conflicts
sudo pacman -Rns xf86-video-intel vulkan-nouveau

```

## 📦 Step 2: Driver Installation

We use the **Arch Wiki compliant** driver stack (`nvidia-open`) for modern Turing+ GPUs.

```bash
sudo pacman -S nvidia-open nvidia-utils nvidia-settings nvidia-prime

```

* **`nvidia-open`**: Open-source kernel module (Required for RTX 20-series and newer).
* **`nvidia-prime`**: Required for offloading specific applications to the dGPU.

## ⚙️ Step 3: Kernel Parameters (GRUB)

Specific kernel flags are required to fix hardware-specific freezes on Raptor Lake CPUs and to handle high-polling rate peripherals.

**File:** `/etc/default/grub`

```bash
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet nvidia-drm.modeset=1 ibt=off usbhid.mousepoll=1"

```

### 🔍 Explanation of Flags:

1. **`nvidia-drm.modeset=1`**: **Required.** Enables DRM (Direct Rendering Manager) KMS, which is essential for VSync and proper resolution switching.
2. **`ibt=off`**: **Critical for 12th/13th/14th Gen Intel.** Disables "Indirect Branch Tracking." Without this, the NVIDIA driver often fails to initialize or crashes on boot due to a kernel security conflict.
3. **`usbhid.mousepoll=1`**: **Critical for Gaming Mice.** Forces all USB mice to a polling rate of 1000Hz (1ms).
* *Issue:* Linux's X11 server can crash (Interrupt Storm) when switching to fullscreen if a mouse is spamming 8,000 interrupts per second.
* *Fix:* This caps the interrupt rate at the hardware level, preventing the CPU freeze.



**Apply Changes:**

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg

```

## 🔋 Step 4: Hybrid Power Management (The "Freeze" Fix)

By default, the NVIDIA driver enters "D3Cold" (Deep Sleep) when the HDMI is unplugged. If you fullscreen a video, the GPU wakes up but comes back with empty video memory, causing the `NVRM: Invalid Head Number` crash.

### 1. Configure NVIDIA Power Options

Create/Edit: `/etc/modprobe.d/nvidia.conf`

```bash
options nvidia NVreg_DynamicPowerManagement=0x02
options nvidia NVreg_PreserveVideoMemoryAllocations=1

```

* **`0x02` (Fine-Grained Power Control):** The GPU turns **OFF** completely on battery (0 Watts) but stays **ON** when an HDMI display is connected. [Source: NVIDIA Documentation](https://www.google.com/search?q=http://us.download.nvidia.com/XFree86/Linux-x86_64/535.113.01/README/dynamicpowermanagement.html)
* **`PreserveVideoMemoryAllocations=1`:** Saves GPU VRAM to system RAM before sleeping. This ensures that when the GPU wakes up for a game or video, the memory is restored instantly, preventing the crash. [Source: NVIDIA Power Management](https://www.google.com/search?q=http://us.download.nvidia.com/XFree86/Linux-x86_64/535.113.01/README/powermanagement.html)

### 2. Enable Systemd Services

These services handle the save/restore logic defined above.

```bash
sudo systemctl enable --now nvidia-suspend.service
sudo systemctl enable --now nvidia-hibernate.service
sudo systemctl enable --now nvidia-resume.service
# Keeps the driver loaded in memory to prevent frequent unload/reload cycles
sudo systemctl enable --now nvidia-persistenced.service

```

## 🖥 Step 5: Display Logic (.xinitrc)

This script manages the handoff between the Intel iGPU (Laptop screen) and NVIDIA dGPU (External Monitor).

**Repo Link:** [View on GitHub](https://github.com/smitpatil06/.dotfiles/blob/main/x11/.xinitrc)

**File:** `~/.xinitrc`

```bash
#!/bin/sh

# Force Qt to use the qt5ct/qt6ct configuration tool
export QT_QPA_PLATFORMTHEME=qt5ct

# Fix scaling for High DPI monitors
export QT_AUTO_SCREEN_SCALE_FACTOR=1
export QT_ENABLE_HIGHDPI_SCALING=1

# Optional: If you use a dark theme, sometimes this helps
export QT_STYLE_OVERRIDE=kvantum

userresources=$HOME/.Xresources
usermodmap=$HOME/.Xmodmap
sysresources=/etc/X11/xinit/.Xresources
sysmodmap=/etc/X11/xinit/.Xmodmap

# merge in defaults and keymaps
if [ -f $sysresources ]; then
    xrdb -merge $sysresources
fi

if [ -f $sysmodmap ]; then
    xmodmap $sysmodmap
fi

if [ -f $userresources ]; then
    xrdb -merge $userresources
fi

if [ -f $usermodmap ]; then
    xmodmap $usermodmap
fi

# start some nice programs
if [ -d /etc/X11/xinit/xinitrc.d ] ; then
    for f in /etc/X11/xinit/xinitrc.d/?*.sh ; do
        [ -x "$f" ] && . "$f"
    done
fi

# Custom
# Define Output Names
EXT="HDMI-1-0"
INT="eDP-1"

# Clean up any existing picom instances BEFORE setup
killall -q picom

# Connect Intel iGPU to NVIDIA GPU
xrandr --setprovideroutputsource modesetting NVIDIA-0

if xrandr | grep -q "$EXT connected"; then
    # --- DUAL MONITOR SETUP (NVIDIA) ---
    xrandr --output $EXT --mode 1920x1080 --rate 100.00 --primary \
           --output $INT --mode 1920x1200 --rate 60.00 --left-of $EXT

    # NVIDIA Driver handles VSync (ForceCompositionPipeline)
    nvidia-settings --assign CurrentMetaMode="nvidia-auto-select +0+0 {ForceCompositionPipeline=On}" &
    
    # Light Picom (Driver does VSync, so we don't need to)
    picom -b --config ~/.config/picom/picom.conf --backend glx --no-vsync &

else
    # --- LAPTOP ONLY SETUP (Intel) ---
    xrandr --output $INT --auto --primary --output $EXT --off

    # Heavy Picom (Compositor MUST handle VSync to stop tearing)
    picom -b --config ~/.config/picom/picom.conf --backend glx --vsync --no-fading-openclose &
fi

# --- Background Processes ---
rfkill unblock wlan &
#/usr/lib/polkit-gnome/polkit-gnome-authentication-agent-1 &

dunst &
slstatus &

# Wallpaper loop
while true; do
    feh --bg-fill --randomize ~/Wallpapers/*
    sleep 300
done &

exec dwm

```

## 🚀 Final Verification

After rebooting, verify the hybrid state:

1. **Check Hybrid Power Mode:**
```bash
cat /proc/driver/nvidia/params | grep DynamicPowerManagement
# Output should be: DynamicPowerManagement: 2

```


2. **Check Mouse Polling Rate:**
```bash
cat /sys/module/usbhid/parameters/mousepoll
# Output should be: 1 (1000Hz)

```



## 🔗 References

* [Arch Wiki: NVIDIA](https://wiki.archlinux.org/title/NVIDIA)
* [Arch Wiki: PRIME](https://wiki.archlinux.org/title/PRIME)
* [Arch Wiki: NVIDIA Tips & Tricks (Preserve Memory)](https://www.google.com/search?q=https://wiki.archlinux.org/title/NVIDIA/Tips_and_tricks%23Preserve_video_memory_after_suspend)
