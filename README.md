# Surface-book-2-camera-fix-in-LINUX
This guide shows you how to fix the cameras on your surface book 2 in linux. may work for other surface devices too.

Context:
I had just recently been trying out different distro's on my surface book 2, everything was a pain. i had a smooth experience previously on my surface laptop (the one that DOSN'T turn into a tablet) and various old asus and hp laptops but the surface book was difficult. sometimes the GPU won't show (Nvidia) sometimes touchscreen wouldn't work etc. Honestly, i had a LOT of help from opencode. without it, i'd be lost as to why things don't work, even through scrolling down reddit threads and tom's hardware posts. now the front camera works at least (i have a job interview in a few days) HOWEVER, the back camera isn't working, and the front camera, though it works, is in VGA quality AND its flipped upside down.

#USEFUL LINKS: 
- https://buymeacoffee.com/genghiskhan - **my buymeacoffee page if you want to support me** (i didn't make any of the fixes, i just had this issue and these fixes worked for me. if you want to support the development, click the links below)
- https://github.com/linux-surface/linux-surface — kernel patches & issues
- https://github.com/linux-surface/kernel — kernel tree with camera fixes
- https://github.com/linux-surface/linux-surface/pull/2123 — dw9719 fix
- https://github.com/linux-surface/linux-surface/pull/2169 — ov8865 green screen fix



# Surface Book 2 Camera Fix on CachyOS (linux-surface kernel) GUIDE

## Environment
- **Device:** Microsoft Surface Book 2 (13.5", i5-7300U, GTX 1050)
- **Distro:** CachyOS (Arch-based)
- **Kernel:** `linux-surface` 6.19.8-arch1-3-surface
- **Camera Stack:** libcamera 0.7.2 (from cachyos-extra-v3 repo)
- **Sensors:**
  - Front: OV5693 (INT33BE:00) — CSI-2 port 0
  - Rear: OV8865 + DW9719 VCM (INT347A:00) — CSI-2 port 2
  - IR: OV7251 (INT347E:00) — CSI-2 port 1

## Prerequisites

```bash
# Required packages
sudo pacman -S linux-surface linux-surface-headers libcamera-tools libcamera-ipa

# Build tools for kernel module
sudo pacman -S base-devel linux-surface-headers
```

## The Problem
On linux-surface 6.19.8, cameras are detected by the kernel but **libcamera reports zero cameras**. Root causes:

1. **dw9719 VCM driver missing `i2c_device_id` table** — upstream removed it; the CIO2 async notifier waits for the VCM to bind, so `cio2_notifier_complete()` never fires → all three sensors end up with 0 media links
2. **ov8865 stale mode bug** — rear camera streams one frame then green-screens (fixed in PR #2169, not in 6.19.8)
3. **Front sensor upside-down** — OV5693 mounted rotated 180°; needs rotation metadata (no calibration files for SB2 in libcamera 0.7.x)

## Fix Steps

### 1. Build fixed dw9719 module

```bash
mkdir -p ~/dw9719-fix && cd ~/dw9719-fix

# Get source with i2c_device_id table (from linux-surface master)
curl -sL https://raw.githubusercontent.com/linux-surface/kernel/master/drivers/media/i2c/dw9719.c -o dw9719.c

# Makefile for out-of-tree build
cat > Makefile <<'EOF'
obj-m += dw9719.o
KERNEL_BUILD := /lib/modules/$(shell uname -r)/build
all:
	make -C $(KERNEL_BUILD) M=$(PWD) modules
clean:
	make -C $(KERNEL_BUILD) M=$(PWD) clean
EOF

make
zstd -f -o dw9719.ko.zst dw9719.ko

# Install as compressed module (kernel expects .ko.zst)
sudo cp dw9719.ko.zst /lib/modules/$(uname -r)/kernel/drivers/media/i2c/dw9719.ko.zst
sudo depmod -a
```

### 2. Ensure IPU3 modules load on boot

```bash
echo -e "ipu3_cio2\nipu3_imgu\nipu_bridge" | sudo tee /etc/modules-load.d/surface-camera.conf
```

### 3. Reboot

```bash
sudo reboot
```

### 4. Verify

```bash
# Should show both cameras
cam --list

# Media graph should show sensor↔CSI2 links
media-ctl -p -d /dev/media0 | grep -A2 "ov"
```

## Current Status (6.19.8)

| Camera | Works? | Notes |
|--------|--------|-------|
| Front (OV5693) | ✅ | Image upside-down — rotate 180° in app |
| Rear (OV8865) | ❌ | Green screen after 1 frame (kernel bug) |
| IR (OV7251) | ⚠️ | Detected, no userspace support |

## Workarounds

**Front camera rotation:**

```bash
# Cheese/Guvcview respect rotation
cheese
guvcview

# GStreamer with manual rotate
gst-launch-1.0 libcamerasrc camera-name="Internal front camera" ! videoflip method=rotate-180 ! videoconvert ! autovideosink
```

**Rear camera (may work once per boot):**

```bash
gst-launch-1.0 libcamerasrc camera-name="Internal back camera" ! video/x-raw,width=1280,height=720 ! videoconvert ! autovideosink
```

## Upstream Fixes (watch these)
- **PR #2123** — dw9719 i2c_device_id table (fixes sensor linking)
- **PR #2169** — ov8865 stale mode fix (fixes rear camera green screen)
- **Calibration files** — OV5693/OV8865 YAML for libcamera IPA (rotation, color, vignetting)

## Notes
- INT3472 discrete GPIO regulators show "dummy regulator" warnings but power sequencing works
- No surface-camera-acpi repo needed — linux-surface master now includes the ACPI quirks
- If `cam --list` still empty after reboot:
  ```bash
  sudo modprobe -r ipu3_imgu ipu3_cio2 ipu_bridge && sudo modprobe ipu_bridge dw9719 ipu3_cio2 ipu3_imgu
  ```
