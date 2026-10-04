# Linux Mint CRT Setup with an AMD R7 240

This is how I got a consumer CRT working as a second display on Linux Mint/X11 using an AMD Radeon R7 240 for analog VGA output.

## Hardware Used

- AMD Radeon R7 240 with VGA output
- Consumer CRT with component input
- [Wakabavideo RGB VGA to YPbPr Component transcoder](https://www.ebay.com/itm/174166352619)
- VGA cable
- Component video cables

My signal chain is:

```text
R7 240 VGA
→ VGA cable
→ Wakabavideo RGB VGA to YPbPr transcoder
→ Component cables
→ CRT

The CRT modes I use are:

```text
320x240p
640x480i
```

The main issue I had was that 240p worked correctly, but 480i was horizontally compressed.

Switching the R7 240 from the `amdgpu` kernel driver to the older `radeon` driver fixed it.

---

## 1. Find the R7 240

Run:

```bash
lspci | grep -Ei 'vga|3d|display'
```

Mine appears as:

```text
11:00.0
```

Check which kernel driver it is using:

```bash
lspci -k -s 11:00.0
```

Originally mine showed:

```text
Kernel driver in use: amdgpu
```

---

## 2. Switch the R7 240 to the Radeon Driver

Edit GRUB:

```bash
sudo xed /etc/default/grub
```

Change:

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```

to:

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash radeon.si_support=1 amdgpu.si_support=0"
```

Then run:

```bash
sudo update-grub
sudo reboot
```

After rebooting, check again:

```bash
lspci -k -s 11:00.0
```

You should see:

```text
Kernel driver in use: radeon
```

### Why

On my R7 240:

| Driver | 240p | 480i |
|---|---|---|
| `amdgpu` | Works | Horizontally compressed |
| `radeon` | Works | Works |

---

## 3. Configure the R7 240 in Xorg

Create the config directory if needed:

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
```

Open:

```bash
sudo xed /etc/X11/xorg.conf.d/20-amd-secondary.conf
```

Mine contains:

```text
Section "Device"
    Identifier "AMD CRT GPU"
    Driver "modesetting"
    BusID "PCI:17:0:0"
    Option "AllowEmptyInitialConfiguration"
EndSection
```

Then reboot.

### Why

This lets Xorg initialize the R7 240 as a secondary GPU so its VGA output can be used.

Your BusID may be different.

---

## 4. Find the CRT Output

Run:

```bash
xrandr
```

Mine uses:

```text
CRT: VGA-1-1
```

Replace `VGA-1-1` in the commands below if your CRT uses a different output name.

---

## 5. Create 640x480i

Create the mode:

```bash
xrandr --newmode "640x480i-test" 12.162382 \
640 658 716 773 \
480 488 493 525 \
interlace -hsync -vsync
```

Attach it to the CRT:

```bash
xrandr --addmode VGA-1-1 "640x480i-test"
```

Enable it:

```bash
xrandr --output VGA-1-1 --mode "640x480i-test"
```

### Why

This gives the CRT a working 480i signal.

This exact mode was horizontally compressed on my R7 240 under `amdgpu`, but worked correctly after switching to `radeon`.

---

## 6. Create 320x240p

Create the mode:

```bash
xrandr --newmode "320x240p" 6.260 \
320 336 368 400 \
240 242 245 261 \
-hsync -vsync
```

Attach it:

```bash
xrandr --addmode VGA-1-1 "320x240p"
```

Enable it:

```bash
xrandr --output VGA-1-1 --mode "320x240p"
```

### Why

This gives the CRT a native 240p mode.

---

## 7. Position the CRT

The CRT needs a position inside your X11 desktop layout.

Use:

```bash
xrandr
```

to see your monitor names and current layout.

For example, if your main monitor is 3440x1440 and you want the CRT to the left and bottom-aligned:

### 480i

```bash
xrandr \
--output VGA-1-1 --mode "640x480i-test" --pos 0x960 \
--output HDMI-0 --primary --mode 3440x1440 --pos 640x0
```

### 240p

```bash
xrandr \
--output VGA-1-1 --mode "320x240p" --pos 0x1200 \
--output HDMI-0 --primary --mode 3440x1440 --pos 320x0
```

These are only example coordinates.

Change the output names, resolutions, and positions to match your own monitor setup.

### Why

XRandR treats all connected displays as one large desktop.

The `--pos` values control where each display sits inside that desktop.

---

## 8. 480i Script

Custom XRandR modes do not survive a reboot, so this script recreates the 480i mode and enables the CRT.

Edit the `MAIN`, `CRT`, and position values to match your own setup.

```bash
#!/usr/bin/env bash
set -euo pipefail

MAIN="HDMI-0"
CRT="VGA-1-1"
MODE="640x480i-test"

MODELINE=(
    12.162382
    640 658 716 773
    480 488 493 525
    interlace -hsync -vsync
)

xrandr --delmode "$CRT" "$MODE" 2>/dev/null || true
xrandr --rmmode "$MODE" 2>/dev/null || true

xrandr --newmode "$MODE" "${MODELINE[@]}"
xrandr --addmode "$CRT" "$MODE"

xrandr \
    --output "$CRT" --mode "$MODE" --pos 0x960 \
    --output "$MAIN" --primary --pos 640x0
```

---

## 9. 240p Script

Edit the `MAIN`, `CRT`, and position values to match your own setup.

```bash
#!/usr/bin/env bash
set -euo pipefail

MAIN="HDMI-0"
CRT="VGA-1-1"
MODE="320x240p"

MODELINE=(
    6.260
    320 336 368 400
    240 242 245 261
    -hsync -vsync
)

xrandr --delmode "$CRT" "$MODE" 2>/dev/null || true
xrandr --rmmode "$MODE" 2>/dev/null || true

xrandr --newmode "$MODE" "${MODELINE[@]}"
xrandr --addmode "$CRT" "$MODE"

xrandr \
    --output "$CRT" --mode "$MODE" --pos 0x1200 \
    --output "$MAIN" --primary --pos 320x0
```

---

## 10. CRT Off Script

```bash
#!/usr/bin/env bash

xrandr \
--output VGA-1-1 --off \
--output HDMI-0 --primary --pos 0x0
```

---

## Notes

You may need to change:

- GPU PCI address
- Xorg BusID
- CRT output name
- main monitor output name
- display positions

My setup uses:

```text
R7 240 PCI address: 11:00.0
Xorg BusID:         PCI:17:0:0
CRT output:         VGA-1-1
Main output:        HDMI-0
CRT:                Toshiba 14AF44
```

> [!WARNING]
> Make sure your CRT and signal chain are appropriate for 240p and 480i before using these modes.
