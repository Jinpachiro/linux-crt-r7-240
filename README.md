# linux-crt-r7-240
This guide documents how I set up a consumer CRT as a secondary display on Linux Mint/X11 using an AMD Radeon R7 240 for analog VGA output. It covers the kernel driver change required for correct 480i output, Xorg configuration, custom 240p and 480i modelines, and scripts for switching between CRT modes.

Linux Mint CRT Setup with an AMD R7 240
This is how I got a consumer CRT working as a second display on Linux Mint/X11 using an AMD Radeon R7 240 for analog VGA output.
My setup:
NVIDIA GPU → main monitor
AMD R7 240 → VGA → transcoder → component → CRT

The CRT modes I use are:
320x240p
640x480i

The main issue I had was that 240p worked correctly, but 480i was horizontally compressed. Switching the R7 240 from the amdgpu kernel driver to the older radeon driver fixed it.
1. Find the R7 240
Run:
lspci | grep -Ei 'vga|3d|display'

Mine appears as:
11:00.0

Check which kernel driver it is using:
lspci -k -s 11:00.0

Originally mine showed:
Kernel driver in use: amdgpu

2. Switch the R7 240 to the Radeon driver
Edit GRUB:
sudo xed /etc/default/grub

Change:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"

to:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash radeon.si_support=1 amdgpu.si_support=0"

Then run:
sudo update-grub
sudo reboot

After rebooting, check again:
lspci -k -s 11:00.0

You should see:
Kernel driver in use: radeon

Why
On my R7 240:
amdgpu:
240p works
480i is horizontally compressed

radeon:
240p works
480i works

3. Configure the R7 240 in Xorg
Create the config directory if needed:
sudo mkdir -p /etc/X11/xorg.conf.d

Open:
sudo xed /etc/X11/xorg.conf.d/20-amd-secondary.conf

Mine contains:
Section "Device"
    Identifier "AMD CRT GPU"
    Driver "modesetting"
    BusID "PCI:17:0:0"
    Option "AllowEmptyInitialConfiguration"
EndSection

Then reboot.
Why
This lets Xorg initialize the R7 240 as a secondary GPU so its VGA output can be used.
Your BusID may be different.
4. Find the CRT output
Run:
xrandr

Mine uses:
CRT:  VGA-1-1
Main: HDMI-0

Replace those names with your own if they are different.
5. Create 640x480i
xrandr --newmode "640x480i-test" 12.162382 \
640 658 716 773 \
480 488 493 525 \
interlace -hsync -vsync

Attach it:
xrandr --addmode VGA-1-1 "640x480i-test"

Enable it:
xrandr --output VGA-1-1 --mode "640x480i-test"

Why
This gives the CRT a 480i signal.
This exact mode was broken under amdgpu on my R7 240 and worked correctly under radeon.
6. Create 320x240p
xrandr --newmode "320x240p" 6.260 \
320 336 368 400 \
240 242 245 261 \
-hsync -vsync

Attach it:
xrandr --addmode VGA-1-1 "320x240p"

Enable it:
xrandr --output VGA-1-1 --mode "320x240p"

Why
This gives the CRT a native 240p mode.
7. Position the CRT
My main monitor is:
3440x1440 @ 239.98 Hz

For 480i:
xrandr \
--output VGA-1-1 --mode "640x480i-test" --pos 0x960 --transform none \
--output HDMI-0 --primary --mode 3440x1440 --rate 239.98 --pos 640x0

For 240p:
xrandr \
--output VGA-1-1 --mode "320x240p" --pos 0x1200 --transform none \
--output HDMI-0 --primary --mode 3440x1440 --rate 239.98 --pos 320x0

Why
These commands place the CRT to the left of my main monitor and line up the bottoms.
Your positions may be different depending on your monitor setup.
8. 480i script
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
    --output "$CRT" --mode "$MODE" --pos 0x960 --transform none \
    --output "$MAIN" --primary --mode 3440x1440 --rate 239.98 --pos 640x0

Why
Custom XRandR modes do not survive a reboot, so this recreates the mode and turns the CRT on.
9. 240p script
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
    --output "$CRT" --mode "$MODE" --pos 0x1200 --transform none \
    --output "$MAIN" --primary --mode 3440x1440 --rate 239.98 --pos 320x0

10. CRT off script
#!/usr/bin/env bash

xrandr \
--output VGA-1-1 --off \
--output HDMI-0 --primary --mode 3440x1440 --rate 239.98 --pos 0x0

Notes
You may need to change:
GPU PCI address
Xorg BusID
CRT output name
main monitor output name
main monitor resolution
main monitor refresh rate
display positions

My setup uses:
R7 240 PCI address: 11:00.0
Xorg BusID: PCI:17:0:0
CRT output: VGA-1-1
Main output: HDMI-0
Main resolution: 3440x1440
Main refresh: 239.98 Hz
CRT: Toshiba 14AF44

[!WARNING]
Do not blindly use these CRT modes on random displays. Make sure your CRT and signal chain are appropriate for 240p and 480i.

This is probably the level I’d keep it at for GitHub: enough explanation to understand why each step exists, but no side history or troubleshooting trivia.
